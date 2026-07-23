---
layout: post
title: "MongoDB Time Series Collection 도입기: 일별 컬렉션에서 벗어나기"
date: 2026-07-16 18:00:00 +0900
categories: [Database, MongoDB]
tags: [MongoDB, TimeSeries, IoT, Kafka, Spring Boot, WiredTiger, Troubleshooting]
---

[이전 글](/posts/azure-event-hubs-kafka/)에서 gateway → Azure Event Hubs(Kafka) → gateway-consumer-mongo 파이프라인을 구성한 이야기를 정리했다. 이번 글은 그 뒤에 이어지는 이야기로, MongoDB에 데이터를 저장하는 방식을 **일별(YYYYMMDD) 컬렉션 구조에서 Time Series Collection 하나로 전환**하면서 겪은 시행착오를 정리한다.

## 배경: 일별 컬렉션 구조의 문제

기존 운영 DB는 `20260716`처럼 날짜별로 컬렉션을 새로 만들어 데이터를 저장하는 구조였다. 실제 운영 DB의 특정 하루 컬렉션은 4,841,768건에 3.17GB를 차지하고 있었다.

이 구조에서 가장 불편했던 건 **여러 날짜에 걸친 조회**였다. 예를 들어 특정 site의 최근 며칠치 온도 데이터를 조회하는 코드는 이런 식이었다.

```java
for (String collection : afterDates) {
    LocalDate collectionDate = LocalDate.parse(collection, FORMATTER);
    LocalDateTime startDateTime = collectionDate.atStartOfDay();
    LocalDateTime endDateTime = DateUtil.endOfDay(collectionDate);

    String startStr = DateUtil.convertLocalDateTimeToISOString(startDateTime);
    String endStr = DateUtil.convertLocalDateTimeToISOString(endDateTime);

    List<IotStateReportDocument> docs = iotStateReportRepository
            .findBySiteIdWithDateRange(siteId, collection, startStr, endStr);

    for (IotStateReportDocument iotDoc : docs) {
        String[] kstDateTime = DateUtil.convertUTCToKSTDateTime(iotDoc.getPayload().getReportDt());
        resultList.add(toTempData(siteId, kstDateTime, iotDoc, facility));
    }
}
```

조회 대상 날짜 수만큼 컬렉션을 순회하면서 매번 별도 쿼리를 날리고, 그 결과를 애플리케이션 레벨에서 다시 합쳐야 했다. 날짜 범위가 늘어날수록 쿼리 횟수도 그만큼 늘어나는 구조였다.

## 왜 MongoDB Time Series Collection인가

시계열 데이터를 저장하는 선택지는 크게 두 가지였다.

1. InfluxDB, TimescaleDB 같은 전용 시계열 DB 도입
2. 이미 운영 중인 MongoDB에 Time Series Collection 기능 적용

전용 시계열 DB가 압축·조회 성능 면에서는 더 유리할 수 있지만, 이미 MongoDB 인프라와 운영 경험이 갖춰진 상태에서 새로운 DB를 하나 더 들이는 건 전환 비용이 크다고 판단했다. MongoDB 5.0부터 지원하는 Time Series Collection이 우리 데이터 패턴(디바이스별로 주기적으로 들어오는 측정치)에 정확히 맞는 기능이라, 이걸로 먼저 검증해보기로 했다.

## 컬렉션 설계

현재 규모는 site 17,000개가 5분마다 데이터를 보고하므로, 하루 기준 `17,000 × 288 = 4,896,000`건이 발생한다.

일별 컬렉션을 쪼개는 대신 **컬렉션 하나 + Time Series 내부 버킷팅**으로 가기로 했다. 컬렉션을 여러 개로 쪼개는 건 조회 시 여러 컬렉션을 순회해야 하는 지금의 문제를 반복하는 것이고, 시계열 컬렉션은 `metaField` + 시간 기준으로 내부적으로 이미 버킷 단위 파티셔닝을 해주기 때문이다.

```js
db.createCollection("iot-state-report", {
  timeseries: {
    timeField: "report_datetime",
    metaField: "site_id",
    granularity: "minutes"
  }
})
```

