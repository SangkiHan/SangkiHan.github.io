---
title: 서버 에러 로그 자동 분석 에이전트 구축기 — RAG부터 Agentic까지
author: SangkiHan
date: 2026-06-05 20:00:00 +0900
categories: [AI, RAG]
tags: [RAG, ChromaDB, Ollama, Agentic, FastAPI, LLM]
---

## 개요

서버에서 에러가 발생했을 때 자동으로 원인을 분석하고 수정 코드까지 제안해주는 에이전트를 만들었습니다.

Spring Boot 서버에서 예외가 발생하면 Webhook으로 알림을 받고, 관련 소스 코드를 찾아 LLM이 분석한 결과를 Slack으로 전송합니다.

이 글에서는 처음 Pipeline RAG를 구축하고, 문제를 발견하고, Agentic RAG로 개선하는 전 과정을 정리합니다.

---

## 전체 시스템 아키텍처

```
[Spring Boot 서버]
  에러 발생 (NullPointerException 등)
       │
       │ POST /api/v1/webhook/error
       ▼
[log-agent-backend (FastAPI)]
       │
       ├─ 1. Git Clone (GitHub repo)
       ├─ 2. ChromaDB 인덱싱
       ├─ 3. LLM 분석 (Ollama)
       └─ 4. Slack 알림
```

---

## 구성 요소

| 컴포넌트 | 역할 |
|---|---|
| FastAPI | Webhook 수신, 파이프라인 오케스트레이션 |
| Ollama | LLM 추론 (`gemma4:12b`), 임베딩 (`nomic-embed-text`) |
| ChromaDB | 벡터 DB — 소스 코드 임베딩 저장/검색 |
| dulwich | Python Git 클라이언트 — repo 클론/패치 |
| Slack Webhook | 분석 결과 알림 전송 |

---

## Step 1. Docker Compose 설치

```yaml
services:
  log-agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OLLAMA_HOST=http://210.183.11.113:11434
      - CHROMA_HOST=chromadb
      - CHROMA_PORT=8000
    depends_on:
      - chromadb

  chromadb:
    image: chromadb/chroma:latest
    volumes:
      - chroma_data:/data
    ports:
      - "8001:8000"

volumes:
  chroma_data:
```

Ollama는 별도 서버(GPU 장착)에서 실행하고 호스트 IP로 연결합니다.

---

## Step 2. Spring Boot 연동

서버에서 에러 발생 시 Webhook을 전송하는 서비스입니다.

```java
@Slf4j
@Service
public class LogAgentWebhookService {

    public void sendError(Exception e, HttpServletRequest request) {
        CompletableFuture.runAsync(() -> {
            ErrorEventPayload payload = ErrorEventPayload.builder()
                    .serverName(serverName)
                    .serverIp(serverIp)
                    .errorType(e.getClass().getSimpleName())
                    .message(e.getMessage())
                    .stackTrace(getStackTrace(e))   // 전체 stack trace 포함
                    .requestMethod(request.getMethod())
                    .requestUrl(request.getRequestURI())
                    .build();

            restClient.post()
                    .uri(webhookUrl)
                    .body(payload)
                    .retrieve()
                    .toBodilessEntity();
        });
    }

    private String getStackTrace(Exception e) {
        StringWriter sw = new StringWriter();
        e.printStackTrace(new PrintWriter(sw));
        return sw.toString();
    }
}
```

JSON 직렬화 시 snake_case로 맞추기 위해 `@JsonProperty`를 사용합니다.

```java
@Getter
@Builder
public class ErrorEventPayload {
    @JsonProperty("server_name")  private String serverName;
    @JsonProperty("server_ip")    private String serverIp;
    @JsonProperty("error_type")   private String errorType;
    @JsonProperty("message")      private String message;
    @JsonProperty("stack_trace")  private String stackTrace;
    @JsonProperty("request_method") private String requestMethod;
    @JsonProperty("request_url")  private String requestUrl;
    @JsonProperty("response_status") private int responseStatus;
}
```

---

## Step 3. RAG 인덱싱 구조

RAG(Retrieval-Augmented Generation)는 LLM이 소스 코드를 "기억"하게 하는 기법입니다.

**핵심 원리**: 텍스트를 고차원 숫자 벡터로 변환하면, 의미가 비슷한 텍스트는 벡터 공간에서 가까운 위치에 놓입니다.

