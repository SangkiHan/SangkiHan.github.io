---
title: MySQL vs Elasticsearch 해시태그 검색 성능 비교 (100만 건 기준)
author: SangkiHan
date: 2026-03-18 14:00:00 +0900
categories: [DB, Elasticsearch]
tags: [DB, Elasticsearch, MySQL, Performance]
---

## 개요

PuppyNote 서비스의 게시물 검색 기능을 개선하면서 MySQL과 Elasticsearch 중 어느 쪽이 더 적합한지 판단하기 위해 성능 벤치마크를 진행했다.

100만 건의 게시물 데이터를 기준으로 **해시태그 정확 검색** 시나리오에서 두 기술의 평균 응답 시간을 비교했다.

---

## 테스트 환경 및 데이터 구성

| 항목 | 내용 |
|---|---|
| 총 게시물 수 | 1,000,000건 |
| 해시태그 풀 | 30종 (강아지, 고양이, 반려동물 등) |
| 게시물당 해시태그 | 1~3개 (랜덤) |
| `강아지` 해시태그 포함 비율 | 약 10% |
| 검색 반복 횟수 | 10회 (평균값 산출) |
| MySQL 인덱스 | `post_hashtags(hashtag)` 인덱스 생성 |

### 테이블 구조 (MySQL)

<details markdown="1">
<summary>DDL 펼치기</summary>

```sql
CREATE TABLE posts (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id      BIGINT       NOT NULL,
    content      TEXT,
    created_date DATETIME,
    updated_date DATETIME
);

CREATE TABLE post_hashtags (
    post_id BIGINT      NOT NULL,
    hashtag VARCHAR(50) NOT NULL,
    INDEX idx_bm_post_hashtags (hashtag)
);
```

</details>

### ES Document 구조 — PostDocument

ES에 저장되는 문서는 `@Document(indexName = "posts")`로 선언된 `PostDocument` 클래스로 정의한다.
각 필드에 붙은 `@Field` 어노테이션이 **매핑 타입을 결정**하고, 타입에 따라 역인덱싱 여부가 달라진다.

```java
@Document(indexName = "posts")
@Setting(settingPath = "es-settings.json")
public class PostDocument {

    @Id
    private String id;

    @Field(type = FieldType.Long)
    private Long postId;

    @Field(type = FieldType.Long)
    private Long userId;

    @Field(type = FieldType.Keyword)
    private String userNickname;

    @Field(type = FieldType.Text, analyzer = "nori")
    private String content;

    @Field(type = FieldType.Keyword)
    private List<String> hashtags;          // ← 역인덱싱 대상

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdDate;
}
```

#### 필드 타입별 저장 방식

| 필드 | 타입 | 역인덱싱 | 설명 |
|---|---|:---:|---|
| `hashtags` | `Keyword` | **O** | 값 전체를 토큰 하나로 저장 — `term query` 정확 매칭 |
| `content` | `Text` (nori) | **O** | 형태소 분해 후 각 토큰을 역인덱싱 — `match query` |
| `userId`, `postId` | `Long` | X | BKD Tree 저장 — 범위 검색 / `filter` 조건으로 사용 |
| `createdDate` | `Date` | X | BKD Tree 저장 — 정렬 / 범위 검색 |

#### Keyword vs Text 역인덱싱 차이

- **`Keyword`** — 값 전체를 그대로 하나의 토큰으로 인덱싱
  ```
  "강아지스타그램" → 토큰: ["강아지스타그램"]
  ```
  `term query`로 정확히 일치하는 문서만 검색한다. 해시태그처럼 **정확 매칭**이 필요한 경우에 적합하다.

- **`Text` + nori 분석기** — 형태소 단위로 분해 후 각 토큰을 역인덱싱
  ```
  "오늘 우리 강아지 귀여워" → 토큰: ["오늘", "우리", "강아지", "귀엽"]
  ```
  `match query`로 부분 단어 검색이 가능하다. 자유 텍스트 검색에 적합하다.

이번 벤치마크의 `hashtags` 필드는 `Keyword` 타입이므로 `term query`가 역인덱싱을 그대로 탐색해 **O(1)에 가까운 조회**를 한다. 이것이 ES가 압도적으로 빠른 핵심 이유다.

