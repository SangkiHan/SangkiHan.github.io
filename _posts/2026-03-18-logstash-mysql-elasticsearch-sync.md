---
title: Logstash로 MySQL → Elasticsearch 실시간 동기화 구축
author: SangkiHan
date: 2026-03-18 16:00:00 +0900
categories: [Database, Elasticsearch]
tags: [Logstash, MySQL, Elasticsearch, 동기화]
---

## 개요

[이전 포스트](/posts/mysql-vs-elasticsearch-performance)에서 해시태그 검색 성능 비교를 통해 Elasticsearch 도입을 결정했다.

이번에는 **MySQL을 원본 데이터 저장소로 유지하면서 Elasticsearch와 자동으로 동기화**하는 파이프라인을 Logstash로 구축한 과정을 정리한다.

---

## 아키텍처

```
[Spring Boot 앱]
      │
      ▼
  MySQL (posts)       ←  원본 데이터
      │
      │  JDBC Input (30초 폴링)
      ▼
  Logstash
      │
      │  Elasticsearch Output
      ▼
  Elasticsearch       ←  검색 전용
```

Logstash가 주기적으로 MySQL을 폴링해 변경된 행을 ES에 업서트(upsert)하고, soft delete된 행을 ES에서 제거한다.

---

## 왜 Logstash인가

기존에는 Spring 애플리케이션에서 직접 ES에 쓰는 방식이었다.

```
MySQL write → 이벤트 발행 → 비동기 ES indexPost() 호출
MySQL delete → 이벤트 발행 → 비동기 ES deletePost() 호출
```

이 방식의 문제점:
- MySQL 커밋 성공 후 ES 인덱싱 실패 시 **데이터 불일치** 발생
- 앱 장애 또는 재시작 시 이벤트 유실 가능
- 검색 관심사가 비즈니스 로직에 침투

Logstash를 사용하면:
- **MySQL이 단일 진실의 원천(Single Source of Truth)**
- 앱은 MySQL만 신경 쓰면 됨
- Logstash가 재시작해도 `last_run` 기준으로 이어서 동기화

---

## Soft Delete 도입

Logstash는 MySQL의 hard delete를 감지하지 못한다.
행이 삭제되면 Logstash 폴링 쿼리에 걸리지 않기 때문이다.

해결책은 `deleted_at` 컬럼을 추가하는 **Soft Delete** 패턴이다.

```sql
ALTER TABLE posts ADD COLUMN deleted_at DATETIME DEFAULT NULL;
```

삭제 요청이 오면 `DELETE` 대신 `deleted_at = NOW()`로 업데이트한다.
Logstash가 `deleted_at IS NOT NULL`인 행을 감지해 ES에서 해당 문서를 삭제한다.

---

## 코드 변경

### 1. Post 엔티티

<details markdown="1">
<summary>Before</summary>

```java
@Entity
@Table(name = "posts")
public class Post extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(columnDefinition = "TEXT")
    private String content;

    @ElementCollection
    @CollectionTable(name = "post_hashtags", joinColumns = @JoinColumn(name = "post_id"))
    @Column(name = "hashtag")
    private List<String> hashtags = new ArrayList<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("orderNum ASC")
    private List<PostImage> images = new ArrayList<>();

    // ...
}
```

</details>

<details markdown="1">
<summary>After</summary>

```java
@Entity
@Table(name = "posts")
public class Post extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(columnDefinition = "TEXT")
    private String content;

    @ElementCollection
    @CollectionTable(name = "post_hashtags", joinColumns = @JoinColumn(name = "post_id"))
    @Column(name = "hashtag")
    private List<String> hashtags = new ArrayList<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("orderNum ASC")
    private List<PostImage> images = new ArrayList<>();

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;   // ← 추가

    public void softDelete() {
        this.deletedAt = LocalDateTime.now();
    }

    public boolean isDeleted() {
        return this.deletedAt != null;
    }
}
```

</details>

---

### 2. PostRepository — 삭제된 게시물 필터링

<details markdown="1">
<summary>Before</summary>