- `timeField`: `report_datetime` — 리포트 발생 시각
- `metaField`: `site_id` — 조회 시 가장 자주 필터링하는 축
- `granularity`: `minutes` — 5분 간격 리포트에 맞춤 (내부적으로 `bucketMaxSpanSeconds: 86400`, 24시간 단위로 버킷 구성)
- 보관 주기 13개월(`expireAfterSeconds: 34190000`)도 함께 설정

## 테스트 환경 구성: gateway 백필 기능

실시간으로 5분마다 쌓이는 걸 기다리면 하루치를 검증하는 데 하루가 걸린다. 그래서 gateway에 **하루치 데이터를 한 번에 발행하는 백필 기능**을 추가했다.

```java
@Component
@ConditionalOnProperty(prefix = "kolon.backfill", name = "enabled", havingValue = "true")
public class BackfillRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        LocalDateTime dayStart = LocalDate.now().atStartOfDay();
        int slotCount = durationMinutes / intervalMinutes;  // 1440 / 5 = 288

        for (int slot = 0; slot < slotCount; slot++) {
            LocalDateTime timestamp = dayStart.plusMinutes((long) slot * intervalMinutes);
            String batchId = UUID.randomUUID().toString();

            IntStream.rangeClosed(1, siteCount).parallel().forEach(i -> {
                String siteId = String.format("%05d", i);
                String message = generator.generate(siteId, timestamp, batchId, siteCount);
                publisher.publish(siteId, message);
            });
        }
        publisher.flush();
    }
}
```

슬롯(5분 단위) 안에서만 site 17,000개를 병렬 처리하고, 슬롯 간에는 순차 진행하도록 구성했다. 같은 site 기준으로는 항상 시간 순서대로 발행되도록 하기 위해서다. `@ConditionalOnProperty`로 백필 모드일 때는 기존 5분 주기 실시간 스케줄러가 자동으로 꺼지게 했다.

여기서부터 여러 트러블슈팅을 거쳤다.

---

## 트러블슈팅 1: metaField 설정 불일치

Compass에서 컬렉션을 만들면서 `metaField`를 `site_id`(단일 필드)로 지정했는데, 컨슈머 코드는 이걸 모르고 `meta`라는 별도 객체를 만들어서 넣고 있었다.

```java
// 실제로는 아무 의미 없던 코드
Document meta = new Document("site_id", doc.getString("site_id"))
        .append("gateway_id", payload.getString("gateway_id"));
doc.put("meta", meta);
```

`db.getCollectionInfos()`로 확인해보니 실제 metaField는 `site_id` 그 자체였다. `meta` 객체는 버킷팅에 전혀 관여하지 않는 순수 중복 데이터로 저장되고 있었던 것이다. 조회 쿼리도 `meta.site_id` 기준으로 짜뒀던 걸 `site_id`로 정정하고, 컨슈머의 `meta` 객체 생성 로직은 제거했다.

같은 맥락에서 `report_datetime`, `site_id`가 원래 `payload.content.report_datetime`, `payload.site_id`처럼 중첩 구조 안에 있었는데, 시계열의 `timeField`/`metaField`는 **최상위 필드만 지정 가능**하기 때문에 generator에서 이 두 필드를 문서 루트로 승격시켰다.

## 트러블슈팅 2: 버킷이 거의 안 묶이는 문제

가장 크게 시간을 쓴 부분이다. 하루치 목표(4,896,000건) 백필이 진행되던 도중 확인해보니 이랬다.

```js
db.getCollection("system.buckets.iot-state-report").countDocuments({})
// 2,839,407
db.getCollection("iot-state-report").countDocuments({})
// 2,876,686
```

버킷과 문서 수가 거의 1:1. site당 하루 288개 측정치가 한 버킷에 담기길 기대했는데, 사실상 측정치 하나당 버킷 하나가 만들어지고 있었다. `db.serverStatus().bucketCatalog`로 원인을 확인했다.