<details markdown="1">
<summary>JSON Document 예시 펼치기</summary>

```json
{
  "postId": 1,
  "userId": 33,
  "userNickname": "벤치마크테스터",
  "content": "오늘 우리 포메라니안 너무 귀여워서 사진 찍었어요!",
  "hashtags": ["강아지", "포메라니안"],
  "createdDate": "2025-03-18T13:00:00"
}
```

</details>

---

## 데이터 삽입 코드

100만 건의 게시물을 MySQL과 Elasticsearch에 동시에 적재한다.
메모리 절약을 위해 배치 단위 multi-row INSERT와 cursor 기반 페이지네이션을 사용했다.

### MySQL — 게시물 + 해시태그 배치 삽입

<details markdown="1">
<summary>insertPostsToMysql() 펼치기</summary>

```java
private static final int TOTAL_POSTS  = 1_000_000;
private static final int POST_BATCH   = 5_000;
private static final int HASHTAG_BATCH = 10_000;

// POST_BATCH 행짜리 multi-row INSERT SQL
private static final String POST_MULTI_INSERT =
        "INSERT INTO posts (user_id, content, created_date, updated_date) VALUES " +
        String.join(",", Collections.nCopies(POST_BATCH, "(?,?,?,?)"));

private void insertPostsToMysql() {
    log.info("[MySQL] 게시물 삽입 시작...");
    long start = System.currentTimeMillis();
    LocalDateTime baseTime = LocalDateTime.now().minusDays(365);
    int totalBatches = TOTAL_POSTS / POST_BATCH;

    // 1단계: posts multi-row INSERT
    Object[] postParams = new Object[POST_BATCH * 4];
    for (int b = 0; b < totalBatches; b++) {
        for (int i = 0; i < POST_BATCH; i++) {
            String pet     = PET_NAMES.get(random.nextInt(PET_NAMES.size()));
            String content = String.format(TEMPLATES.get(random.nextInt(TEMPLATES.size())), pet);
            LocalDateTime createdAt = baseTime.plusSeconds((long) b * POST_BATCH + i);
            int idx = i * 4;
            postParams[idx]     = testUserId;
            postParams[idx + 1] = content;
            postParams[idx + 2] = createdAt;
            postParams[idx + 3] = createdAt;
        }
        jdbcTemplate.update(POST_MULTI_INSERT, postParams);
        log.info("[MySQL] 게시물 삽입 진행: " + ((long)(b + 1) * POST_BATCH) + "/" + TOTAL_POSTS);
    }
    log.info("[MySQL] 게시물 " + TOTAL_POSTS + "건 삽입 완료 (" + (System.currentTimeMillis() - start) + "ms)");

    // 2단계: post_hashtags 삽입 (ID를 청크로 조회하여 메모리 절약)
    log.info("[MySQL] 해시태그 삽입 시작...");
    start = System.currentTimeMillis();

    final int ID_CHUNK = 50_000;
    List<Long>   pidBuf  = new ArrayList<>(HASHTAG_BATCH + 10);
    List<String> tagBuf  = new ArrayList<>(HASHTAG_BATCH + 10);
    long totalHashtags = 0;
    int  idOffset = 0;

    while (true) {
        List<Long> chunk = jdbcTemplate.queryForList(
                "SELECT id FROM posts WHERE user_id = ? ORDER BY id ASC LIMIT ? OFFSET ?",
                Long.class, testUserId, ID_CHUNK, idOffset
        );
        if (chunk.isEmpty()) break;

        for (Long postId : chunk) {
            Set<String> selected = new LinkedHashSet<>();

            // '강아지' 태그를 약 10% 확률로 강제 포함 → 약 10만 건이 해당 태그 보유
            if (random.nextInt(10) == 0) selected.add(TARGET_HASHTAG);

            int tagCount = random.nextInt(3) + 1;  // 1~3개
            while (selected.size() < tagCount) {
                selected.add(HASHTAG_POOL.get(random.nextInt(HASHTAG_POOL.size())));
            }

            for (String tag : selected) {
                pidBuf.add(postId);
                tagBuf.add(tag);
            }

            if (pidBuf.size() >= HASHTAG_BATCH) {
                totalHashtags += pidBuf.size();
                bulkInsertHashtags(pidBuf, tagBuf);
                pidBuf.clear();
                tagBuf.clear();
            }
        }
        idOffset += ID_CHUNK;
        log.info("[MySQL] 해시태그 진행 중: post " + idOffset + "/" + TOTAL_POSTS + " 처리 완료");
    }

    if (!pidBuf.isEmpty()) {
        totalHashtags += pidBuf.size();
        bulkInsertHashtags(pidBuf, tagBuf);
    }
    log.info("[MySQL] 해시태그 " + totalHashtags + "건 삽입 완료 (" + (System.currentTimeMillis() - start) + "ms)");
}

/** 가변 행 수의 post_hashtags multi-row INSERT */
private void bulkInsertHashtags(List<Long> pids, List<String> tags) {
    String sql = "INSERT INTO post_hashtags (post_id, hashtag) VALUES " +
            String.join(",", Collections.nCopies(pids.size(), "(?,?)"));
    Object[] params = new Object[pids.size() * 2];
    for (int i = 0; i < pids.size(); i++) {
        params[i * 2]     = pids.get(i);
        params[i * 2 + 1] = tags.get(i);
    }
    jdbcTemplate.update(sql, params);
}
```