```java
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("SELECT p FROM Post p JOIN FETCH p.user ORDER BY p.createdDate DESC")
    Page<Post> findAllWithUser(Pageable pageable);

    @Query("SELECT p FROM Post p JOIN FETCH p.user WHERE p.id = :postId")
    Optional<Post> findByIdWithUser(@Param("postId") Long postId);

    @Query("SELECT p FROM Post p JOIN FETCH p.user WHERE p.id IN :ids ORDER BY p.createdDate DESC")
    List<Post> findByIdInWithUser(@Param("ids") List<Long> ids);
}
```

</details>

<details markdown="1">
<summary>After</summary>

```java
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("SELECT p FROM Post p JOIN FETCH p.user WHERE p.deletedAt IS NULL ORDER BY p.createdDate DESC")
    Page<Post> findAllWithUser(Pageable pageable);

    @Query("SELECT p FROM Post p JOIN FETCH p.user WHERE p.id = :postId AND p.deletedAt IS NULL")
    Optional<Post> findByIdWithUser(@Param("postId") Long postId);

    @Query("SELECT p FROM Post p JOIN FETCH p.user WHERE p.id IN :ids AND p.deletedAt IS NULL ORDER BY p.createdDate DESC")
    List<Post> findByIdInWithUser(@Param("ids") List<Long> ids);
}
```

</details>

---

### 3. CommunityPostWriteServiceImpl — ES 이벤트 제거, soft delete 적용

<details markdown="1">
<summary>Before</summary>

```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommunityPostWriteServiceImpl implements CommunityPostWriteService {

    private final PostRepository postRepository;
    private final SecurityService securityService;
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;  // ← ES 이벤트 발행

    @Override
    public Long createPost(PostCreateServiceRequest request) {
        // ...
        postRepository.save(post);
        addImages(post, request.getImageKeys());

        // ES 인덱싱 이벤트 발행 (트랜잭션 커밋 후 비동기 실행)
        eventPublisher.publishEvent(new PostIndexEvent(post, request.getHashtags()));
        return post.getId();
    }

    @Override
    public void updatePost(Long postId, PostUpdateServiceRequest request) {
        // ...
        post.updateContent(request.getContent());
        post.updateHashtags(request.getHashtags());
        // ...

        // ES 재인덱싱
        eventPublisher.publishEvent(new PostIndexEvent(post, request.getHashtags()));
    }

    @Override
    public void deletePost(Long postId) {
        // ...
        postRepository.delete(post);  // hard delete

        // ES 삭제 이벤트 발행
        eventPublisher.publishEvent(new PostDeleteEvent(postId));
    }
}
```

</details>

<details markdown="1">
<summary>After</summary>

```java
@Service
@RequiredArgsConstructor
@Transactional
public class CommunityPostWriteServiceImpl implements CommunityPostWriteService {

    private final PostRepository postRepository;
    private final SecurityService securityService;
    private final UserRepository userRepository;
    // ApplicationEventPublisher 제거 — ES 동기화는 Logstash가 담당

    @Override
    public Long createPost(PostCreateServiceRequest request) {
        // ...
        postRepository.save(post);
        addImages(post, request.getImageKeys());
        // Logstash가 updated_date 기준으로 폴링해 ES에 자동 인덱싱
        return post.getId();
    }

    @Override
    public void updatePost(Long postId, PostUpdateServiceRequest request) {
        // ...
        post.updateContent(request.getContent());
        post.updateHashtags(request.getHashtags());
        // ...
        // Logstash가 updated_date 변경을 감지해 ES 자동 업데이트
    }

    @Override
    public void deletePost(Long postId) {
        // ...
        // Logstash가 deleted_at 컬럼을 감지해 ES에서 삭제
        post.softDelete();
    }
}
```

</details>

---

### 4. PostSearchService — indexPost / deletePost 제거

<details markdown="1">
<summary>Before</summary>

```java
@Service
@Profile("!test")
@RequiredArgsConstructor
public class PostSearchService {

    private final ElasticsearchOperations elasticsearchOperations;
    private final PostSearchRepository postSearchRepository;

    // ES 직접 인덱싱
    public void indexPost(Post post, List<String> hashtags) {
        PostDocument document = PostDocument.builder()
                .postId(post.getId())
                .userId(post.getUser().getId())
                .userNickname(post.getUser().getNickName())
                .content(post.getContent())
                .hashtags(hashtags)
                .createdDate(post.getCreatedDate())
                .build();
        postSearchRepository.save(document);
    }

    // ES 직접 삭제
    public void deletePost(Long postId) {
        postSearchRepository.deleteById(String.valueOf(postId));
    }

    public List<String> searchHashtags(String keyword) { ... }
    public List<Long> search(String keyword, int page, int size) { ... }
}
```