```js
{
  numBuckets: 9033,
  memoryUsage: 101025072,
  numBucketsClosedDueToMemoryThreshold: 3459670,
  avgNumMeasurementsPerCommit: 1
}
```

`numBucketsClosedDueToMemoryThreshold`가 거의 모든 버킷 생성 건수와 맞먹었다. site 17,000개가 동시에 리포트를 보내는 구조라 항상 17,000개의 버킷이 동시에 열려있어야 하는데, `timeseriesIdleBucketExpiryMemoryUsageThreshold`(유휴 버킷 메모리 임계치) 기본값이 100MB 수준이라 버킷이 채 채워지기도 전에 계속 강제로 닫히고 있었던 것이다.

```js
db.adminCommand({
  setParameter: 1,
  timeseriesIdleBucketExpiryMemoryUsageThreshold: 536870912  // 512MB
})
```

값을 올리고 컬렉션을 초기화한 뒤 재검증하니, 버킷 수가 정확히 17,000개(= site 수)로 나왔다.

```js
db.getCollection("system.buckets.iot-state-report").countDocuments({})
// 17000
```

## 트러블슈팅 3: 처리 속도 개선과 VM 다운

버킷 문제를 해결한 뒤 처리 속도를 올리려고 컨슈머를 `insertOne` 단건 처리에서 **Kafka batch listener + MongoDB `insertMany`**로 전환했다.

```java
@KafkaListener(topics = "${gateway.kafka.topic}", containerFactory = "batchFactory")
public void consume(List<String> messages) {
    List<Document> docs = messages.stream()
            .map(this::toDocument)
            .toList();
    mongoTemplate.getCollection(collectionName).insertMany(docs);
}
```

gateway 쪽도 site 17,000개 생성을 병렬 스트림으로 돌리고, Kafka producer의 `batch-size`, `linger-ms`, `buffer.memory`를 키워서 처리량을 최대로 끌어올렸다. 컨슈머는 `max-poll-records: 5000`으로 한 번에 최대 5,000건씩 처리하게 설정했다.

이 상태로 하루치(490만 건) 전체를 재검증하다가 **MongoDB가 올라간 VM이 통째로 다운**됐다. 직전에 버킷 카탈로그 메모리 임계치를 512MB로 올려둔 상태에서, 극단적으로 튜닝한 처리량이 한꺼번에 몰리면서 VM 메모리가 고갈된 것으로 보였다. `max-poll-records`를 5000에서 300으로 낮춰 한 번에 처리하는 배치 크기를 줄였다.

## 트러블슈팅 4: WiredTiger 캐시와 버킷 카탈로그는 별개다

VM이 다운된 원인을 더 정확히 이해하려고 메모리 구조를 다시 짚었다.

- **WiredTiger 캐시**: MongoDB 전체(문서, 인덱스 등)가 공용으로 쓰는 범용 캐시. `(RAM - 1GB) × 50%`가 기본값이며, 우리 환경에서는 1.39GB로 관측됐다 (역산하면 VM RAM이 약 4GB 수준).
- **`timeseriesIdleBucketExpiryMemoryUsageThreshold`**: 시계열 버킷 카탈로그 전용으로 별도로 잡히는 예산. WiredTiger 캐시와 **합산되는 별개의 메모리 풀**이다.

4GB VM 기준으로 캐시(1.39GB) + 버킷 임계치(512MB)만 해도 이미 1.9GB, 여기에 극단적으로 튜닝한 부하가 겹치면서 남은 여유(약 2GB)로는 버티지 못한 것으로 결론지었다. 이후로는 `2코어 4GB` 규모 기준 임계치를 512MB보다 낮춰(약 256~384MB) 안전 마진을 더 두는 쪽으로 조정했다.

추가로, 시계열 버킷 하나가 자랄 수 있는 최대 크기 자체가 `WiredTiger 캐시 크기 / (2 × 활성 버킷 수)`로 동적으로 계산된다는 것도 확인했다. 활성 버킷이 17,000개인 상태에서는 이 값이 수십 KB 수준까지 작아져서, 캐시가 작으면 버킷이 하루치(288개)를 다 채우기 전에 용량 한도로 닫혀버릴 수 있다.

