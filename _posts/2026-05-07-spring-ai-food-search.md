---
title: Spring AI + Ollama로 반려동물 음식 검색 기능 구현하기
author: SangkiHan
date: 2026-05-07 11:00:00 +0900
categories: [Spring, AI]
tags: [Spring, Spring AI, Ollama, Elasticsearch, Logstash]
---

## 개요

PuppyNote 프로젝트에 반려동물 음식 안전 정보를 제공하는 AI 검색 기능을 추가했다.

"당근은 강아지에게 먹여도 되나요?" 같은 질문을 입력하면 AI가 안전 여부와 영양 정보를 답변해주는 기능이다.

AI 호출은 비용(응답 시간)이 크기 때문에 **Elasticsearch를 1차 캐시로** 사용해 동일한 질문이 반복될 경우 AI를 거치지 않고 바로 응답하는 구조로 설계했다.

AI 프레임워크는 **Spring AI 1.0.0**을 선택했다. OpenAI, Ollama, Claude 등 다양한 모델을 동일한 인터페이스(`ChatClient`)로 교체할 수 있다는 점이 매력적이었다. 로컬 LLM 서버인 Ollama와 연동해 한국어에 특화된 **EXAONE 3.5 7.8B** 모델을 사용했다.

---

## 전체 프로세스 흐름

```
클라이언트
  │
  ├─ 1. GET /api/v1/foods?question=당근  ──▶ Elasticsearch 검색
  │                                           │
  │                                    결과 있음 ──▶ RDB에서 상세 조회 ──▶ 응답
  │                                           │
  │                                    결과 없음 ──▶ 빈 배열 반환
  │
  ├─ (클라이언트가 사용자에게 AI 조회 여부 확인)
  │
  └─ 2. POST /api/v1/foods/ai  ──▶ RDB 중복 확인
                                      │
                               이미 있음 ──▶ 기존 답변 반환
                                      │
                               없음 ──▶ Ollama AI 호출
                                           │
                                      음식 아님 ──▶ 400 에러
                                           │
                                      음식 맞음 ──▶ RDB 저장 ──▶ 응답
                                                        │
                                                   Logstash가 30초마다
                                                   ES에 자동 동기화
```

ES에는 검색에 필요한 최소 필드(`foodChatHistoryId`, `question`, `safetyLevel`)만 저장하고, 상세 데이터(`answer`)는 RDB에서 가져온다. 커뮤니티 게시판과 동일한 패턴이다.

---

## Spring AI 설정

### build.gradle

```groovy
dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:1.0.0"
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
    maven { url 'https://repo.spring.io/release' }
}

dependencies {
    // Spring AI - Ollama
    implementation 'org.springframework.ai:spring-ai-starter-model-ollama'
}
```

Spring AI 1.0.0 GA에서 아티팩트 이름이 바뀌었다. 이전 버전에서 쓰던 `spring-ai-ollama-spring-boot-starter`는 동작하지 않는다.

| 버전 | 아티팩트명 |
|------|-----------|
| 1.0.0 이전 | `spring-ai-ollama-spring-boot-starter` |
| 1.0.0 GA | `spring-ai-starter-model-ollama` |

### application.yml

```yaml
spring:
  ai:
    ollama:
      base-url: ${OLLAMA_HOST:http://localhost:11434}
      chat:
        model: exaone3.5:7.8b
        options:
          temperature: 0.3
```

`temperature`는 응답의 창의성을 조절한다. 0에 가까울수록 일관되고 예측 가능한 답변을 생성한다. 음식 안전 정보처럼 사실에 기반한 답변이 필요한 경우 낮은 값(0.3)이 적합하다.

---

## Elasticsearch 문서 구조

```java
@Getter
@NoArgsConstructor
@Document(indexName = "food_chat")
@Setting(settingPath = "es-settings.json")
public class FoodDocument {

    @Id
    private String id;

    @Field(type = FieldType.Long)
    private Long foodChatHistoryId;

    @Field(type = FieldType.Text, analyzer = "nori")
    private String question;

    @Field(type = FieldType.Keyword)
    private String safetyLevel;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdDate;
}
```

`question` 필드에 **nori 분석기**를 적용했다. nori는 한국어 형태소 분석기로 "당근은 강아지에게 줘도 되나요?"를 "당근", "강아지", "주다" 등의 형태소로 분리해 검색 정확도를 높여준다.