</details>

<details markdown="1">
<summary>After</summary>

```java
@Service
@Profile("!test")
@RequiredArgsConstructor
public class PostSearchService {

    private final ElasticsearchOperations elasticsearchOperations;
    private final PostSearchRepository postSearchRepository;

    // indexPost(), deletePost() 제거 — Logstash가 담당

    public List<String> searchHashtags(String keyword) {
        NativeQuery query = NativeQuery.builder()
                .withQuery(q -> q
                        .wildcard(w -> w.field("hashtags").wildcard("*" + keyword + "*"))
                )
                .withPageable(PageRequest.of(0, 200))
                .build();

        SearchHits<PostDocument> hits = elasticsearchOperations.search(query, PostDocument.class);
        return hits.stream()
                .flatMap(hit -> hit.getContent().getHashtags().stream())
                .filter(tag -> tag.contains(keyword))
                .distinct()
                .sorted()
                .collect(Collectors.toList());
    }

    public List<Long> search(String keyword, int page, int size) {
        NativeQuery query = NativeQuery.builder()
                .withQuery(q -> q
                        .term(t -> t.field("hashtags").value(keyword))
                )
                .withPageable(PageRequest.of(page, size))
                .withSort(Sort.by(Sort.Direction.DESC, "createdDate"))
                .build();

        SearchHits<PostDocument> hits = elasticsearchOperations.search(query, PostDocument.class);
        return hits.stream()
                .map(hit -> hit.getContent().getPostId())
                .collect(Collectors.toList());
    }
}
```

</details>

---

### 5. 이벤트 클래스 삭제

Logstash 도입으로 더 이상 필요하지 않아 전부 제거했다.

| 파일 | 조치 |
|---|---|
| `PostIndexEvent.java` | 삭제 |
| `PostDeleteEvent.java` | 삭제 |
| `PostEventListener.java` | 삭제 |

---

## Logstash 구성

### 서버 디렉토리 구조

```
/home/tkdrl8908/logstash/
├── docker-compose.yml
├── .env
├── config/
│   └── logstash.yml
├── pipeline/
│   └── posts.conf
└── drivers/
    └── mysql-connector-j.jar
```

### docker-compose.yml

```yaml
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    container_name: puppynote-logstash
    restart: unless-stopped
    volumes:
      - ./pipeline:/usr/share/logstash/pipeline
      - ./config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./drivers/mysql-connector-j.jar:/usr/share/logstash/drivers/mysql-connector-j.jar
      - logstash_data:/usr/share/logstash/data
    env_file:
      - .env
    networks:
      - puppynote-net

volumes:
  logstash_data:

networks:
  puppynote-net:
    external: true
```

`logstash_data` 볼륨은 `last_run` 메타데이터를 영속화해 컨테이너 재시작 후에도 중단 지점부터 동기화를 이어간다.

### pipeline/posts.conf

<details markdown="1">
<summary>posts.conf 전체 펼치기</summary>