## 트러블슈팅 5: 데이터 순서 보장 문제

버킷 재검증 도중 site당 버킷 수가 17,000개에서 다시 35,000개, 이후 5만 개 이상으로 늘어나는 현상이 나타났다. `bucketCatalog` 스냅샷을 전/후로 비교해서 원인을 좁혔다.

```js
numBucketsClosedDueToTimeBackward: 14177  // 새로 등장
numBucketsClosedDueToMemoryThreshold: 100538  // 그대로
```

`numBucketsClosedDueToTimeBackward`는 이미 들어간 데이터보다 이른 시각의 데이터가 나중에 도착해서 버킷이 강제로 닫혔다는 뜻이다. 원인은 Kafka producer 설정이었다. 처리량을 위해 `acks: "1"`을 명시했는데, 이러면 Kafka 프로듀서의 idempotence가 조용히 비활성화되고, 기본값인 `max.in.flight.requests.per.connection=5`와 맞물려 **재시도가 발생하면 같은 파티션 안에서도 순서가 뒤바뀔 수 있는 구조**였다. VM 장애로 네트워크가 불안정했던 시점과 겹치면서 실제로 순서 역전이 발생한 것으로 보인다.

1차로 `max.in.flight.requests.per.connection: 1`을 적용해 순서를 강제했더니 이번엔 반대 문제가 생겼다.

```
org.apache.kafka.common.errors.TimeoutException:
Expiring 315 record(s) for sems-test-27:120013 ms has passed since batch creation
```

연결당 요청을 하나씩만 순차 전송하도록 강제하니, 극강으로 튜닝해둔 생성 속도를 파이프라인이 못 따라가서 배치가 120초 안에 전송되지 못하고 타임아웃이 났다. 최종적으로는 `max.in.flight.requests.per.connection` 제한 대신 **`enable.idempotence: true` + `acks: all`** 조합으로 전환했다. 멱등성 프로듀서는 연결당 최대 5개 요청을 동시에 보내면서도 시퀀스 번호로 순서를 보장하기 때문에, 순서 보장과 처리량을 동시에 챙길 수 있는 방향이다.

## 트러블슈팅 6: 시간이 지날수록 CPU가 치솟아 VM이 다운되는 문제

트러블슈팅 3~5를 거쳐 안정화됐다고 보고 하루치(490만 건) 백필을 처음부터 끝까지 다시 돌렸는데, 이번엔 다른 패턴의 장애가 나타났다. 초반엔 멀쩡하다가 시간이 지날수록 MongoDB의 CPU 사용량이 서서히 올라가더니, 결국 VM이 응답 불능 상태에 빠졌다.

**원인: 하루 종일 안 닫히는 버킷**

`granularity: "minutes"`는 내부적으로 `bucketMaxSpanSeconds: 86400`(24시간)으로 설정된다. site별 버킷이 자정부터 자정까지 하루 종일 하나도 안 닫히고 계속 커지기만 한다는 뜻이다.

시계열 버킷은 append-only가 아니라, 측정치가 추가될 때마다 버킷 문서 전체를 다시 압축·인코딩하는 구조다. 버킷에 쌓인 측정치가 많아질수록 이 재인코딩 비용도 커진다. 그래서 하루 초반(버킷당 측정치 몇 개)엔 가볍던 insert가 하루 후반(버킷당 측정치 200개 이상)으로 갈수록 점점 무거워진다. 17,000개 site의 버킷이 전부 이 패턴을 동시에 겪으니, 시간이 지날수록 CPU 사용량이 우상향하는 그래프가 나온 것이다. 발행 주기를 30초→100초로 늦춰봐도 근본적으론 해결되지 않았다 — 페이싱은 순간 처리율을 낮춰 큐 적체를 늦출 뿐, 버킷 자체가 커지는 건 막지 못하기 때문이다.

**해결: 버킷 최대 수명을 1시간으로 축소**

