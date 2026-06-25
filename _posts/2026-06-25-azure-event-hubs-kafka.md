---
layout: post
title: "Azure Event Hubs를 Kafka Broker로 사용하기"
date: 2026-06-25 17:00:00 +0900
categories: [Event, Kafka]
tags: [Azure, EventHubs, Kafka, Spring Boot, IoT]
---

이전 프로젝트에서는 Kafka KRaft 클러스터를 직접 VM에 구축해서 운영했다. 이번 신규 프로젝트를 시작하면서 Kafka Broker를 어떻게 구성할지 고민하다가, 처음부터 Azure Event Hubs의 Kafka 호환 엔드포인트를 사용하기로 결정했다. 선택 이유와 설정 과정, 파티션 설계 결정에 대해 정리한다.

## 왜 Azure Event Hubs를 선택했나

### 자체 Kafka 클러스터 운영의 경험

이전 프로젝트에서 Kafka KRaft 클러스터를 직접 운영하면서 서비스 자체는 잘 동작했지만 운영 측면에서 신경 써야 할 부분이 많았다.

- **인프라 관리**: 브로커 장애 시 직접 대응해야 함
- **확장 작업**: 처리량이 늘어날 때 브로커를 추가하고 파티션을 재배분하는 작업 필요
- **모니터링**: Kafka 자체 메트릭 수집 및 알림 구성 직접 세팅

신규 프로젝트에서는 인프라 운영 부담을 줄이고 서비스 개발에 집중하고 싶었다.

### Azure Event Hubs 선택 이유

| 항목 | 자체 Kafka 클러스터 | Azure Event Hubs |
|------|-------------------|-----------------|
| 브로커 관리 | 직접 | Azure가 관리 |
| 확장 | 브로커 추가 + 파티션 재배분 | TU 조정 또는 Auto-Inflate |
| ZooKeeper/Controller | 직접 구성 | 불필요 |
| Kafka 프로토콜 호환 | - | 포트 9093으로 그대로 사용 |
| 비용 | VM 고정 비용 | 사용량 기반 |

**가장 큰 이유는 Kafka 프로토콜 호환성**이다. 이전 프로젝트에서 사용하던 Spring Kafka 코드 구조를 그대로 가져오면서 `bootstrap-servers`와 인증 설정만 변경하면 연결할 수 있었다.

---

## Azure Event Hubs 핵심 개념

Kafka에 익숙하다면 다음 매핑으로 이해하면 쉽다.

| Kafka | Azure Event Hubs |
|-------|-----------------|
| Broker Cluster | Namespace |
| Topic | Event Hub |
| Consumer Group | Consumer Group (수동 생성 필요) |
| Partition | Partition |

### 중요한 차이점

**Consumer Group 자동 생성 안 됨**  
일반 Kafka는 Consumer가 처음 연결하면 Consumer Group이 자동으로 생성된다. Azure Event Hubs는 `$Default` 그룹만 기본 제공되고, 나머지는 Portal이나 CLI로 직접 생성해야 한다.

**파티션 수 변경 불가**  
생성 후 파티션 수를 늘릴 수 없다. 처음 설계 시 신중하게 결정해야 한다.

**LZ4 압축 미지원**  
Standard 티어는 LZ4 압축을 지원하지 않는다. `compression-type: lz4`로 설정하면 `UnsupportedForMessageFormatException`이 발생한다. `none` 또는 `gzip`을 사용해야 한다.

---

## 장단점

### 장점

- **운영 부담 제로**: Controller, ZooKeeper, 브로커 장애 대응을 Azure가 담당
- **코드 변경 최소화**: Kafka 프로토콜 그대로 사용, 연결 설정만 변경
- **Auto-Inflate**: 처리량 초과 시 Throughput Unit 자동 확장
- **Azure 생태계 통합**: Portal에서 메트릭, Consumer Group 오프셋 등 바로 확인

### 단점

- **파티션 수 고정**: 생성 후 변경 불가 → 초기 설계가 중요
- **LZ4 미지원**: Standard 티어 기준, 압축 옵션 제한
- **Log Compaction 미지원**: `cleanup.policy=compact` 사용 불가, `delete`만 지원
- **최대 보존 기간 7일**: Standard 티어 기준
- **Consumer Group 수동 생성**: 자동 생성을 기대하면 연결이 안 됨

---

## 파티션을 32개로 설정한 이유

### 처리량 관점

현재 IoT 기기 17,000개가 5분마다 데이터를 전송한다. Consumer는 Spring Kafka로 구성하며 `listener.concurrency: 5`로 5개 스레드가 동시에 처리한다.

파티션이 5개였다면 지금 당장은 1:1 대응이 되지만, Consumer를 더 늘리거나 인스턴스를 2개 이상 띄울 때 확장이 불가능하다.

**파티션은 병렬 처리의 최대 단위**다. Consumer 수 ≤ 파티션 수여야 병렬성을 활용할 수 있다.