</details>

### Elasticsearch — 벌크 인덱싱

<details markdown="1">
<summary>indexPostsToElasticsearch() 펼치기</summary>

```java
private static final int ES_BATCH = 1_000;

private void indexPostsToElasticsearch() {
    log.info("[ES] 인덱싱 시작...");
    long start = System.currentTimeMillis();
    long lastId = 0;
    long totalIndexed = 0;

    while (true) {
        // OFFSET 대신 cursor(lastId) 기반 페이지네이션 — 후반부 메모리/속도 문제 해결
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
                "SELECT id, content, created_date FROM posts " +
                "WHERE user_id = ? AND id > ? ORDER BY id ASC LIMIT ?",
                testUserId, lastId, ES_BATCH
        );
        if (rows.isEmpty()) break;

        List<Long> batchIds = rows.stream()
                .map(r -> ((Number) r.get("id")).longValue())
                .toList();
        lastId = batchIds.get(batchIds.size() - 1);

        // 해당 배치의 해시태그 조회
        String placeholders = String.join(",", Collections.nCopies(batchIds.size(), "?"));
        List<Map<String, Object>> hashtagRows = jdbcTemplate.queryForList(
                "SELECT post_id, hashtag FROM post_hashtags WHERE post_id IN (" + placeholders + ")",
                batchIds.toArray()
        );

        Map<Long, List<String>> hashtagMap = new HashMap<>(hashtagRows.size() * 2);
        for (Map<String, Object> row : hashtagRows) {
            Long postId = ((Number) row.get("post_id")).longValue();
            hashtagMap.computeIfAbsent(postId, k -> new ArrayList<>()).add((String) row.get("hashtag"));
        }

        // ES 벌크 인덱싱
        List<IndexQuery> queries = new ArrayList<>(rows.size());
        for (Map<String, Object> row : rows) {
            Long postId = ((Number) row.get("id")).longValue();
            Object rawDate = row.get("created_date");
            LocalDateTime createdDate = rawDate instanceof java.sql.Timestamp ts
                    ? ts.toLocalDateTime() : (LocalDateTime) rawDate;

            PostDocument doc = PostDocument.builder()
                    .postId(postId)
                    .userId(testUserId)
                    .userNickname("벤치마크테스터")
                    .content((String) row.get("content"))
                    .hashtags(hashtagMap.getOrDefault(postId, List.of()))
                    .createdDate(createdDate)
                    .build();

            queries.add(new IndexQueryBuilder()
                    .withId(String.valueOf(postId))
                    .withObject(doc)
                    .build());
        }

        elasticsearchOperations.bulkIndex(queries, PostDocument.class);
        totalIndexed += rows.size();

        if (totalIndexed % 100_000 == 0) {
            log.info("[ES] 인덱싱 진행: " + totalIndexed + "/" + TOTAL_POSTS);
        }
    }

    // 즉시 검색 가능하도록 refresh
    elasticsearchOperations.indexOps(PostDocument.class).refresh();
    log.info("[ES] 인덱싱 완료 - " + totalIndexed + "건 / " + (System.currentTimeMillis() - start) + "ms");
}
```