```
"NullPointerException in OllamaService"
        │ nomic-embed-text
        ▼
[0.12, -0.34, 0.87, 0.05, ...]  ← 768차원 벡터

"OllamaService.java 코드 내용..."
        │ nomic-embed-text
        ▼
[0.11, -0.32, 0.85, 0.06, ...]  ← 비슷한 벡터 → 유사도 높음
```

### 인덱싱 흐름

```
GitHub repo
    │
    │ dulwich clone (depth=1)
    ▼
로컬 파일 시스템
    │
    │ .java .kt .py .ts 파일 추출
    │ 1500자 단위 청크 분할
    ▼
Ollama /api/embed (nomic-embed-text)
    │
    │ 벡터 변환 (배치 50개씩)
    ▼
ChromaDB upsert
    │ id: "파일경로__chunk0"
    │ document: "코드 내용"
    │ embedding: [0.12, -0.34, ...]
    │ metadata: {path: "src/...", commit: "a8bd616c"}
    ▼
인덱싱 완료 (commit hash로 변경 여부 관리)
```

commit hash가 바뀌지 않으면 재인덱싱을 건너뜁니다.

---

## 추론 과정 진화

### v1 — Pipeline RAG

처음 구현한 방식입니다. Python 코드가 미리 관련 파일 5개를 골라서 LLM에 전달합니다.

```
에러 수신
    │
    ├─ stack trace 파싱 → 파일 경로 추출
    ├─ RAG 검색 → 유사 파일 top-5
    │
    │  (두 결과 합치기)
    ▼
LLM에 파일 5개 + 에러 로그 전달
    │
    ▼
분석 결과 JSON 반환
```

**문제점**: RAG 검색이 `NullPointerException`처럼 흔한 에러 메시지로 검색하면 엉뚱한 파일이 나옵니다. 에러가 난 파일이 아니라 의미적으로 비슷한 단어가 많은 파일을 가져오기 때문입니다.

```
검색어: "NullPointerException: Cannot invoke String.toUpperCase()"

RAG 결과:
  - OllamaService.java        ← 관련 없음
  - WeatherServiceImpl.java   ← 관련 없음
  - UserControllerTest.java   ← 관련 없음

실제 에러 파일:
  - TestErrorController.java  ← 없음!
```

그럼에도 LLM이 정답을 맞추는 경우가 있었는데, stack trace에 클래스명(`TestErrorController`)이 이미 들어있어서 RAG 없이도 추론 가능했기 때문입니다. RAG가 실제로 기여하지 않은 셈입니다.

---

### v2 — Agentic RAG

LLM이 직접 무엇을 검색할지 판단하게 합니다. Ollama `/api/chat` + tool calling을 활용합니다.

```
에러 수신
    │
    ▼
LLM에 에러 로그만 전달
    │
    ├─ LLM: "TestErrorController 검색해줘"
    │        └─ search_files("TestErrorController")
    │              └─ ChromaDB 검색 → 결과 반환
    │
    ├─ LLM: "이 파일 내용 읽어줘"
    │        └─ read_file("src/.../TestErrorController.java")
    │              └─ 파일 내용 반환
    │
    ├─ LLM: "주입된 서비스도 찾아볼게"
    │        └─ search_files("SomeService @Autowired")
    │              └─ ChromaDB 검색 → 결과 반환
    │
    ▼
LLM: 충분한 컨텍스트 확보 → 분석 결과 JSON 반환
```

**이 방식이 필요한 이유**: 에러는 `TestErrorController`에서 났지만 실제 원인은 `@Configuration` 클래스나 `@Bean` 정의, 또는 주입된 서비스에 있을 수 있습니다. LLM이 코드를 읽으면서 의존 관계를 따라가는 것이 중요합니다.

**문제 발생**: `gemma4:12b` 소형 모델에서 tool calling 판단이 불안정합니다.

```
시도 1: tool을 7번 호출하며 파일을 못 찾고 → 빈 응답 반환
시도 2: tool을 0번 호출하고 에러 메시지만 보고 바로 응답 → 소스 코드 미참조
```

---

### v3 — Hybrid (Python 결정론적 + LLM Agentic)

Python이 stack trace 파싱을 담당하고, LLM은 추가 탐색만 합니다.