### 왜 하필 32개인가

- **Azure Event Hubs Standard 티어 최대값이 32**
- 5개 Consumer가 각 6~7개 파티션을 담당 → 균등 분배
- 향후 Consumer를 최대 32개까지 스케일아웃 가능
- Premium 티어로 업그레이드하면 파티션을 더 늘릴 수 있지만, 현재 규모에서 32개로 충분

```
파티션 32개 / Consumer 5개 = 파티션당 6.4개 → 5개 스레드가 6~7개씩 담당
파티션 32개 / Consumer 10개 = 파티션당 3.2개 → 인스턴스 2개 운영 시에도 균등 분배
```

파티션 5개로 시작하면 지금 당장 속도 차이는 없다. 처리 속도는 Consumer 수에 달려있기 때문이다. 하지만 **나중에 파티션을 늘릴 수 없으므로**, 처음부터 확장 여지를 두고 32개로 설정했다.

---

## 연결 설정

### SASL_SSL 인증

Azure Event Hubs의 Kafka 엔드포인트는 포트 9093을 사용하며 SASL_SSL 인증이 필수다.

```yaml
spring:
  kafka:
    bootstrap-servers: <네임스페이스>.servicebus.windows.net:9093
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: PLAIN
      sasl.jaas.config: >
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="$ConnectionString"
        password="<Connection String>";
```

`username`은 항상 `$ConnectionString` 고정 문자열이고, `password`에 Azure Portal에서 복사한 Connection String 전체를 넣는다.

### Consumer 설정

```yaml
spring:
  kafka:
    consumer:
      group-id: mongo-consumer
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
    listener:
      concurrency: 5
```

`CooperativeStickyAssignor`를 사용하면 리밸런싱 시 기존에 담당하던 파티션을 유지하면서 증분 방식으로 재배분한다. Consumer 추가/제거 시 전체 파티션이 재배분되는 Stop-The-World 리밸런싱을 피할 수 있다.

### Producer 설정 주의사항

```yaml
spring:
  kafka:
    producer:
      compression-type: none  # lz4 사용 시 UnsupportedForMessageFormatException 발생
```

`compression-type: lz4`로 설정했다가 아래 오류를 만났다.

```
org.apache.kafka.common.errors.UnsupportedForMessageFormatException:
The message format version on the broker does not support the request.
```

Azure Event Hubs Standard 티어가 LZ4를 지원하지 않아 발생하는 오류다. `none` 또는 `gzip`으로 변경하면 해결된다.

---

## Azure에서 사전 준비 작업

### 1. Event Hub(토픽) 생성

Azure Portal → Event Hubs 네임스페이스 → Event Hubs → `+ Event Hub`

- **Name**: `kolon`
- **Partition count**: `32` (생성 후 변경 불가)
- **Retention time**: 최대 168시간(7일)
- **Cleanup policy**: `Delete` (Compact 미지원)

DLQ용 Event Hub도 별도 생성한다. DLQ는 원본 토픽과 동일한 파티션 번호로 메시지를 라우팅하므로 **파티션 수를 반드시 동일하게** 맞춰야 한다.

- **Name**: `kolon-dlq`
- **Partition count**: `32`

### 2. Consumer Group 생성

Event Hub 상세 → Consumer groups → `+ Consumer group`

- `kolon` Event Hub에 `mongo-consumer` 생성
- `kolon-dlq` Event Hub에 `mongo-consumer-dlq` 생성

### 3. Connection String 확인

네임스페이스 → Shared access policies → `RootManageSharedAccessKey` → Connection string 복사

---

## 결과

Consumer 로그에서 32개 파티션이 5개 스레드에 균등하게 배분된 것을 확인했다.

```
mongo-consumer: partitions assigned: [kolon-0, kolon-5, kolon-10, kolon-15, kolon-22, kolon-27]
mongo-consumer: partitions assigned: [kolon-1, kolon-6, kolon-11, kolon-16, kolon-23, kolon-28]
mongo-consumer: partitions assigned: [kolon-2, kolon-7, kolon-12, kolon-17, kolon-19, kolon-24, kolon-30]
mongo-consumer: partitions assigned: [kolon-3, kolon-8, kolon-13, kolon-18, kolon-20, kolon-25, kolon-31]
mongo-consumer: partitions assigned: [kolon-4, kolon-9, kolon-14, kolon-21, kolon-26, kolon-29]
```

17,000건 기준 전체 처리 시간은 약 13초로, Kafka → MongoDB 단건 insert 파이프라인 치고 정상적인 수치다.

자체 Kafka 클러스터를 직접 운영하는 부담 없이 Kafka 프로토콜 그대로 사용할 수 있다는 점에서, 규모가 크지 않은 프로젝트에서는 Azure Event Hubs가 좋은 선택지가 될 수 있다.