</details>

### 삽입 전략 포인트

| 항목 | 전략 | 이유 |
|---|---|---|
| posts 삽입 | multi-row INSERT (5,000건 배치) | 단건 INSERT 대비 네트워크 왕복 횟수 최소화 |
| post_hashtags 삽입 | ID 청크(50,000) 조회 후 배치 INSERT | 전체 ID를 메모리에 올리지 않아 OOM 방지 |
| ES 인덱싱 | cursor 기반 페이지네이션 + bulkIndex(1,000) | OFFSET 방식은 후반부로 갈수록 느려지는 문제 회피 |
| ES refresh | 인덱싱 완료 후 1회 강제 refresh | 벤치마크 전 모든 문서가 검색 가능한 상태 보장 |

---

## 벤치마크 코드

<details markdown="1">
<summary>전체 벤치마크 테스트 코드 펼치기</summary>

```java
@Test
@Order(1)
@DisplayName("[벤치마크 1] 해시태그 정확 검색: MySQL JOIN vs ES term query")
void benchmarkHashtagExactSearch() {
    log.info("\n========== [벤치마크 1] 해시태그 정확 검색 ==========");
    log.info("키워드: '" + TARGET_HASHTAG + "' | 반복: " + ITERATIONS + "회");

    // --- MySQL: posts JOIN post_hashtags WHERE hashtag = ? ---
    long mysqlTotalMs = 0;
    long mysqlMatchCount = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        long s = System.nanoTime();

        mysqlMatchCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(DISTINCT p.id) FROM posts p " +
                        "JOIN post_hashtags ph ON p.id = ph.post_id " +
                        "WHERE ph.hashtag = ?",
                Long.class, TARGET_HASHTAG
        );
        jdbcTemplate.queryForList(
                "SELECT p.id FROM posts p " +
                        "JOIN post_hashtags ph ON p.id = ph.post_id " +
                        "WHERE ph.hashtag = ? " +
                        "ORDER BY p.created_date DESC LIMIT 20",
                Long.class, TARGET_HASHTAG
        );

        mysqlTotalMs += (System.nanoTime() - s) / 1_000_000;
    }

    // --- ES: term query on hashtags field ---
    long esTotalMs = 0;
    long esMatchCount = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        long s = System.nanoTime();

        NativeQuery query = NativeQuery.builder()
                .withQuery(q -> q.term(t -> t.field("hashtags").value(TARGET_HASHTAG)))
                .withPageable(PageRequest.of(0, 20))
                .withSort(Sort.by(Sort.Direction.DESC, "createdDate"))
                .build();
        SearchHits<PostDocument> hits = elasticsearchOperations.search(query, PostDocument.class);

        esTotalMs += (System.nanoTime() - s) / 1_000_000;
        esMatchCount = hits.getTotalHits();
    }

    printCompareResult("MySQL JOIN", mysqlTotalMs, mysqlMatchCount,
            "ES term query", esTotalMs, esMatchCount);
}
```

</details>

---

## 벤치마크 1 — 해시태그 정확 검색

키워드 `강아지`로 정확 매칭 검색을 수행한다.
각 반복마다 **COUNT 쿼리 + SELECT 쿼리** 2개를 순서대로 실행했다.

### MySQL 쿼리

```sql
-- 총 건수 조회
SELECT COUNT(DISTINCT p.id)
FROM posts p
JOIN post_hashtags ph ON p.id = ph.post_id
WHERE ph.hashtag = '강아지';

-- 최신순 20건 조회
SELECT p.id
FROM posts p
JOIN post_hashtags ph ON p.id = ph.post_id
WHERE ph.hashtag = '강아지'
ORDER BY p.created_date DESC
LIMIT 20;
```

### Elasticsearch 쿼리

