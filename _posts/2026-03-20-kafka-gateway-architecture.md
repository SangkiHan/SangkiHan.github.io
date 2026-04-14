---
layout: post
title: "IoT Gateway 아키텍처 개선 - Function App에서 Kafka 기반으로"
date: 2026-03-20 14:00:00 +0900
categories: [Event, Kafka]
tags: [Architecture, Kafka, IoT, Spring Boot]
---

IoT 기기에서 수집된 데이터를 저장하는 파이프라인을 운영하던 중, 기존 아키텍처의 한계를 느끼고 Kafka 기반으로 전환하게 되었다.

## 기존 아키텍처

```
IoT 기기 → MQTT → Gateway 서버 → Azure Function App → Table Storage
                                                      ↓
                                                   배치 작업
                                                      ↓
                                                   MongoDB
```

IoT 기기에서 5분마다 17,000개의 데이터를 Gateway 서버로 전송하고, Gateway가 Azure Function App을 호출해 Table Storage에 저장한 뒤, 배치 작업으로 MongoDB에 동기화하는 구조였다.

### 문제점

- **Function App 비용**: 대량 호출 시 Azure Function App의 비용이 급격히 증가
- **배치 지연**: Table Storage → MongoDB 동기화는 배치로 처리되어 실시간성이 없음
- **단일 흐름**: 데이터를 추가 소비자(Consumer)가 필요할 때마다 Gateway 코드를 수정해야 함
- **재처리 불가**: Function App 실패 시 재처리 메커니즘이 없어 데이터 유실 가능성 존재

---

## 개선된 아키텍처

```
IoT 기기 → MQTT → Gateway 서버 → Kafka Broker → Consumer (MongoDB)
                                       → Consumer (Table Storage)  ← 추후 확장
```

Gateway 서버는 Kafka에 메시지를 발행하고, 각 Consumer가 독립적으로 데이터를 처리하도록 변경했다.

### 개선 효과

| 항목 | 기존 | 개선 |
|---|---|---|
| MongoDB 저장 방식 | 배치 (지연 발생) | 실시간 |
| 장애 내성 | Function App 실패 시 유실 | Kafka offset으로 재처리 보장 |
| 확장성 | Gateway 코드 수정 필요 | Consumer만 추가하면 됨 |
| 비용 | Azure Function App 호출 비용 | Kafka 브로커 유지 비용 (고정) |
| 중복 처리 방지 | 별도 처리 없음 | `rowKey`를 `@Id`로 사용한 idempotent upsert |

---

## 인프라 구성 (docker-compose)

처음에는 ZooKeeper 방식으로 구성했다가 KRaft 방식으로 전환했다.

### ZooKeeper vs KRaft

| 항목 | ZooKeeper 방식 | KRaft 방식 |
|---|---|---|
| 별도 프로세스 | ZooKeeper 필요 | 없음 |
| Controller 관리 | ZooKeeper가 담당 | Kafka가 직접 담당 |
| 브로커 3대 운영 시 | ZooKeeper 3대 + Kafka 3대 = 6대 | Kafka 3대만 |
| 관리 복잡도 | 높음 | 낮음 |

ZooKeeper는 자체도 과반수 규칙이 적용되는 클러스터라 브로커 3대에 ZooKeeper 3대가 추가로 필요했다. KRaft는 Kafka 안에 Controller 역할이 통합되어 브로커 3대만으로 동일한 고가용성을 구성할 수 있다.

### KRaft 기반 docker-compose

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      CLUSTER_ID: 'q1Sh3IspTsyGH5YQFZK1pw'  # 클러스터 고유 ID, 같은 클러스터 브로커는 동일해야 함
      KAFKA_NODE_ID: 1                          # 브로커 고유 ID
      KAFKA_PROCESS_ROLES: broker,controller    # ZooKeeper 없이 Kafka가 직접 Controller 역할
      KAFKA_LISTENERS: PLAINTEXT_INTERNAL://0.0.0.0:29092,PLAINTEXT_EXTERNAL://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT_INTERNAL://kafka:29092,PLAINTEXT_EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT_INTERNAL:PLAINTEXT,PLAINTEXT_EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT_INTERNAL  # 브로커 간 복제에 사용할 리스너
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER           # Controller 통신 전용 리스너
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093          # Controller 투표 멤버 목록
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka_data:/var/lib/kafka/data

  kafka-init:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      - kafka
    entrypoint: [ "/bin/sh", "-c" ]
    command: >
      "sleep 10 &&
      kafka-topics --create --if-not-exists
      --topic gateway-data
      --bootstrap-server kafka:29092
      --partitions 5
      --replication-factor 1"

  mongodb:
    image: mongo:7.0
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin1234
    volumes:
      - mongodb_data:/data/db

