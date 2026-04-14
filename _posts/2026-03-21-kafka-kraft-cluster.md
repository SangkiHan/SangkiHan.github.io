---
layout: post
title: "Kafka KRaft 클러스터 구성 - VM 3대로 고가용성 확보하기"
date: 2026-03-21 16:00:00 +0900
categories: [Event, Kafka]
tags: [Architecture, Kafka, KRaft, Cluster, HA]
---

이전 포스팅에서 Kafka 단일 브로커로 IoT 데이터 파이프라인을 구성했다. 단일 브로커는 브로커가 죽는 순간 전체 파이프라인이 멈춘다. 운영 환경에서는 이를 허용할 수 없어 VM 3대로 Kafka 클러스터를 구성했다.

---

## 왜 3대인가

Kafka 클러스터는 **과반수(Quorum)** 원칙으로 동작한다.

| 브로커 수 | 과반수 | 허용 장애 수 |
|:---:|:---:|:---:|
| 1대 | 1대 | 0대 |
| 2대 | 2대 | **0대** |
| 3대 | 2대 | **1대** |
| 5대 | 3대 | **2대** |

2대로 구성하면 1대가 죽었을 때 남은 1대가 과반수를 충족하지 못해 클러스터가 멈춘다. 1대와 장애 내성이 동일하다. **최소 3대가 있어야 1대 장애를 허용할 수 있다.**

---

## ZooKeeper 방식을 쓰지 않은 이유

기존 Kafka는 클러스터 관리를 ZooKeeper라는 별도 프로세스에 의존했다.

```
ZooKeeper 방식:
ZooKeeper 3대 + Kafka 3대 = 총 6개 프로세스 관리 필요

KRaft 방식:
Kafka 3대만 관리
```

ZooKeeper 자체도 클러스터라 3대가 필요하고, 별도로 운영/모니터링해야 한다. Kafka 3.x부터 도입된 **KRaft**는 ZooKeeper 없이 Kafka 자체적으로 Controller 선출과 클러스터 관리를 수행한다. 관리 포인트가 절반으로 줄어든다.

---

## KRaft Controller 선출 원리

KRaft는 **Raft 합의 알고리즘**을 기반으로 한다.

### 평상시

```
브로커1 (Active Controller) ← 파티션 Leader 관리, 클러스터 메타데이터 담당
브로커2 (Follower)          ← 데이터 복제 + Controller 대기
브로커3 (Follower)          ← 데이터 복제 + Controller 대기
```

3대 모두 데이터를 처리하면서 동시에 서로 **heartbeat**를 주고받는다.

### 브로커1 장애 시

```
1. 브로커2, 브로커3이 브로커1의 heartbeat 끊김 감지
2. election timeout(랜덤 150~300ms) 발동
3. 먼저 timeout된 브로커(예: 브로커2)가 후보 선언
   → Term 번호를 1 올리고 자신에게 투표
   → 브로커3에게 투표 요청
4. 브로커3이 수락 → 브로커2가 과반수(2/3) 확보
5. 브로커2가 새 Active Controller로 선출
6. 브로커1이 담당하던 파티션 Leader를 브로커3으로 재배정
```

election timeout을 랜덤으로 설정하는 이유는 두 브로커가 동시에 후보가 되는 충돌을 방지하기 위해서다.

### 브로커1 복구 시

```
브로커1 재시작 → 클러스터 재접속
브로커2: "현재 Term=2, 내가 Controller"
브로커1: Term=1 < Term=2 → 자동으로 Follower 합류
```

복구된 브로커는 Term이 낮아서 자동으로 Follower가 된다. **한번 선출된 Controller는 장애가 나기 전까지 유지**된다. 불필요한 재선출이 없어 클러스터가 안정적이다.

---

## docker-compose 구성

각 VM에 docker-compose 파일을 하나씩 올린다. `<VM1_IP>`, `<VM2_IP>`, `<VM3_IP>`는 실제 VM IP로 교체한다.

### VM1 — docker-compose1.yml