```
에러 수신
    │
    ├─ [Python - 결정론적]
    │   stack trace 파싱
    │   "at com.example.TestErrorController.triggerNpe(TestErrorController.java:15)"
    │          │
    │          │ 정규식으로 경로 추출
    │          ▼
    │   src/main/java/com/example/TestErrorController.java
    │   src/test/java/com/example/TestErrorController.java  ← 둘 다 시도
    │          │
    │          │ 파일 읽기 (100% 정확)
    │          ▼
    │   초기 메시지에 파일 내용 포함
    │
    ├─ [LLM - Agentic, tool: search_files만]
    │   "에러 파일은 이미 있음, 근본 원인 탐색"
    │          │
    │          ├─ search_files("Config Bean NullPointerException")
    │          │     └─ 경로 + 내용 동시 반환 (read_file 별도 호출 불필요)
    │          │
    │          └─ 충분하면 바로 응답
    │
    ▼
분석 결과 JSON 반환 → DB 저장 → Slack 전송
```

**개선 포인트**:
- stack trace 파일 읽기는 LLM 판단 없이 **무조건** 실행
- `search_files`가 파일 경로뿐 아니라 **내용까지 반환** → LLM이 한 번의 tool call로 컨텍스트 확보
- `read_file` tool 제거 → LLM의 결정 복잡도 감소
- max iteration 5로 제한, 마지막 iteration 전 강제 응답 유도

---

## 실제 로그 비교

### v1 Pipeline RAG (실패 케이스)

```
[pipeline] RAG paths=['OllamaService.java', 'WeatherServiceImpl.java', ...]  ← 엉뚱한 파일
[pipeline] stack paths=[]                                                      ← 파일 못 찾음
[pipeline] calling ollama — files=5
→ 2분 후 httpx.ReadTimeout                                                    ← 타임아웃
[pipeline] record saved id=10                                                  ← 빈 결과 저장
```

### v3 Hybrid (성공 케이스)

```
[agent] pre-loaded stack trace files: ['src/.../TestErrorController.java']    ← 정확한 파일
[pipeline] calling ollama (agentic)
[agent] search_files query='Config Bean injection' → ['AppConfig.java', ...]  ← 근본 원인 탐색
[agent] done after 2 iterations
[pipeline] ollama response length=641                                          ← 정상 응답
[pipeline] record saved id=13
[pipeline] slack sent ts=...
```

---

## httpx Timeout 설정

Ollama 응답 시간은 모델 크기와 컨텍스트 길이에 따라 크게 달라집니다. `gemma4:12b` + 소스 파일 5개 기준으로 120초를 초과하는 경우가 발생했습니다.

백그라운드 태스크이므로 read timeout을 제거하고 connect timeout만 유지합니다.

```python
httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=10.0,  # Ollama 서버 다운 감지
        read=None,     # 생성 시간 무제한 (백그라운드 태스크)
        write=30.0,
        pool=5.0,
    )
)
```

---

## HTTP 요청/응답 로깅 미들웨어

Java의 HandlerInterceptor처럼 모든 요청/응답을 로깅합니다.

```python
class HttpLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        if request.url.path in _SKIP_PATHS:  # /health 제외
            return await call_next(request)

        body = await request.body()
        logger.info("→ %s %s | body=%s", request.method, request.url, _decode(body))

        start = time.perf_counter()
        response = await call_next(request)
        elapsed_ms = (time.perf_counter() - start) * 1000

        resp_body = b""
        async for chunk in response.body_iterator:
            resp_body += chunk

        logger.info("← %s %s | %d | %.0fms | body=%s",
                    request.method, request.url.path,
                    response.status_code, elapsed_ms, _decode(resp_body))

        return Response(content=resp_body, status_code=response.status_code,
                        headers=dict(response.headers), media_type=response.media_type)
```

---

## 정리

| 버전 | 방식 | 문제 |
|---|---|---|
| v1 Pipeline RAG | Python이 top-5 파일 선정 후 LLM 전달 | 엉뚱한 파일 선정, 타임아웃 |
| v2 Agentic RAG | LLM이 tool calling으로 직접 검색 | 소형 모델의 불안정한 tool calling |
| v3 Hybrid | Python(stack trace) + LLM(근본 원인 탐색) | — |

핵심은 **결정론적으로 풀 수 있는 것(stack trace → 파일 경로)은 Python이, 의미 추론이 필요한 것(근본 원인 탐색)은 LLM이** 담당하는 역할 분리입니다.

소형 로컬 모델의 한계를 보완하는 방법으로, LLM에게 판단을 맡기는 영역을 최소화하고 확실한 것은 코드로 처리하는 방향이 실용적이었습니다.
