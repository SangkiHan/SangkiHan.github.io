---
layout: post
title: "AI 오류 자동 분석 에이전트 구축기 (2) — FastAPI와 파이프라인 설계"
date: 2026-05-11 11:00:00 +0900
categories: [Project, AI]
tags: [FastAPI, Python, Spring Boot, Webhook, BackgroundTasks]
---

에이전트의 뼈대가 되는 FastAPI 서버를 어떻게 설계했는지, 그리고 puppynote-server(Spring Boot)에서 어떻게 에러를 보내는지 정리한다.

## 왜 FastAPI인가

에이전트 백엔드로 Python을 선택한 이유는 LLM 생태계 때문이다. Ollama Python SDK, LangChain, ChromaDB 등 LLM 관련 라이브러리가 Python에 집중되어 있다. Spring Boot에서 같은 걸 구현하려면 직접 HTTP 통신을 전부 손으로 짜야 한다.

FastAPI를 고른 이유는 **async 지원**이다. LLM 분석, 임베딩, GitHub API 호출이 모두 I/O 대기이므로 `asyncio`로 동시에 처리하는 게 자연스럽다.

| 요소 | 선택 | 이유 |
|---|---|---|
| 언어 | Python | LLM 라이브러리 생태계 |
| 프레임워크 | FastAPI | async-first, Pydantic 스키마 |
| DB | SQLite + SQLAlchemy async | 단순 단일 노드 |
| 벡터 DB | ChromaDB | 로컬 운영 가능 |

---

## puppynote-server: ControllerAdvice 추가

Spring Boot에서 발생하는 모든 예외를 한 곳에서 잡아 FastAPI로 전송한다. 기존 `@ControllerAdvice`에 웹훅 전송 로직을 추가했다.

```java
// GlobalExceptionHandler.java (puppynote-server)
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(
        Exception ex, HttpServletRequest request) {

    ErrorPayload payload = ErrorPayload.builder()
            .serverName("puppynote-server")
            .serverIp(getLocalIp())
            .errorType(ex.getClass().getSimpleName())
            .message(ex.getMessage())
            .requestMethod(request.getMethod())
            .requestUrl(request.getRequestURI())
            .stackTrace(getStackTrace(ex))
            .build();

    // 비동기 전송 — 웹훅 실패가 원래 응답에 영향 없도록
    CompletableFuture.runAsync(() -> webhookClient.sendError(payload));

    return ResponseEntity.status(500).body(toErrorResponse(ex));
}
```

`CompletableFuture.runAsync()`로 비동기 전송해서 웹훅이 실패해도 원래 API 응답에 영향을 주지 않는다.

---

## FastAPI: 웹훅 수신

```python
# app/api/v1/routes/webhook.py

@router.post("/error")
async def receive_error(
    payload: ErrorEventPayload,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    # 등록된 서버인지 IP로 확인
    result = await db.execute(
        select(Server)
        .join(ServerHost, ServerHost.server_id == Server.id)
        .where(ServerHost.host == payload.server_ip, Server.is_active == True)
    )
    server = result.scalar_one_or_none()
    if not server:
        return {"status": "ignored"}

    # 60초 내 동일 에러 중복 방지
    key = _dedup_key(payload.server_ip, payload.stack_trace)
    now = asyncio.get_event_loop().time()
    if key in _recent_errors and now - _recent_errors[key] < 60:
        return {"status": "deduplicated"}
    _recent_errors[key] = now

    # 즉시 200 반환, 분석은 백그라운드에서
    background_tasks.add_task(_analysis_pipeline, server, payload)
    return {"status": "received"}
```

수신 즉시 200을 반환하고 분석 파이프라인은 `BackgroundTasks`로 넘긴다. 클라이언트(puppynote-server)가 응답을 기다리지 않아도 된다.

---

## 분석 파이프라인