```java
NativeQuery query = NativeQuery.builder()
    .withQuery(q -> q.term(t -> t.field("hashtags").value("강아지")))
    .withPageable(PageRequest.of(0, 20))
    .withSort(Sort.by(Sort.Direction.DESC, "createdDate"))
    .build();
```

---

## 실행 로그

<details markdown="1">
<summary>전체 실행 로그 펼치기</summary>

```
13:11:13.389 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest -
========== [벤치마크 1] 해시태그 정확 검색 ==========
13:11:13.389 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest - 키워드: '강아지' | 반복: 10회

-- [1회차] COUNT
13:11:26.347 [Test worker] INFO  p6spy -
Execute DML :
    SELECT COUNT(DISTINCT p.id)
    FROM posts p
    JOIN post_hashtags ph ON p.id = ph.post_id
    WHERE ph.hashtag = '강아지'
Execution Time: 12944 ms

-- [1회차] SELECT
13:11:26.647 [Test worker] INFO  p6spy -
Execute DML :
    SELECT p.id
    FROM posts p
    JOIN post_hashtags ph ON p.id = ph.post_id
    WHERE ph.hashtag = '강아지'
    ORDER BY p.created_date DESC
    LIMIT 20
Execution Time: 294 ms

-- [2회차] COUNT
13:11:27.503 [Test worker] INFO  p6spy - Execution Time: 855 ms
-- [2회차] SELECT
13:11:28.158 [Test worker] INFO  p6spy - Execution Time: 653 ms

-- [3회차] COUNT
13:11:28.475 [Test worker] INFO  p6spy - Execution Time: 314 ms
-- [3회차] SELECT
13:11:29.497 [Test worker] INFO  p6spy - Execution Time: 1021 ms

-- [4회차] COUNT
13:11:29.833 [Test worker] INFO  p6spy - Execution Time: 334 ms
-- [4회차] SELECT
13:11:30.245 [Test worker] INFO  p6spy - Execution Time: 410 ms

-- [5회차] COUNT
13:11:30.449 [Test worker] INFO  p6spy - Execution Time: 204 ms
-- [5회차] SELECT
13:11:31.136 [Test worker] INFO  p6spy - Execution Time: 685 ms

-- [6회차] COUNT
13:11:31.417 [Test worker] INFO  p6spy - Execution Time: 279 ms
-- [6회차] SELECT
13:11:31.764 [Test worker] INFO  p6spy - Execution Time: 345 ms

-- [7회차] COUNT
13:11:32.517 [Test worker] INFO  p6spy - Execution Time: 751 ms
-- [7회차] SELECT
13:11:32.727 [Test worker] INFO  p6spy - Execution Time: 207 ms

-- [8회차] COUNT
13:11:33.378 [Test worker] INFO  p6spy - Execution Time: 651 ms
-- [8회차] SELECT
13:11:33.651 [Test worker] INFO  p6spy - Execution Time: 271 ms

-- [9회차] COUNT
13:11:33.926 [Test worker] INFO  p6spy - Execution Time: 272 ms
-- [9회차] SELECT
13:11:34.228 [Test worker] INFO  p6spy - Execution Time: 302 ms

-- [10회차] COUNT
13:11:34.550 [Test worker] INFO  p6spy - Execution Time: 320 ms
-- [10회차] SELECT
13:11:35.286 [Test worker] INFO  p6spy - Execution Time: 735 ms

13:11:36.037 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest - --- 결과 (10 반복 평균) ---
13:11:36.037 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest - MySQL JOIN   | 평균 2189ms | 매칭: 10000건
13:11:36.037 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest - ES term query | 평균 75ms  | 매칭: 10000건
13:11:36.037 [Test worker] INFO  c.p.c.p.SearchPerformanceBenchmarkTest - >>> ES가 MySQL보다 29.2배 빠름
```

</details>

---

## 측정 결과

### 회차별 MySQL 실행 시간

| 회차 | COUNT (ms) | SELECT (ms) | 합계 (ms) |
|:---:|---:|---:|---:|
| 1 | 12,944 | 294 | **13,238** |
| 2 | 855 | 653 | 1,508 |
| 3 | 314 | 1,021 | 1,335 |
| 4 | 334 | 410 | 744 |
| 5 | 204 | 685 | 889 |
| 6 | 279 | 345 | 624 |
| 7 | 751 | 207 | 958 |
| 8 | 651 | 271 | 922 |
| 9 | 272 | 302 | 574 |
| 10 | 320 | 735 | 1,055 |