```yaml
services:
  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka-1
    ports:
      - "9092:9092"   # Producer/Consumer 통신
      - "9093:9093"   # KRaft Controller 통신
    environment:
      CLUSTER_ID: 'q1Sh3IspTsyGH5YQFZK1pw'            # 3대 모두 동일
      KAFKA_NODE_ID: 1                                   # 브로커 고유 ID
      KAFKA_PROCESS_ROLES: broker,controller             # Kafka가 직접 Controller 역할
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://<VM1_IP>:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@<VM1_IP>:9093,2@<VM2_IP>:9093,3@<VM3_IP>:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3         # 브로커 3대 → 3으로 설정
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka1_data:/var/lib/kafka/data

  # 토픽 생성은 VM1에서만 실행 (클러스터 전체에 적용됨)
  kafka-init:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      - kafka-1
    entrypoint: [ "/bin/sh", "-c" ]
    command: >
      "sleep 15 &&
      kafka-topics --create --if-not-exists
      --topic gateway-data
      --bootstrap-server <VM1_IP>:9092
      --partitions 5
      --replication-factor 3"

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
  kafka1_data:
  mongodb_data:
```

### VM2 — docker-compose2.yml

```yaml
services:
  kafka-2:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka-2
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      CLUSTER_ID: 'q1Sh3IspTsyGH5YQFZK1pw'
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://<VM2_IP>:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@<VM1_IP>:9093,2@<VM2_IP>:9093,3@<VM3_IP>:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka2_data:/var/lib/kafka/data

volumes:
  kafka2_data:
```

### VM3 — docker-compose3.yml

```yaml
services:
  kafka-3:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka-3
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      CLUSTER_ID: 'q1Sh3IspTsyGH5YQFZK1pw'
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://<VM3_IP>:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@<VM1_IP>:9093,2@<VM2_IP>:9093,3@<VM3_IP>:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka3_data:/var/lib/kafka/data

volumes:
  kafka3_data:
```

---

## 주요 설정 설명

| 설정 | 값 | 설명 |
|---|---|---|
| `CLUSTER_ID` | 동일한 값 | 같은 클러스터 브로커임을 식별. 3대 모두 동일해야 함 |
| `KAFKA_NODE_ID` | 1 / 2 / 3 | 브로커 고유 ID. 각 VM마다 다르게 설정 |
| `KAFKA_PROCESS_ROLES` | broker,controller | ZooKeeper 없이 Kafka가 직접 Controller 역할 수행 |
| `KAFKA_CONTROLLER_QUORUM_VOTERS` | 1@vm1:9093,... | heartbeat를 주고받을 Controller 멤버 목록. 순서는 우선순위 아님 |
| `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR` | 3 | 브로커 수만큼 설정. 1대로 줄이면 에러 |
| `replication-factor` | 3 | 파티션 데이터를 3대에 복제. 1대 장애 시 데이터 유실 없음 |

---

## Spring Boot 연결 설정

```yaml
spring:
  kafka:
    bootstrap-servers: <VM1_IP>:9092,<VM2_IP>:9092,<VM3_IP>:9092
```

3개를 모두 적어두는 이유가 있다. `bootstrap-servers`는 최초 접속용 주소일 뿐이고, 이후 Kafka가 클러스터 전체 메타데이터를 응답해줘서 실제 통신은 각 파티션 Leader에게 직접 한다.

```
Producer → VM1:9092 최초 접속
         ← 메타데이터 수신: 파티션0=VM2, 파티션1=VM3, 파티션2=VM1...
         → 이후 각 파티션 Leader에게 직접 전송
```

VM1이 죽은 상태에서 시작하더라도 VM2나 VM3으로 접속해 정상 동작한다.

---

## 실행 순서

```bash
# VM1
docker-compose -f docker-compose1.yml up -d

# VM2
docker-compose -f docker-compose2.yml up -d

# VM3
docker-compose -f docker-compose3.yml up -d
```