volumes:
  kafka_data:
  mongodb_data:
```

### 리스너 3개를 분리한 이유

```
PLAINTEXT_INTERNAL://kafka:29092  → Docker 컨테이너 간 통신 (kafka-init → kafka)
PLAINTEXT_EXTERNAL://localhost:9092 → 호스트(Spring Boot) → Kafka
CONTROLLER://kafka:9093           → 브로커 간 Controller 선출/통신 전용 (KRaft 추가)
```

Docker 내부에서 `localhost:9092`를 쓰면 컨테이너가 자기 자신을 가리켜 연결이 실패한다. INTERNAL/EXTERNAL을 분리해 이 문제를 해결하고, KRaft Controller 통신은 별도 포트(9093)로 격리했다.

### kafka-init 설정 설명

```yaml
sleep 10           # Kafka 내부 준비 대기 (depends_on은 시작만 보장, 준비 완료는 보장 안 함)
--if-not-exists    # 이미 토픽 있으면 에러 없이 스킵
--bootstrap-server kafka:29092   # Docker 내부 리스너 주소 사용
--partitions 5     # 파티션 5개 (성능 측정 후 결정, 아래 참고)
--replication-factor 1  # 브로커 1대라 1로 설정, 운영 3대면 3으로 변경
```

---

## Producer (Gateway 서버)

### application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      batch-size: 524288   # 512KB
      linger-ms: 20
      compression-type: lz4

gateway:
  kafka:
    topic: gateway-data
  publisher:
    site-count: 17000
```

`linger-ms`와 `batch-size`를 키워 producer가 메시지를 묶어서 전송하도록 튜닝했다. lz4 압축으로 네트워크 대역폭도 절감한다.

### Publisher 코드

```java
@Component
public class GatewayDataPublisher {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final String topic;

    public void publish(String key, String message) {
        kafkaTemplate.send(topic, key, message)
                .exceptionally(ex -> {
                    log.error("Kafka publish failed. key={}, error={}", key, ex.getMessage());
                    return null;
                });
    }
}
```

`kafkaTemplate.send()`는 비동기로 동작한다. 17,000개를 루프로 보내도 블로킹 없이 빠르게 처리된다.

### Scheduler 코드

```java
@Scheduled(fixedRate = 5 * 60 * 1000, initialDelay = 0)
public void publish() {
    LocalDateTime now = LocalDateTime.now();
    String batchId = UUID.randomUUID().toString();
    long start = System.currentTimeMillis();

    for (int i = 1; i <= siteCount; i++) {
        String siteId = String.format("%05d", i);
        String message = generator.generate(siteId, now, batchId, siteCount);
        publisher.publish(siteId, message);
    }

    long elapsed = System.currentTimeMillis() - start;
    log.info("Published {} messages in {}ms batchId={}", siteCount, elapsed, batchId);
}
```

5분마다 17,000개 메시지를 발행한다. `batchId`와 `batchSize`를 각 메시지에 포함시켜 Consumer 측에서 배치 완료 시점과 처리 시간을 측정할 수 있도록 했다.

---

## Consumer (MongoDB)

### application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: mongo-consumer
      auto-offset-reset: earliest
    listener:
      concurrency: 5
  data:
    mongodb:
      uri: mongodb://admin:admin1234@localhost:27017/gateway?authSource=admin

gateway:
  kafka:
    topic: gateway-data