---

## Ollama 서비스 구현 - Structured Output

초기에는 AI 응답에서 `[GOOD]`, `[NOTION]`, `[BAD]` 태그를 정규식으로 파싱하는 방식을 사용했다.

```java
// 초기 구현 - 정규식 파싱
Matcher matcher = SAFETY_PATTERN.matcher(content);
if (matcher.find()) {
    SafetyLevel safetyLevel = SafetyLevel.valueOf(matcher.group(1));
    String[] parts = content.split("\\R", 2);
    String answer = (parts.length > 1 ? parts[1] : "").trim();
    ...
}
```

문제는 AI가 프롬프트 지시를 완전히 따르지 않는 경우가 있다는 것이다. "강아지에게 안전하다면: [GOOD]" 처럼 태그 앞에 설명을 붙이거나, 포맷을 살짝 바꾸면 파싱이 깨진다.

Spring AI의 **Structured Output** 기능을 사용하면 이 문제를 근본적으로 해결할 수 있다. AI 응답을 지정한 Java 타입으로 직접 변환해준다.

```java
@Slf4j
@Service
public class OllamaService {

    private static final String SYSTEM_PROMPT =
            "당신은 반려동물 영양 전문가입니다.\n" +
                    "사용자의 질문이 강아지 또는 반려동물에게 먹이는 음식에 관한 질문인지 판단하고 JSON으로 응답하세요.\n\n" +
                    "isFood: 음식 관련 질문이면 true, 아니면 false\n" +
                    "safetyLevel: GOOD(안전) / NOTION(주의) / BAD(위험) / null (음식 아닐 때)\n" +
                    "answer: 음식 관련 질문이면 영양 정보, 주의사항, 적절한 섭취량을 한국어로 상세히 작성, 아니면 null";

    private final ChatClient chatClient;

    public OllamaService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
                .defaultSystem(SYSTEM_PROMPT)
                .build();
    }

    public OllamaResult ask(String question) {
        try {
            FoodAnalysis analysis = chatClient.prompt()
                    .user(question)
                    .call()
                    .entity(FoodAnalysis.class);  // Structured Output

            if (analysis == null || !analysis.isFood()) {
                return OllamaResult.notFood();
            }

            SafetyLevel safetyLevel = analysis.safetyLevel() != null
                    ? SafetyLevel.valueOf(analysis.safetyLevel())
                    : null;

            return OllamaResult.food(analysis.answer(), safetyLevel);
        } catch (Exception e) {
            log.error("Ollama 호출 실패: {}", e.getMessage());
            throw new RuntimeException("AI 서비스 호출에 실패했습니다.");
        }
    }

    record FoodAnalysis(boolean isFood, String safetyLevel, String answer) {}

    public record OllamaResult(boolean isFood, String answer, SafetyLevel safetyLevel) {
        public static OllamaResult food(String answer, SafetyLevel safetyLevel) {
            return new OllamaResult(true, answer, safetyLevel);
        }
        public static OllamaResult notFood() {
            return new OllamaResult(false, null, null);
        }
    }
}
```

### Structured Output 내부 동작

`entity(FoodAnalysis.class)`를 호출하면 Spring AI가 `FoodAnalysis` 클래스의 JSON 스키마를 자동으로 생성해 프롬프트에 추가한다.

```
Your response should be in JSON format.
Do not include any explanations, only provide a RFC8259 compliant JSON response.
The JSON schema for the response:
{
  "isFood": boolean,
  "safetyLevel": string,
  "answer": string
}
```

AI가 JSON을 반환하면 Spring AI가 `FoodAnalysis` 객체로 역직렬화한다. 정규식도, 첫 줄 파싱도 필요 없다.

| | 정규식 파싱 | Structured Output |
|--|------------|-------------------|
| AI 포맷 이탈 시 | 파싱 실패, null 반환 | 역직렬화 실패 예외 |
| 파싱 코드 | 정규식 + split | 없음 |
| 필드 추가 | 프롬프트 + 파싱 코드 수정 | record 필드 추가만 |

---

## Elasticsearch 검색 구현