```python
async def _analysis_pipeline(server: Server, payload: ErrorEventPayload) -> None:
    async with AsyncSessionLocal() as db:
        raw_log = _build_raw_log(payload)

        # 1. git clone (항상 새로 클론 — 상태 오염 방지)
        await asyncio.to_thread(
            git_service.fetch, server.id, server.git_repo_url,
            server.git_branch, server.github_token or ""
        )
        commit = await asyncio.to_thread(
            git_service.get_remote_head, server.id, server.git_branch
        )

        # 2. RAG 인덱싱 (commit 변경 시에만)
        if not rag_service.is_indexed(server.id, commit):
            chunks = await asyncio.to_thread(
                git_service.list_all_files_at_commit, server.id, commit
            )
            await rag_service.index_repo(server.id, commit, chunks)

        # 3. LLM 분석
        suggestion = await ollama_service.analyze_log(
            server.id, raw_log, payload.stack_trace
        )

        # 4. 수정 결과를 DB에 미리 저장 (approve 시 즉시 사용)
        suggestion = _enrich_with_patched_content(suggestion, server.id)

        # 5. DB 저장 + Slack 전송
        record = AnalysisRecord(...)
        db.add(record)
        await db.commit()
        await slack_service.send_analysis(server, record)
```

파이프라인의 핵심 원칙은 **분석 시점에 모든 계산을 완료**하는 것이다. `_enrich_with_patched_content()`가 LLM이 제안한 `before/after`를 실제 파일에 적용해 완성된 파일 내용(`patched_content`)을 미리 계산한다. 수락 버튼 클릭 시에는 이미 계산된 내용을 그대로 GitHub에 푸시하면 된다.

---

## Slack 수락/거절 처리

```python
# app/api/v1/routes/slack_events.py

@router.post("/actions")
async def handle_slack_action(request: Request, background_tasks: BackgroundTasks):
    # ... payload 파싱 ...

    # status 즉시 업데이트 + 200 반환
    async with AsyncSessionLocal() as db:
        record = await db.get(AnalysisRecord, record_id)
        record.status = "approved" if action_id == "approve_fix" else "rejected"
        await db.commit()

    if action_id == "approve_fix":
        background_tasks.add_task(_handle_approve, record_id)  # GitHub PR 생성
    else:
        background_tasks.add_task(_handle_reject, record_id)   # Slack 알림만

    return Response(status_code=200)  # Slack에 즉시 응답
```

Slack은 3초 내에 200 응답이 없으면 타임아웃으로 처리한다. GitHub PR 생성에 시간이 걸리므로 `BackgroundTasks`로 분리하고 즉시 200을 반환한다.

---

## 통신 흐름 상세

```
[Before — 에러 발생 시]

puppynote-server                  개발자
     │  예외 발생                    │
     │  ControllerAdvice 처리        │
     │──────────────────────────────▶│  Slack 단순 알림
     │  "에러 발생했습니다"           │  로그 직접 확인 필요
     │                               │  원인 파악 수동으로

[After — 에이전트 도입 후]

puppynote-server    FastAPI(노트북)     Ollama(M4)     GitHub     Slack
     │ 예외 발생        │                   │              │          │
     │──POST /error──▶ │                   │              │          │
     │◀── 200 ──────── │                   │              │          │
     │ (비동기 전송)    │──BackgroundTask   │              │          │
     │                 │──git clone──────▶ 로컬 저장소    │          │
     │                 │──RAG 인덱싱──────▶ ChromaDB      │          │
     │                 │──analyze────────▶ │              │          │
     │                 │                  │ search_files  │          │
     │                 │◀─────────────────  JSON 분석결과 │          │
     │                 │──────────────────────────────────────────────▶
     │                 │                               원인+수정+버튼 │
     │                 │                          개발자 수락 클릭    │
     │                 │◀────────────────────────── POST /slack/actions
     │                 │──BackgroundTask   │              │          │
     │                 │──GitHub PR 생성────────────────▶ │          │
     │                 │──────────────────────────────────────────────▶
     │                 │                                     PR URL  │
```