3대가 모두 뜬 후 VM1의 kafka-init이 토픽을 생성한다. 토픽은 클러스터 전체에 공유되므로 1번만 실행하면 된다.

---

## 단일 브로커 vs 3대 클러스터 비교

| 항목 | 단일 브로커 | 3대 클러스터 |
|---|---|---|
| 브로커 장애 시 | 전체 중단 | 1대 장애 허용, 무중단 |
| 데이터 복제 | 없음 | 3대에 복제 |
| Controller 선출 | 해당 없음 | Raft 알고리즘으로 자동 선출 |
| 처리량 | 단일 브로커 처리량 | 파티션이 3대에 분산되어 부하 분산 |
| 관리 복잡도 | 낮음 | 중간 (KRaft로 ZooKeeper 제거) |

---

## Spring Boot Producer/Consumer 설정 상세

### Serializer/Deserializer

Kafka는 메시지를 **바이트(byte[])** 로 전송한다. Java 타입을 바이트로 변환하는 설정이 필요하다.

```yaml
spring:
  kafka:
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

```
Producer: Java String → StringSerializer → byte[] → Kafka 전송
Consumer: Kafka 수신 → byte[] → StringDeserializer → Java String
```

우리 프로젝트는 메시지를 String(JSON 문자열)으로 직렬화해서 보내고, Consumer에서 `objectMapper.readValue()`로 역직렬화한다.

---

### Producer 배치 전송 설정

```yaml
spring:
  kafka:
    producer:
      batch-size: 524288   # 512KB
      linger-ms: 20
      compression-type: lz4
```

배치 없이 보내면 메시지마다 네트워크 왕복이 발생한다.

```
배치 없이 17,000개:  네트워크 왕복 17,000번
배치로 17,000개:     네트워크 왕복 수십번
```

**batch-size: 512KB** — 메시지가 쌓여서 512KB가 되면 즉시 전송

**linger-ms: 20** — 512KB가 안 차도 20ms 기다렸다가 전송. 둘 중 먼저 조건이 충족되면 전송.

```
512KB 찼을 때  → 즉시 전송
20ms 지났을 때 → 512KB 안 찼어도 전송
```

**compression-type: lz4** — 배치를 압축해서 전송

| 압축 타입 | 압축률 | 속도 |
|---|---|---|
| none | 없음 | 가장 빠름 |
| lz4 | 보통 | 빠름 |
| snappy | 보통 | 보통 |
| gzip | 높음 | 느림 |

lz4는 압축률과 속도의 균형이 좋아서 Kafka에서 가장 많이 쓴다.

---

### Consumer 파티션 배정 알고리즘

Consumer가 시작하면 Group Coordinator가 파티션을 배정한다. 기본값은 **RangeAssignor**다.

**RangeAssignor (기본값)** — 파티션을 범위로 잘라서 배정

```
파티션 5개, Consumer 2대

파티션0, 1, 2 → Consumer1
파티션3, 4    → Consumer2
```

**RoundRobinAssignor** — 파티션을 하나씩 번갈아가며 배정

```
파티션 5개, Consumer 2대

파티션0, 2, 4 → Consumer1
파티션1, 3    → Consumer2
```

RoundRobin으로 변경 시:

```yaml
spring:
  kafka:
    consumer:
      partition-assignment-strategy: 
        org.apache.kafka.clients.consumer.RoundRobinAssignor
```

---

## 정리

VM 3대 KRaft 클러스터 구성으로 얻은 것:

- **1대 장애 허용**: 브로커 1대가 죽어도 나머지 2대가 과반수를 유지해 클러스터가 계속 동작한다
- **자동 복구**: Raft 알고리즘으로 수십초 내에 새 Controller가 선출되고 파티션 Leader가 재배정된다
- **ZooKeeper 제거**: KRaft 도입으로 관리할 프로세스가 절반으로 줄었다
- **데이터 무손실**: replication-factor 3으로 브로커 1대가 죽어도 데이터는 나머지 2대에 보존된다
