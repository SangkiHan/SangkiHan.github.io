---
title: Redis Write-Behind Cache로 게시물 좋아요 구현하기
author: SangkiHan
date: 2026-03-24 10:00:00 +0900
categories: [Spring, Redis]
tags: [Redis, Spring Boot, Cache, Write-Behind, Lua Script, Batch]
---

## 개요

PuppyNote 서비스의 게시물 좋아요 기능을 구현하면서 **DB 부하 최소화**와 **실시간 반영**이라는 두 가지 요구사항을 동시에 만족시켜야 했다.

단순히 좋아요를 누를 때마다 DB에 바로 쓰는 방식은 트래픽이 몰릴 때 DB 부하가 급격히 증가한다.
이를 해결하기 위해 **Redis Write-Behind Cache 패턴**을 적용했다.

---

## 문제 정의

### 단순 DB 직접 쓰기 방식의 문제

```
[유저 좋아요 클릭] → INSERT/DELETE → MySQL
```

- 좋아요/취소 이벤트마다 DB write 발생
- 같은 게시물에 트래픽이 몰리면 DB 병목

### 요구사항

1. 좋아요 토글 응답이 빠를 것
2. 새로고침해도 좋아요 여부와 좋아요 수가 정확히 보일 것
3. 결국 DB에는 정확히 반영될 것

---

## 전체 아키텍처

```
[좋아요 토글] → Redis (Lua 스크립트 원자적 처리)
                    ↓
              [1분마다 Batch]
                    ↓
                  MySQL
```

### Redis 키 구조

| 키 | 타입 | 용도 |
|---|---|---|
| `user:liked:{userId}:{postId}` | String | 특정 유저의 좋아요 여부 캐시 |
| `post:like:count:{postId}` | String | 게시물의 총 좋아요 수 |
| `post:like:dirty` | Set | 변경이 발생한 postId 목록 |
| `post:like:delta:add:{postId}` | Set | 이번 주기에 좋아요한 userId 목록 |
| `post:like:delta:remove:{postId}` | Set | 이번 주기에 좋아요 취소한 userId 목록 |

---

## 1단계: 좋아요 토글 (Redis Lua 스크립트)

좋아요 토글은 여러 Redis 명령이 원자적으로 실행되어야 한다.
Redis의 **Lua 스크립트**를 활용하면 여러 명령을 단일 트랜잭션처럼 처리할 수 있다.

### 캐시 초기화

토글 전에 Redis에 캐시가 없으면 DB에서 현재 상태를 읽어 캐시를 워밍업한다.

```java
private void initializeCacheIfAbsent(Long postId, Long userId) {
    if (!postLikeRedisService.existsCountCache(postId)) {
        long count = postLikeRepository.countByPostId(postId);
        postLikeRedisService.setCountCache(postId, count);
    }
    if (!postLikeRedisService.existsLikedCache(userId, postId)) {
        boolean liked = postLikeRepository.findByPostIdAndUserId(postId, userId).isPresent();
        postLikeRedisService.setLikedCache(userId, postId, liked);
    }
}
```

### Lua 스크립트 원자적 토글

```lua
local likedKey    = KEYS[1]  -- user:liked:{userId}:{postId}
local countKey    = KEYS[2]  -- post:like:count:{postId}
local dirtyKey    = KEYS[3]  -- post:like:dirty
local deltaAddKey = KEYS[4]  -- post:like:delta:add:{postId}
local deltaRemKey = KEYS[5]  -- post:like:delta:remove:{postId}

local userId   = ARGV[1]
local postId   = ARGV[2]
local likedTtl = tonumber(ARGV[3])
local countTtl = tonumber(ARGV[4])

local current = redis.call('GET', likedKey)
local liked
local likeCount

if current == '1' then
    -- 좋아요 취소
    redis.call('SET', likedKey, '0', 'EX', likedTtl)
    likeCount = redis.call('DECR', countKey)
    redis.call('SADD', deltaRemKey, userId)
    redis.call('SREM', deltaAddKey, userId)
    liked = 0
else
    -- 좋아요
    redis.call('SET', likedKey, '1', 'EX', likedTtl)
    likeCount = redis.call('INCR', countKey)
    redis.call('SADD', deltaAddKey, userId)
    redis.call('SREM', deltaRemKey, userId)
    liked = 1
end

redis.call('EXPIRE', countKey, countTtl)
redis.call('SADD', dirtyKey, postId)
return {liked, likeCount}
```