`bucketMaxSpanSeconds`를 하루가 아니라 1시간으로 낮추면, 버킷이 정시마다 강제로 닫히고 새로 열리면서 insert 비용이 주기적으로 리셋된다. 문제는 이 값을 직접 지정하는 `bucketRoundingSeconds`/커스텀 `bucketMaxSpanSeconds` 옵션이 MongoDB 6.3부터 추가된 기능인데, 사용 중인 `mongo:6-jammy` 이미지는 MongoDB 6.0.x라 지원하지 않았다.

```
MongoServerError[IDLUnknownField]: BSON field 'create.timeseries.bucketRoundingSeconds' is an unknown field.
```

대신 `granularity` 프리셋 중 `"seconds"`를 선택해서 우회했다. 실제 리포트 주기는 5분이지만, granularity는 예상 리포트 빈도에 대한 힌트일 뿐 실제 timeField 정밀도를 강제하지는 않는다.

```js
db.createCollection("iot-state-report", {
  timeseries: {
    timeField: "report_datetime",
    metaField: "site_id",
    granularity: "seconds"  // bucketMaxSpanSeconds: 3600
  },
  expireAfterSeconds: 34190000
})
```

이 변경으로 하루 버킷 수는 site당 1개(17,000개/일)에서 24개(408,000개/일)로 늘어나지만, 버킷 하나의 최대 크기가 시간 단위로 캡핑되면서 insert 비용이 하루 종일 우상향하는 대신 매 정시마다 리셋되는 톱니파 패턴으로 바뀐다.

**부록: 임계치 설정이 재시작마다 초기화되던 문제**

이 VM 다운을 조사하다가, 트러블슈팅 2에서 512MB로 올려뒀던 `timeseriesIdleBucketExpiryMemoryUsageThreshold`가 어느 순간 다시 기본값으로 돌아가 있는 걸 발견했다. `docker-compose.yml`에 mongod 설정 파일이나 `command` 오버라이드가 전혀 없어서, `setParameter`로 준 런타임 설정이 어디에도 영구 저장되지 않고 있었던 것이다. `restart: always` 정책 때문에 컨테이너가 크래시 후 자동 재기동될 때마다 이 값이 조용히 초기화되고 있었다.

`docker inspect`로 `OOMKilled: false`가 나와서 처음엔 헷갈렸는데, 컨테이너에 별도 메모리 제한이 없어서 컨테이너 단위가 아니라 **호스트 VM 전체**가 메모리 부족으로 응답 불능에 빠진 것으로 보였다(mongod 로그도 정상 종료 흔적 없이 그냥 뚝 끊겨 있었다 — 외부에서 강제 종료됐다는 신호). 최종적으로 `docker-compose.yml`에 `command`를 추가해서, 재시작 여부와 무관하게 항상 같은 설정으로 뜨도록 고쳤다.

```yaml
command: [
  "mongod",
  "--wiredTigerCacheSizeGB", "1.2",
  "--setParameter", "timeseriesIdleBucketExpiryMemoryUsageThreshold=536870912"
]
```

---

## 용량 비교

`granularity: "seconds"`(1시간 버킷) 설정으로 하루치(490만 건) 백필을 처음부터 끝까지 완주했다.

```js
db.getCollection("iot-state-report").countDocuments({})
// 4896000
db.getCollection("system.buckets.iot-state-report").countDocuments({})
// 408000  (17,000 site × 24버킷/일)
```

`collStats`로 확인한 실제 저장 용량:

| 구분 | 건수 | 용량 | 건당 용량 |
|---|---|---|---|
| 기존 YYYYMMDD 컬렉션 (실측) | 4,841,768 | 약 3.17GB | 약 703B |
| Time Series Collection (실측, 100% 완주) | 4,896,000 | 약 1.74GB | 약 355B |

용량이 **약 45% 줄었다**. 이전에 36% 진행 시점 데이터로 비례 계산했던 예상치(약 1.60GB)와 비슷한 수준으로, 실측치가 조금 더 컸다.

버킷 통계도 설계 의도대로 정확히 맞아떨어졌다.

