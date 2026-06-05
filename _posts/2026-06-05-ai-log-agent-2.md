---
title: "AI Log Analysis Agent 구축기 (2) — Spring Boot 글로벌 예외 핸들러 & 웹훅 연동"
date: 2026-06-05 01:00:00 +0900
categories: [Project, DevSecOps]
tags: [spring-boot, java, webhook, exception-handler]
---

## 외부 서비스 연동 설계 원칙

log-agent와 연동하는 Spring Boot 서비스(`puppynote-server`)에는 **최소한의 코드만 추가**한다.

- git, commit hash 같은 로직은 일절 포함하지 않는다
- 에러 페이로드를 구성해서 POST 요청만 보낸다
- 비동기(`CompletableFuture`)로 실행해서 기존 응답에 영향을 주지 않는다

---

## ErrorEventPayload — 에러 정보 DTO

```java
@Getter
@Builder
public class ErrorEventPayload {
    private String serverName;
    private String serverIp;
    private String errorType;
    private String message;
    private String stackTrace;
    private String requestMethod;
    private String requestUrl;
    private String requestBody;
    private int responseStatus;
}
```

`gitCommit` 같은 필드는 없다. log-agent가 git에서 직접 가져온다.

---

## LogAgentWebhookService — 웹훅 발송 서비스

```java
@Slf4j
@Service
public class LogAgentWebhookService {

    private final RestClient restClient = RestClient.create();
    private final String serverIp;

    @Value("${log-agent.webhook-url}")
    private String webhookUrl;

    @Value("${log-agent.server-name}")
    private String serverName;

    public LogAgentWebhookService() {
        String ip;
        try {
            ip = InetAddress.getLocalHost().getHostAddress();
        } catch (Exception e) {
            ip = "unknown";
        }
        this.serverIp = ip;
    }

    public void sendError(Exception e, HttpServletRequest request) {
        CompletableFuture.runAsync(() -> {
            ErrorEventPayload payload = ErrorEventPayload.builder()
                .serverName(serverName)
                .serverIp(serverIp)           // 코드에서 직접 IP 획득
                .errorType(e.getClass().getSimpleName())
                .message(e.getMessage() != null ? e.getMessage() : "")
                .stackTrace(getStackTrace(e))
                .requestMethod(request.getMethod())
                .requestUrl(request.getRequestURI())
                .responseStatus(500)
                .build();

            restClient.post()
                .uri(webhookUrl)
                .contentType(MediaType.APPLICATION_JSON)
                .body(payload)
                .retrieve()
                .toBodilessEntity();
        });
    }
}
```

**핵심 포인트**: `InetAddress.getLocalHost().getHostAddress()`로 서버 IP를 코드에서 직접 가져온다.
환경변수로 IP를 주입하면 배포 환경마다 설정이 달라지는 문제가 생기기 때문이다.

---

## 글로벌 예외 핸들러에 연동

```java
@RestControllerAdvice
public class ApiControllerAdvice {

    private final LogAgentWebhookService logAgentWebhookService;

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(
            Exception e, HttpServletRequest request) {

        logAgentWebhookService.sendError(e, request);  // 비동기 발송

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(e.getMessage()));
    }
}
```

기존 예외 핸들러에 **한 줄만 추가**하면 모든 예외가 자동으로 log-agent로 전송된다.

---

## application-dev.yml 설정

```yaml
log-agent:
  webhook-url: http://log-agent-api.sangkihan.co.kr/api/v1/webhook/error
  server-name: puppynote
```

서버명과 웹훅 URL만 환경별로 관리하면 된다.

---

## 테스트용 에러 컨트롤러

```java
@RestController
@RequestMapping("/test")
@Profile("prd")
public class TestErrorController {

    @GetMapping("/error")
    public void triggerError() {
        throw new RuntimeException("log-agent 연동 테스트용 강제 에러입니다.");
    }

    @GetMapping("/npe")
    public String triggerNpe() {
        String value = null;
        return value.toUpperCase();  // NullPointerException 발생
    }
}
```

`@Profile("prd")`로 운영 환경에서만 활성화된다.
`/test/npe`를 호출하면 NullPointerException이 발생하고 log-agent로 웹훅이 전송된다.

---

## 정리

| 포인트 | 내용 |
|---|---|
| 코드 최소화 | git 로직 없이 에러 페이로드만 전송 |
| 비동기 처리 | `CompletableFuture`로 기존 응답에 영향 없음 |
| IP 자동 감지 | `InetAddress.getLocalHost()`로 환경변수 설정 불필요 |
| 확장성 | 어떤 서비스든 동일한 규격으로 연동 가능 |