**핵심 동작:**
- `user:liked:{userId}:{postId}` 값을 읽어 현재 상태 판단
- 상태에 따라 INCR/DECR로 좋아요 수 즉시 반영
- `delta:add` 또는 `delta:remove` Set에 userId 추가 (배치에서 DB 동기화 시 사용)
- `post:like:dirty` Set에 postId 추가 (배치 처리 대상 마킹)
- 모든 과정이 단일 Lua 스크립트로 원자적 실행

---

## 2단계: 게시물 조회 시 Redis 우선 읽기

좋아요를 눌렀을 때 Redis에 즉시 반영되므로, **조회도 Redis에서 먼저 읽어야** 새로고침 시 정확한 값을 보여줄 수 있다.

### 단건 조회

```java
private LikeInfo resolveLikeInfo(Long postId, Long currentUserId) {
    try {
        Optional<Boolean> cachedLiked = currentUserId != null
                ? postLikeRedisService.getLikedStatus(postId, currentUserId)
                : Optional.empty();
        Optional<Long> cachedCount = postLikeRedisService.getLikeCount(postId);

        if (cachedLiked.isPresent() && cachedCount.isPresent()) {
            return new LikeInfo(cachedCount.get(), cachedLiked.get());
        }
    } catch (Exception e) {
        log.warn("[좋아요 Redis 조회 실패] postId={}, DB fallback 처리", postId, e);
    }

    // Redis 미스 또는 연결 실패 → DB fallback
    PostLikeAggDto likeAgg = postLikeRepository
            .findLikeAggByPostIds(List.of(postId), currentUserId != null ? currentUserId : 0L)
            .stream().findFirst().orElse(null);

    return new LikeInfo(
            likeAgg != null ? likeAgg.getLikeCount() : 0L,
            likeAgg != null && likeAgg.getIsLikedCount() > 0
    );
}
```

### 목록 조회 (N+1 방지 MGET 배치)

게시물 목록 조회 시 N번의 Redis 조회를 방지하기 위해 **MGET**으로 한 번에 처리한다.

```java
private Map<Long, LikeInfo> resolveLikeInfoBatch(List<Long> postIds, Long currentUserId) {
    Map<Long, Boolean> redisLikedMap = Map.of();
    Map<Long, Long> redisCountMap = Map.of();

    try {
        redisLikedMap = currentUserId != null
                ? postLikeRedisService.getLikedStatusBatch(postIds, currentUserId)
                : Map.of();
        redisCountMap = postLikeRedisService.getLikeCountBatch(postIds);
    } catch (Exception e) {
        log.warn("[좋아요 Redis 배치 조회 실패] DB fallback 처리", e);
    }

    // Redis에서 liked와 count 둘 다 hit한 postId는 DB 조회 제외
    List<Long> dbFallbackIds = postIds.stream()
            .filter(id -> !redisLikedMap.containsKey(id) || !redisCountMap.containsKey(id))
            .toList();

    // 캐시 미스 항목만 DB 조회
    Map<Long, PostLikeAggDto> likeAggMap = new HashMap<>();
    if (!dbFallbackIds.isEmpty()) {
        postLikeRepository.findLikeAggByPostIds(dbFallbackIds, currentUserId != null ? currentUserId : 0L)
                .forEach(dto -> likeAggMap.put(dto.getPostId(), dto));
    }
    // ...
}
```

---

## 3단계: 배치 DB 동기화 (Write-Behind)

1분마다 실행되는 스케줄러가 dirty Set에 쌓인 postId들을 DB에 동기화한다.

### RENAME을 이용한 원자적 분리

배치 처리 중에도 새로운 좋아요 이벤트가 계속 들어올 수 있다.
`RENAME` 명령으로 처리할 대상을 분리하면 **처리 중 새 이벤트는 원본 키에 계속 쌓이고**, 배치는 분리된 키만 처리한다.

```java
@Transactional
@Scheduled(fixedRate = 60000)
public void syncLikesToDB() {
    if (!redisService.hasKey(PostLikeRedisKey.DIRTY.of())) {
        return;
    }

    // RENAME으로 dirty set을 원자적으로 분리
    String processingKey = PostLikeRedisKey.DIRTY_PROCESSING.of(System.currentTimeMillis());
    redisService.rename(PostLikeRedisKey.DIRTY.of(), processingKey);

    Set<String> dirtyPostIds = redisService.sMembers(processingKey);
    redisService.delete(processingKey);

    for (String postIdStr : dirtyPostIds) {
        try {
            syncPost(Long.parseLong(postIdStr));
        } catch (Exception e) {
            log.error("[좋아요 배치 동기화] 게시물 동기화 실패 postId={}", postIdStr, e);
            // 실패한 postId는 다음 사이클에서 재시도
            redisService.sAdd(PostLikeRedisKey.DIRTY.of(), postIdStr);
        }
    }
}
```