```java
@Slf4j
@Service
@Profile("!test")
@RequiredArgsConstructor
public class FoodSearchService {

    private final ElasticsearchOperations elasticsearchOperations;
    private final FoodDocumentRepository foodDocumentRepository;

    public List<Long> search(String question, int page, int size) {
        try {
            NativeQuery query = NativeQuery.builder()
                    .withQuery(q -> q
                            .match(m -> m
                                    .field("question")
                                    .query(question)
                                    .minimumShouldMatch("80%")
                            )
                    )
                    .withMinScore(1.5f)
                    .withPageable(PageRequest.of(page, size))
                    .build();

            SearchHits<FoodDocument> hits = elasticsearchOperations.search(query, FoodDocument.class);
            return hits.stream()
                    .map(hit -> hit.getContent().getFoodChatHistoryId())
                    .collect(Collectors.toList());
        } catch (Exception e) {
            log.warn("ES 조회 실패, AI 호출로 전환합니다: {}", e.getMessage());
            return List.of();
        }
    }

    public void delete(String id) {
        foodDocumentRepository.deleteById(id);
    }
}
```

검색 조건:
- `minimumShouldMatch("80%")`: 형태소 중 80% 이상 일치해야 결과로 반환
- `withMinScore(1.5f)`: 관련성 점수가 1.5 미만인 결과 제외

ES 장애(503 등)가 발생해도 예외를 잡아 빈 리스트를 반환하도록 처리했다. ES가 다운되더라도 AI 호출로 자연스럽게 폴백된다.

`@Profile("!test")`를 적용해 테스트 환경에서는 이 빈이 등록되지 않는다.

---

## 서비스 로직

```java
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class FoodChatServiceImpl implements FoodChatService {

    private final FoodChatHistoryRepository foodChatHistoryRepository;
    private final OllamaService ollamaService;
    private final FoodSearchService foodSearchService;

    @Override
    public FoodListResponse search(FoodChatServiceRequest request) {
        // question 없으면 등록순 전체 조회
        if (request.question() == null || request.question().isBlank()) {
            List<FoodResponse> responses = foodChatHistoryRepository
                    .findAllOrderByCreatedDateDesc(request.page(), request.size())
                    .stream()
                    .map(FoodResponse::of)
                    .toList();
            return FoodListResponse.of(responses, request.page(), foodChatHistoryRepository.count());
        }

        List<Long> ids = foodSearchService.search(request.question(), request.page(), request.size());
        if (ids.isEmpty()) {
            return FoodListResponse.of(List.of(), request.page(), 0L);
        }
        log.info("ES 캐시에서 음식 정보 반환: {}", request.question());
        List<FoodResponse> responses = foodChatHistoryRepository.findAllByIdIn(ids)
                .stream()
                .map(FoodResponse::of)
                .toList();
        return FoodListResponse.of(responses, request.page(), responses.size());
    }

    @Override
    public FoodResponse ask(FoodAiServiceRequest request) {
        String question = request.question();

        // DB 중복 질문 방지
        Optional<FoodChatHistory> existing = foodChatHistoryRepository.findByQuestion(question);
        if (existing.isPresent()) {
            log.info("DB에 동일 질문 존재, 기존 답변 반환: {}", question);
            return FoodResponse.of(existing.get());
        }

        OllamaService.OllamaResult result = ollamaService.ask(question);

        if (!result.isFood()) {
            throw new PuppyNoteException("음식에 관한 질문만 해주세요.");
        }

        FoodChatHistory saved = foodChatHistoryRepository.save(
                FoodChatHistory.of(question, result.answer(), result.safetyLevel())
        );

        return FoodResponse.of(saved);
    }
}
```

`question` 컬럼이 `TEXT` 타입이라 MySQL unique constraint를 걸 수 없어 애플리케이션 레벨에서 중복을 방지했다.

---

## API 구조

```java
@Validated
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/foods")
public class FoodChatController {

    private final FoodChatService foodChatService;
    private final FoodSearchService foodSearchService;

    @GetMapping
    public ApiResponse<FoodListResponse> search(
            @RequestParam(required = false) String question,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size
    ) {
        return ApiResponse.ok(foodChatService.search(new FoodChatServiceRequest(question, page, size)));
    }

    @PostMapping("/ai")
    public ApiResponse<FoodResponse> ask(@RequestBody @Valid FoodChatRequest request) {
        return ApiResponse.ok(foodChatService.ask(request.toServiceRequest()));
    }

    @DeleteMapping("/documents/{id}")
    public ApiResponse<Void> deleteDocument(@PathVariable String id) {
        foodSearchService.delete(id);
        return ApiResponse.ok(null);
    }
}
```

