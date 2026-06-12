---
layout: post
title: "Slack으로 부리는 개인 비서 — FastAPI + LangGraph로 만든 AI 에이전트"
date: 2026-05-28 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Slack, FastAPI, LangGraph, LLM, Ollama, Python, PersonalAssistant]
---

log-agent를 운영하면서 "에러 분석 에이전트를 만드는 데 쓴 구조를 일상에도 쓸 수 있지 않을까"는 생각이 들었다. 일정 관리, 지출 기록, 할 일 목록, 리마인더 같은 것들을 하나의 Slack 채널에서 말로 처리하는 개인 비서다.

기획한 기능 목록은 이렇다.

| 기능 | 수단 |
|---|---|
| 일정 추가·조회 | Google Calendar API |
| 지출 기록·조회 | Google Sheets API |
| 할 일 추가·완료 | MySQL |
| 리마인더 설정·취소 | APScheduler + MySQL |
| 장기 기억 | ChromaDB + Ollama embedding |
| 웹 검색 | DuckDuckGo |

이걸 모두 Slack DM 하나로 처리한다. "다음주 화요일 오후 2시 치과 예약해줘", "스타벅스 6500원", "30분 후에 약 먹으라고 알려줘" — 이 세 문장이 각각 Calendar, Sheets, APScheduler로 연결된다.

---

## 전체 구조

```
Slack DM
    ↓
FastAPI (Slack Events API 수신)
    ↓
LangGraph ReAct 에이전트
    ↓
도구 선택 및 실행
    ├── Google Calendar API
    ├── Google Sheets API
    ├── ChromaDB (장기 기억)
    ├── MySQL (Todo / Reminder)
    └── DuckDuckGo (웹 검색)
    ↓
Slack DM 응답
```

핵심은 LangGraph의 ReAct 에이전트다. 사용자 메시지를 받으면 어떤 도구를 쓸지 LLM이 판단하고, 도구를 실행한 결과를 다시 LLM에게 넘겨 최종 답변을 만든다.

---

## Slack Events API 연동

Slack 앱을 만들고 "Event Subscriptions"를 활성화하면 메시지 수신 시 서버로 HTTP POST가 들어온다.

```python
@router.post("/events")
async def slack_events(request: Request, background_tasks: BackgroundTasks):
    body = await request.json()
    event = body.get("event", {})

    if event.get("type") == "message" and not event.get("bot_id"):
        user_id = event.get("user")
        text = event.get("text", "")
        channel = event.get("channel")
        if user_id == settings.slack_my_user_id:
            background_tasks.add_task(handle_message, user_id, text, channel)

    return {"ok": True}
```

`background_tasks`로 처리하는 이유가 있다. Slack은 이벤트를 받고 3초 안에 HTTP 200을 응답하지 않으면 재전송한다. LLM 응답은 몇 초에서 몇십 초가 걸리기 때문에 즉시 200을 반환하고 백그라운드에서 처리한다.

`bot_id` 체크는 필수다. 봇 자신이 메시지를 보낼 때도 이벤트가 발생하는데, 여기서 걸러주지 않으면 무한루프가 된다.

---

## 도구 정의 방식

LangGraph에서 도구는 `@langchain_tool` 데코레이터로 정의한다.

```python
def _make_tools(user_id: str):
    @langchain_tool
    async def add_calendar_event(title: str, date: str, time: str, description: str = "") -> str:
        """Google Calendar에 일정을 추가합니다. date: YYYY-MM-DD, time: HH:MM"""
        from app.services import calendar_service
        loop = asyncio.get_event_loop()
        result = await asyncio.wait_for(
            loop.run_in_executor(None, lambda: calendar_service.add_event(title, date, time, description)),
            timeout=45,
        )
        return result

    # ... 다른 도구들

    return [add_calendar_event, list_calendar_events, add_expense, ...]
```

`user_id`를 클로저로 캡처하는 방식을 쓴다. 각 사용자 요청마다 도구 세트를 새로 만들면 user_id가 도구 함수 안에 자동으로 바인딩된다.

도구마다 `asyncio.wait_for(timeout=45)`를 붙인다. 외부 API(Google Calendar, Sheets)는 간혹 응답이 늦는데, 하나의 도구가 막히면 전체 에이전트가 멈추기 때문이다.

---

## 에이전트 실행

```python
async def _invoke_graph(llm, tools, message: str, timeout: int):
    graph = create_react_agent(model=llm, tools=tools, prompt=_build_system_prompt())
    return await asyncio.wait_for(
        graph.ainvoke(
            {"messages": [("user", message)]},
            config={"recursion_limit": 20},
        ),
        timeout=timeout,
    )
```

`recursion_limit=20`은 LangGraph 노드 실행 횟수 기준이다. 도구 1회 호출 = 2 스텝(agent 노드 + tools 노드)이므로, 20은 최대 도구 9회 호출을 허용한다. 처음엔 6으로 설정했다가 "19시 30분 리마인더를 18시로 수정해줘" 같은 요청에서 list → cancel → set_reminder 세 번을 연속 호출해야 해서 한도에 걸렸다. 실제로 운영해보며 조정했다.

---

## 시스템 프롬프트 설계

에이전트가 도구를 잘 쓰도록 유도하는 게 생각보다 까다롭다.

```python
def _build_system_prompt() -> str:
    today = date.today().isoformat()
    return f"""당신은 유능한 개인 비서입니다. 오늘 날짜: {today}. 항상 한국어로 답변하세요.

**도구 호출 규칙 — 반드시 준수:**

| 상황 | 호출할 도구 |
|---|---|
| 일정 추가·예약·약속 언급 | add_calendar_event |
| 알림·리마인더 설정 | set_reminder + add_todo (둘 다 호출) |
| 리마인더 목록 조회 | list_reminders |
...

**Slack mrkdwn 형식 — 반드시 준수:**
- 굵게: *텍스트* (별표 1개) — **텍스트** 사용 금지
- 목록: • 항목 — *   항목 사용 금지
- 헤더 없음: *굵게* 로 대체 — ### 사용 금지
```

Slack은 마크다운이 아니라 자체 mrkdwn 형식을 쓴다. `**굵게**`나 `### 제목`이 그대로 텍스트로 출력된다. 시스템 프롬프트에 명시하지 않으면 LLM은 표준 마크다운을 쓴다.

"오늘 날짜"를 매번 포함하는 이유도 있다. LLM의 학습 데이터 기준일이 있기 때문에, "다음주 토요일"을 실제 날짜로 바꾸려면 오늘이 언제인지 알려줘야 한다.

---

## 다음

이 구조 위에 기능을 하나씩 붙였다. 다음 글에서는 ChromaDB를 이용한 장기 기억 구현을 정리한다.
