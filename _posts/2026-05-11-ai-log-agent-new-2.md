---
layout: post
title: "AI 오류 자동 분석 에이전트 구축기 (2) — FastAPI와 파이프라인 설계"
date: 2026-05-11 11:00:00 +0900
categories: [Project, AI]
tags: [FastAPI, Python, Spring Boot, Webhook, BackgroundTasks]
---

에이전트의 뼈대가 되는 FastAPI 서버를 어떻게 설계했는지, 그리고 SEMS(Spring Boot)에서 어떻게 에러를 보내는지 정리한다.

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

## SEMS: ControllerAdvice 추가

Spring Boot에서 발생하는 모든 예외를 한 곳에서 잡아 FastAPI로 전송한다. 기존 `@ControllerAdvice`에 웹훅 전송 로직을 추가했다.

```java
// GlobalExceptionHandler.java (SEMS)
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(
        Exception ex, HttpServletRequest request) {

    ErrorPayload payload = ErrorPayload.builder()
            .serverName("SEMS")
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

수신 즉시 200을 반환하고 분석 파이프라인은 `BackgroundTasks`로 넘긴다. 클라이언트(SEMS)가 응답을 기다리지 않아도 된다.

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

        # 4. 원본 파일 저장 + 패치 사전 계산
        suggestion = _enrich_with_patched_content(suggestion, server.id)

        # 5. DB 저장 + Slack 전송
        record = AnalysisRecord(...)
        db.add(record)
        await db.commit()
        await slack_service.send_analysis(server, record)
```

---

## 파일 패치 전략 — 삽질의 기록

처음 설계는 단순했다. LLM이 `before`/`after` 스니펫을 주면 파일에서 `before`를 찾아 `after`로 교체한다. 그런데 **LLM이 주는 `before`가 실제 파일과 미묘하게 달라서** substring 매칭이 계속 실패했다.

문제 패턴:
- 줄 끝 공백 차이 (`\r\n` vs `\n`)
- 들여쓰기가 빠진 채로 출력 (클래스 내부 메서드인데 0칸 들여쓰기로 출력)
- LLM이 실제로 없는 코드를 hallucinate

이를 해결하기 위해 4단계 fallback 전략을 구현했다.

```python
def _find_and_replace(original, before, after):
    # 1차: 정확한 매칭
    if before in original:
        return original.replace(before, after, 1)

    # 2차: CRLF → LF 정규화
    orig_lf = original.replace("\r\n", "\n")
    before_lf = before.replace("\r\n", "\n")
    if before_lf in orig_lf:
        return orig_lf.replace(before_lf, after_lf, 1)

    # 3차: 들여쓰기 보정 (4/8/2/12칸 시도)
    before_dedented = textwrap.dedent(before_lf)
    for indent in ("    ", "        ", "  ", "\t", "            "):
        re_before = "\n".join(indent + l if l.strip() else l
                              for l in before_dedented.splitlines())
        if re_before in orig_lf:
            ...
            return orig_lf.replace(re_before, re_after, 1)

    # 4차: 라인 단위 fuzzy 매칭 (공백·빈 줄 완전 무시)
    return _fuzzy_replace(orig_lf, before_lf, after_lf)
```

4차 fuzzy 매칭은 각 라인을 `strip()`해서 순서대로 찾는다. LLM이 들여쓰기를 어떻게 출력해도 내용만 맞으면 매칭된다.

그런데 여기서도 LLM이 아예 없는 코드를 `before`로 hallucinate하면 4단계 모두 실패한다.

### 최종 해결: 원본 저장 + 수락 시 LLM 위임

분석 시점에 수정 대상 파일의 **원본 전체**를 DB에 저장해두고, 수락 버튼 클릭 시 3단계로 패치를 시도한다.

```
분석 시점
  → 원본 파일 전체를 original_content로 DB 저장
  → before→after fuzzy 매칭 시도
  → 성공: patched_content도 함께 저장

수락 클릭
  → patched_content 있으면 바로 GitHub 푸시
  → 없으면 1단계: original_content에 fuzzy 매칭 재시도
  → 실패 시 2단계: LLM에게 직접 위임
      "이 파일에 이 변경사항을 적용해줘"
      → Ollama가 원본 + before/after를 보고 완성 파일 반환
  → 그래도 실패 시: 수정 없이 description-only PR
```

```python
# github_service.py — 수락 시 패치 적용
if not patched_content:
    result = _find_and_replace(original_content, before, after)  # fuzzy
    if result:
        patched_content = result
    else:
        # LLM에게 패치 적용 위임
        patched_content = await apply_patch_with_llm(original_content, before, after)
```

LLM은 `before`가 조금 틀려도 의도를 파악해서 올바른 위치에 `after`를 적용한다. string 매칭으로는 절대 해결 못 하는 hallucination 문제를 LLM으로 우회한다.

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

SEMS                  개발자
     │  예외 발생                    │
     │  ControllerAdvice 처리        │
     │──────────────────────────────▶│  Slack 단순 알림
     │  "에러 발생했습니다"           │  로그 직접 확인 필요
     │                               │  원인 파악 수동으로

[After — 에이전트 도입 후]

SEMS    FastAPI(노트북)     Ollama(M4)     GitHub     Slack
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

---

➡️ 다음 글: [AI 오류 자동 분석 에이전트 구축기 (3) — Embedding 모델과 ChromaDB RAG 구축](/posts/ai-log-agent-new-3/)