```js
{
  bucketCount: 408000,
  avgBucketSize: 15334,                       // 약 15KB, 125KB 제한에 여유
  numBucketsOpenedDueToMetadata: 17000,       // site당 최초 1회
  numBucketsClosedDueToTimeForward: 391000,   // 정시 경계 정상 종료 (408000 - 17000)
  numBucketsClosedDueToMemoryThreshold: 0,    // 메모리 임계치로 인한 조기 종료 없음
  numBytesUncompressed: 18532618000,
  numBytesCompressed: 5450507176               // 버킷 레벨 압축률 약 3.4배
}
```

`numBucketsClosedDueToMemoryThreshold: 0`은 트러블슈팅 2에서 겪었던 조기 종료·파편화가 이번엔 전혀 없었다는 뜻이고, site당 버킷 24개(직접 카운트로도 확인)는 설계 의도(1시간 버킷 × 24시간)와 정확히 일치한다.

## 조회 코드 Before / After

가장 처음 문제 삼았던 "날짜별 컬렉션을 순회하는 for문"은 컬렉션이 하나로 통합되면서 완전히 사라졌다.

**Before**

```java
for (String collection : afterDates) {
    LocalDate collectionDate = LocalDate.parse(collection, FORMATTER);
    LocalDateTime startDateTime = collectionDate.atStartOfDay();
    LocalDateTime endDateTime = DateUtil.endOfDay(collectionDate);

    List<IotStateReportDocument> docs = iotStateReportRepository
            .findBySiteIdWithDateRange(siteId, collection, startStr, endStr);

    for (IotStateReportDocument iotDoc : docs) {
        // ... resultList에 데이터 누적
    }
}
```

**After**

```java
LocalDateTime startDateTime = fromDate.atStartOfDay();
LocalDateTime endDateTime = DateUtil.endOfDay(toDate);

List<IotStateReportDocument> docs = iotStateReportRepository
        .findBySiteIdAndReportDatetimeBetween(siteId, startDateTime, endDateTime);

for (IotStateReportDocument iotDoc : docs) {
    // ... resultList에 데이터 누적
}
```

조회 범위가 하루든 한 달이든 컬렉션 이름을 조합하거나 순회할 필요 없이, `site_id` + `report_datetime` 범위 하나로 끝난다. 내부적으로는 `site_id` 인덱스와 버킷의 `control.min`/`control.max`로 관련 버킷만 먼저 골라낸 뒤, 그 버킷들만 풀어서 반환하는 방식이라 컬렉션을 여러 개로 쪼갰을 때보다 조회 로직도, 조회 성능도 개선된다.

## 남은 과제

- `enable.idempotence` + `acks: all` 전환 이후 순서 보장과 처리량이 동시에 확보되는지 재검증
- 버킷을 1시간 단위로 쪼개면서 site당 하루 버킷 수가 1개 → 24개로 늘었는데, 13개월치가 쌓였을 때 "특정 site + 짧은 기간" 조회는 영향이 적어도 "여러 site 동시 조회" 또는 "한 site의 장기간 조회"에서는 버킷 개수가 그만큼 늘어난 만큼 성능이 어떻게 변하는지 실측 필요
- 2vCPU/4GB라는 현재 스펙이 최종 운영 규모(향후 site 수 증가 시)를 감당할 수 있을지 판단 — 부족하면 WiredTiger 캐시 확대(수직 확장) 또는 site_id 기준 샤딩(수평 확장) 검토
- 매니지드 서비스(Atlas 등) 전환 여부 — `setParameter` 계열 세부 튜닝이 막혀있는 대신 장애 복원력(자동 failover)을 얻는 트레이드오프가 있어, 운영 안정성과 튜닝 자유도 중 무엇을 우선할지 결정 필요

시계열 컬렉션 자체는 우리 데이터 패턴에 맞는 선택이었지만, 고카디널리티(site 17,000개) + 동일 주기 리포트라는 조합이 예상보다 메모리·CPU 튜닝에 예민하다는 걸 이번 과정에서 확인했다. 다음 글에서는 idempotence 전환 이후 재검증 결과와 장기 보관 데이터에 대한 넓은 범위 조회 성능 실측치를 정리할 예정이다.