| 메서드 | 엔드포인트 | 설명 |
|--------|-----------|------|
| GET | `/api/v1/foods` | question 없으면 전체, 있으면 ES 검색 |
| POST | `/api/v1/foods/ai` | AI 호출 후 DB 저장 |
| DELETE | `/api/v1/foods/documents/{id}` | ES 문서 직접 삭제 |

DELETE API는 ES 문서만 삭제하고 RDB는 그대로 유지한다. 삭제 후 다시 AI 조회를 하면 Logstash가 재동기화한다.

---

## Logstash 파이프라인

ES 쓰기는 애플리케이션에서 직접 하지 않고 Logstash가 30초마다 RDB를 폴링해 동기화한다.

```ruby
input {
  jdbc {
    jdbc_driver_library    => "/usr/share/logstash/drivers/mysql-connector-j.jar"
    jdbc_driver_class      => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://${DB_HOST}:3306/puppynote?useSSL=false&serverTimezone=Asia/Seoul"
    jdbc_user              => "${DB_USERNAME}"
    jdbc_password          => "${DB_PASSWORD}"
    use_column_value       => true
    tracking_column        => "updated_date"
    tracking_column_type   => "timestamp"
    last_run_metadata_path => "/usr/share/logstash/data/.last_run_food_chat"
    schedule               => "*/30 * * * * *"
    statement              => "
      SELECT
        id,
        question,
        safety_level,
        created_date,
        updated_date
      FROM food_chat_histories
      WHERE updated_date > :sql_last_value
      ORDER BY updated_date ASC
    "
  }
}

filter {
  mutate {
    rename => {
      "id"           => "foodChatHistoryId"
      "safety_level" => "safetyLevel"
      "created_date" => "createdDate"
      "updated_date" => "updatedDate"
    }
  }

  ruby {
    code => "
      val = event.get('createdDate')
      if val
        event.set('createdDate', val.to_s[0, 19])
      end
    "
  }

  mutate {
    add_field    => { "[@metadata][doc_id]" => "%{foodChatHistoryId}" }
    remove_field => ["@version", "@timestamp", "updatedDate"]
  }
}

output {
  elasticsearch {
    hosts         => ["http://${ES_HOST}:9200"]
    user          => "${ES_USERNAME}"
    password      => "${ES_PASSWORD}"
    index         => "food_chat"
    action        => "update"
    document_id   => "%{[@metadata][doc_id]}"
    doc_as_upsert => true
  }
}
```

`answer` 필드는 ES에 저장하지 않는다. ES는 검색용 ID 인덱스 역할만 하고, 상세 데이터는 RDB에서 조회한다.

`food_chat_histories`는 소프트 딜리트가 없어 delete 분기 없이 upsert만 처리한다.

---

## 트러블슈팅

**1. Spring AI 1.0.0 아티팩트명 변경**

`spring-ai-ollama-spring-boot-starter`로 의존성을 추가했더니 `ChatClient.Builder` 빈을 찾지 못하는 오류가 발생했다. 1.0.0 GA에서 아티팩트명이 `spring-ai-starter-model-ollama`로 변경된 것을 확인했다.

**2. ES 503 장애 시 500 에러 전파**

초기 구현에서 ES가 다운되면 503이 그대로 전파되어 500 에러가 발생했다. `FoodSearchService.search()`에서 모든 예외를 catch해 빈 리스트를 반환하도록 수정했다. ES 장애 시에도 AI 호출로 자연스럽게 폴백된다.

**3. AI 응답 파싱 불안정 → Structured Output으로 해결**

정규식으로 `[GOOD]` 태그를 파싱할 때 AI가 "강아지에게 안전하다면: [GOOD]" 처럼 앞에 설명을 붙이면 태그만 제거해도 설명 텍스트가 답변에 남는 문제가 있었다. `split("\\R", 2)`로 첫 줄 전체를 제거하는 방식으로 1차 해결했지만, AI가 포맷을 지키지 않는 근본적인 문제는 남아있었다.

Spring AI의 Structured Output(`entity(FoodAnalysis.class)`)으로 전환해 JSON 스키마를 강제하도록 변경했다. 이후 파싱 관련 문제가 사라졌다.