```

### Consumer 코드

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class GatewayDataConsumer {

    private final GatewayDataRepository repository;
    private final ObjectMapper objectMapper;

    private final ConcurrentHashMap<String, long[]> batchTracker = new ConcurrentHashMap<>();

    @KafkaListener(topics = "${gateway.kafka.topic}", groupId = "${spring.kafka.consumer.group-id}")
    public void consume(String message) {
        try {
            GatewayDataDocument document = objectMapper.readValue(message, GatewayDataDocument.class);
            repository.save(document);

            String batchId = document.getBatchId();
            int batchSize = document.getBatchSize();

            long[] state = batchTracker.computeIfAbsent(batchId, k -> new long[]{0L, System.currentTimeMillis()});

            long count;
            synchronized (state) {
                state[0]++;
                count = state[0];
            }

            if (count == batchSize) {
                long elapsed = System.currentTimeMillis() - state[1];
                log.info("Batch complete. batchId={} count={} elapsed={}ms", batchId, batchSize, elapsed);
                batchTracker.remove(batchId);
            }
        } catch (Exception e) {
            log.error("Failed to consume message. error={}", e.getMessage(), e);
        }
    }
}
```

### Document 모델

```java
@Getter
@NoArgsConstructor
@Document(collection = "gateway_data")
public class GatewayDataDocument {

    @Field("partition_key")
    @JsonProperty("PartitionKey")
    private String partitionKey;

    @Id
    @JsonProperty("RowKey")
    private String rowKey;          // MongoDB _id로 사용 → 중복 insert 방지

    @Field("delivery_info")
    @JsonProperty("delivery_info")
    private String deliveryInfo;

    @Field("subject_code")
    @JsonProperty("subject_code")
    private String subjectCode;

    @Field("payload")
    private Map<String, Object> payload;

    @Field("message_datetime")
    @JsonProperty("message_datetime")
    private String messageDatetime;

    // ...
}
```

`rowKey`를 MongoDB의 `@Id`로 사용하는 것이 핵심이다. Kafka는 **at-least-once** 전달을 보장하므로 Consumer 장애 후 재시작 시 메시지가 중복 처리될 수 있다. `rowKey`를 `_id`로 쓰면 동일한 문서를 두 번 저장해도 upsert가 되어 중복 데이터가 생기지 않는다.

---

## 파티션별 처리 성능 비교

17,000개 메시지 기준으로 파티션 수와 consumer concurrency를 맞춰가며 end-to-end 처리 시간을 측정했다.

| 파티션 수 | elapsed | 이전 대비 개선율 |
|:---:|---:|---:|
| 1 | 18,587ms | - |
| 3 | 7,548ms | -59% |
| 4 | 6,223ms | -17% |
| **5** | **5,590ms** | **-10%** |
| 6 | 5,125ms | -8% |

### 파티션 5개로 결정한 이유

5→6 구간에서 개선율이 8%로 떨어지며 수렴하는 것이 보인다. MongoDB가 single node이기 때문에 파티션을 아무리 늘려도 MongoDB write 처리량의 한계에 수렴한다.

파티션이 많아질수록 Kafka 브로커의 메모리와 파일 핸들 사용량도 증가한다. 성능 개선 폭이 줄어드는 시점에서 멈추는 것이 리소스 대비 효율이 좋다.

**결론: 파티션 5개, concurrency 5개가 현재 인프라에서의 최적점이다.**

실제 운영 VM(4Core 8GB) 기준에서는 여기서 시작해 모니터링하며 조정하는 방식을 권장한다. MongoDB를 replica set이나 별도 고사양 서버로 분리하면 파티션을 더 늘릴 여지가 생긴다.

---

## 정리

Kafka를 도입하면서 얻은 가장 큰 이점은 **데이터 파이프라인의 분리**다. Gateway는 Kafka에 던지기만 하고, 어떤 Consumer가 붙든 신경 쓰지 않아도 된다. MongoDB Consumer 외에 Table Storage Consumer, 알람 Consumer 등을 나중에 추가할 때 Gateway 코드는 건드릴 필요가 없다.

또한 Kafka의 offset 관리 덕분에 Consumer가 죽었다가 살아나도 마지막으로 처리한 위치부터 다시 읽을 수 있어 데이터 유실 걱정이 없다.