> 1회차에 12,944ms가 소요된 것은 **버퍼 풀 워밍업이 되지 않은 콜드 스타트** 상태였기 때문이다.
> 2회차부터는 InnoDB 버퍼 풀에 데이터가 캐싱되면서 응답 시간이 크게 줄었다.

### 최종 비교 (1회차 콜드 스타트 제외)

| 방식 | 평균 응답 시간 (Warm Start) | 매칭 건수   |
|---|---|---------|
| MySQL JOIN | **957 ms** | 10,000건 |
| ES term query | **75 ms** | 10,000건 |

> **ES가 MySQL보다 약 12.8배 빠름 (워밍업된 상태 기준)**

---

## 분석

### MySQL이 느린 이유

1. **JOIN 비용**: `posts`와 `post_hashtags`를 `post_id`로 JOIN하면서 대량의 row를 처리한다.
2. **COUNT(DISTINCT)**: 해시태그 조건에 맞는 post_id 전체를 읽어 중복 제거를 수행하므로 인덱스를 타더라도 집계 비용이 크다.
3. **버퍼 풀 의존성**: 캐시가 없는 상태(콜드 스타트)에서는 디스크 I/O가 발생해 응답 시간이 급격히 늘어난다. (1회차 13.2초)
4. **행 기반 저장**: MySQL은 행(row) 단위로 데이터를 저장하므로 특정 컬럼 기준 집계가 상대적으로 비효율적이다.

### Elasticsearch가 빠른 이유

1. **역색인(Inverted Index)**: ES는 `hashtags` 필드에 역색인을 구성하므로 `강아지`를 포함하는 문서를 O(1)에 가깝게 찾아낸다.
2. **term query**: 형태소 분석 없이 정확 매칭만 하므로 오버헤드가 거의 없다.
3. **메모리 우선 아키텍처**: Lucene 세그먼트가 메모리에 로드되어 있어 디스크 I/O 의존도가 낮다.
4. **페이지네이션 내장**: `from/size` 기반으로 상위 20건만 효율적으로 반환한다.

---

## MySQL 1회차 콜드 스타트 문제

눈에 띄는 점은 1회차 COUNT 쿼리에서 **12,944ms**가 소요됐다는 것이다.

이는 MySQL InnoDB 버퍼 풀이 비어 있는 상태에서 100만 건의 데이터를 디스크에서 읽어야 했기 때문이다.
2회차부터 동일 쿼리가 204~855ms로 내려온 것을 보면 버퍼 풀 캐싱 효과가 상당하다.

따라서 실질적인 성능 비교를 위해 **1회차를 제외한 2~10회차의 평균을 내면 MySQL은 약 957ms**가 소요된다. 이 경우에도 Elasticsearch(75ms)가 약 **12.8배** 더 빠른 성능을 보여준다.

운영 환경에서는 서버 재시작 직후나 버퍼 풀 크기가 부족한 경우 이런 콜드 스타트 문제가 실사용자 경험에 영향을 줄 수 있다.

반면 Elasticsearch는 Lucene 세그먼트 캐시 덕분에 첫 요청부터 일관된 빠른 응답을 보인다.

---

## 결론

| 항목 | MySQL | Elasticsearch |
|---|---|---|
| 해시태그 정확 검색 | 평균 957ms (Warm) | 평균 75ms |
| 콜드 스타트 민감도 | 높음 | 낮음 |
| 구성 복잡도 | 낮음 | 높음 |
| 운영 비용 | 낮음 | 높음 |

100만 건 규모에서 해시태그 검색은 (워밍업된 상태 기준) Elasticsearch가 MySQL 대비 약 **13배** 빠른 것으로 측정됐다.

검색 트래픽이 높거나 데이터가 계속 증가하는 서비스라면 Elasticsearch 도입이 합리적인 선택이다.
단, 운영 복잡도와 인프라 비용이 추가되므로 데이터 규모와 검색 빈도를 고려해 도입 여부를 결정해야 한다.