```ruby
input {
  jdbc {
    jdbc_driver_library    => "/usr/share/logstash/drivers/mysql-connector-j.jar"
    jdbc_driver_class      => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://${DB_HOST}:3306/puppynote?useSSL=false&serverTimezone=Asia/Seoul"
    jdbc_user              => "${DB_USERNAME}"
    jdbc_password          => "${DB_PASSWORD}"

    # updated_date 기준 증분 동기화
    use_column_value       => true
    tracking_column        => "updated_date"
    tracking_column_type   => "timestamp"
    last_run_metadata_path => "/usr/share/logstash/data/.last_run_posts"

    # 30초마다 폴링
    schedule => "*/30 * * * * *"

    statement => "
      SELECT
        p.id            AS post_id,
        p.user_id,
        p.content,
        p.created_date,
        p.updated_date,
        p.deleted_at,
        GROUP_CONCAT(ph.hashtag ORDER BY ph.hashtag SEPARATOR ',') AS hashtags
      FROM posts p
      LEFT JOIN post_hashtags ph ON p.id = ph.post_id
      WHERE p.updated_date > :sql_last_value
      GROUP BY p.id, p.user_id, p.content, p.created_date, p.updated_date, p.deleted_at
      ORDER BY p.updated_date ASC
    "
  }
}

filter {
  # "강아지,고양이" → ["강아지", "고양이"] 배열로 변환
  if [hashtags] {
    mutate {
      split => { "hashtags" => "," }
    }
  } else {
    mutate {
      add_field => { "hashtags" => [] }
    }
  }

  mutate {
    add_field    => { "[@metadata][post_id]" => "%{post_id}" }
    remove_field => ["@version", "@timestamp"]
  }
}

output {
  if [deleted_at] and [deleted_at] != "" {
    # soft delete 된 게시물 → ES 삭제
    elasticsearch {
      hosts       => ["http://${ES_HOST}:9200"]
      user        => "${ES_USERNAME}"
      password    => "${ES_PASSWORD}"
      index       => "posts"
      action      => "delete"
      document_id => "%{[@metadata][post_id]}"
    }
  } else {
    # 신규/수정 게시물 → 업서트
    elasticsearch {
      hosts         => ["http://${ES_HOST}:9200"]
      user          => "${ES_USERNAME}"
      password      => "${ES_PASSWORD}"
      index         => "posts"
      action        => "update"
      document_id   => "%{[@metadata][post_id]}"
      doc_as_upsert => true
    }
  }
}
```

</details>

---

## 동기화 흐름 상세

### 신규/수정 게시물

```
1. 앱이 MySQL에 INSERT or UPDATE
   → BaseTimeEntity가 updated_date 자동 갱신

2. Logstash 폴링 쿼리
   WHERE updated_date > last_run_value 로 변경 행 감지

3. filter 블록
   GROUP_CONCAT 결과를 배열로 변환

4. output 블록
   doc_as_upsert: true → 없으면 insert, 있으면 update
```

### 삭제 게시물

```
1. 앱이 post.softDelete() 호출
   → deleted_at = NOW(), updated_date 자동 갱신

2. Logstash 폴링 쿼리
   WHERE updated_date > last_run_value 로 해당 행 감지
   deleted_at 값이 존재함

3. output 블록
   action: "delete" → ES에서 해당 document 제거
```

---

## 배포

**1. MySQL Connector 다운로드**

```bash
mkdir -p /home/tkdrl8908/logstash/drivers
wget -O /home/tkdrl8908/logstash/drivers/mysql-connector-j.jar \
  https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.3.0/mysql-connector-j-8.3.0.jar
```

**2. .env 작성**

```bash
cd /home/tkdrl8908/logstash
cp .env.example .env
vi .env
```

```
DB_HOST=MySQL서버IP
DB_USERNAME=root
DB_PASSWORD=비밀번호
ES_HOST=ES서버IP
ES_USERNAME=elastic
ES_PASSWORD=비밀번호
```

**3. 실행**

```bash
docker-compose up -d
docker-compose logs -f
```

정상 동작 시 로그:

```
[INFO] JDBC - Starting JDBC input...
[INFO] Successfully polled N rows
[INFO] Sending N events to Elasticsearch
```

---

## 결과 — 전후 비교

| 항목 | Before (앱 직접 ES write) | After (Logstash) |
|---|---|---|
| ES 인덱싱 주체 | Spring 앱 | Logstash |
| 데이터 불일치 위험 | 있음 (이벤트 유실 가능) | 없음 (MySQL이 단일 원본) |
| 앱 코드 관심사 | MySQL + ES | MySQL만 |
| 장애 복구 | 이벤트 유실 시 수동 재인덱싱 필요 | Logstash 재시작 시 last_run부터 자동 복구 |
| 삭제 처리 | ES delete 직접 호출 | soft delete → Logstash 감지 |
| 동기화 지연 | 즉시 (비동기) | 최대 30초 |

동기화 지연이 최대 30초 발생하지만, 검색 데이터의 특성상 수십 초의 지연은 허용 가능하다.
데이터 일관성과 코드 단순성을 확보한 것이 더 큰 이점이다.