### delta Set으로 최소 DB 쓰기

`delta:add`, `delta:remove` Set을 사용하면 DB에 변경된 userId만 정확히 처리할 수 있다.

```java
private void syncPost(Long postId) {
    long timestamp = System.currentTimeMillis();

    // delta set을 원자적으로 분리 (처리 중 새 변경은 원본 키에 계속 쌓임)
    Set<String> deltaAdd = postLikeRedisService.popDeltaAdd(postId, timestamp);
    Set<String> deltaRemove = postLikeRedisService.popDeltaRemove(postId, timestamp);

    if (deltaAdd.isEmpty() && deltaRemove.isEmpty()) return;

    List<Long> toInsert = deltaAdd.stream().map(Long::parseLong).toList();
    List<Long> toDelete = deltaRemove.stream().map(Long::parseLong).toList();

    // INSERT IGNORE: 동일 주기 내 중복 삽입 시도 무시
    toInsert.forEach(userId -> postLikeRepository.insertIgnore(postId, userId));

    if (!toDelete.isEmpty()) {
        postLikeRepository.deleteByPostIdAndUserIdIn(postId, toDelete);
    }
}
```

**INSERT IGNORE** 쿼리를 사용해 like → unlike → like 같은 동일 주기 내 중복 시도를 DB 레벨에서 안전하게 처리한다.

```java
@Modifying
@Query(value = "INSERT IGNORE INTO post_likes (post_id, user_id, created_date, updated_date) " +
               "VALUES (:postId, :userId, NOW(), NOW())", nativeQuery = true)
void insertIgnore(@Param("postId") Long postId, @Param("userId") Long userId);
```

---

## 전체 흐름 요약

```
[유저 좋아요 클릭]
        ↓
① initializeCacheIfAbsent() - Redis 캐시 없으면 DB에서 로드
        ↓
② Lua 스크립트 원자적 실행
   - user:liked:{userId}:{postId} 토글 (0 ↔ 1)
   - post:like:count:{postId} INCR/DECR
   - delta:add 또는 delta:remove에 userId 추가
   - dirty Set에 postId 추가
        ↓
③ [응답 반환] liked, likeCount 즉시 반환

[게시물 조회]
        ↓
① Redis에서 user:liked, post:like:count 조회 (MGET)
② 캐시 미스 or Redis 장애 → DB fallback

[1분마다 Batch]
        ↓
① dirty Set RENAME으로 처리 대상 분리
② 각 postId의 delta:add, delta:remove RENAME으로 분리
③ INSERT IGNORE (add), DELETE IN (remove) DB 반영
```

---

## Redis 장애 대응

Redis 연결이 실패해도 서비스가 정상 동작하도록 모든 Redis 조회에 try-catch와 DB fallback을 적용했다.

```java
try {
    // Redis 조회 시도
} catch (Exception e) {
    log.warn("[좋아요 Redis 조회 실패] DB fallback 처리", e);
}
// DB fallback
```

Redis가 내려가면 성능은 저하되지만 **서비스 가용성은 유지**된다.

---

## Nginx SSL TCP 프록시

Redis 서버가 외부 도메인(`db.puppynote.co.kr:6380`)으로 노출되어 있어 Nginx에서 SSL 터미네이션을 처리한다.
Redis는 HTTP가 아닌 TCP 프로토콜이므로 Nginx `stream` 모듈을 사용해야 한다.

```nginx
stream {
    server {
        listen 6380 ssl;
        ssl_certificate     /etc/letsencrypt/live/db.puppynote.co.kr/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/db.puppynote.co.kr/privkey.pem;
        proxy_pass localhost:6379;  # 내부 Redis로 TCP 전달
    }
}
```

Spring 앱에서는 `ssl.enabled: true` 설정으로 TLS 연결:

```yaml
# application-prd.yml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6380
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

---

## 결론

| 항목 | 기존 방식 | Write-Behind Cache |
|---|---|---|
| 좋아요 토글 DB 쓰기 | 매번 즉시 | 1분마다 배치 |
| 새로고침 시 좋아요 반영 | DB 정확 | Redis 실시간 캐시 |
| Redis 장애 시 | - | DB fallback으로 정상 운영 |
| 동시성 안전성 | DB 트랜잭션 의존 | Lua 스크립트 원자적 처리 |

Write-Behind Cache 패턴은 **쓰기 빈도가 높고 즉각적인 반영이 필요한** 좋아요, 조회수, 투표 같은 기능에 효과적이다.
