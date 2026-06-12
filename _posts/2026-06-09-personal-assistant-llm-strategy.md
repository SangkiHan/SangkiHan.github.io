---
layout: post
title: "개인 비서 LLM 전략 — Gemini 우선, Ollama 폴백"
date: 2026-06-09 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Gemini, Ollama, LLM, Fallback, LangChain, Python, RateLimit]
---

처음엔 Ollama만 쓰다가 Gemini를 주력으로 바꿨다. 이유는 속도다.

Ollama에서 `gemma4:12b`를 쓰면 응답 시간이 20~30초다. Gemini API는 2~3초다. 일상적으로 쓰는 개인 비서에서 응답을 30초 기다리는 건 너무 불편하다.

그렇다고 Ollama를 완전히 버리면 API 할당량이 소진됐을 때 쓸 수 없게 된다. Gemini 무료 티어는 모델마다 하루 요청 한도가 있다. 이걸 넘기면 다음날까지 기다려야 한다.

결론: Gemini를 우선 쓰고, 할당량 초과(429 에러) 시 자동으로 Ollama로 전환한다.

---

## 구현

```python
def _is_quota_error(e: Exception) -> bool:
    s = str(e).lower()
    return any(k in s for k in ("quota", "429", "resource exhausted", "rate limit"))


def _make_gemini():
    from langchain_google_genai import ChatGoogleGenerativeAI
    return ChatGoogleGenerativeAI(
        model=settings.gemini_model,
        google_api_key=settings.gemini_api_key,
    )


def _make_ollama():
    return ChatOllama(
        model=settings.ollama_model,
        base_url=settings.ollama_host,
        think=False,
    )


async def chat(user_id: str, message: str) -> str:
    use_gemini = bool(settings.gemini_api_key)
    llm = _make_gemini() if use_gemini else _make_ollama()

    try:
        result = await _invoke_graph(llm, tools, message, timeout=120)
    except Exception as e:
        if use_gemini and _is_quota_error(e):
            logger.warning("[agent] Gemini 할당량 초과 → Ollama로 전환: %s", e)
            llm = _make_ollama()
            result = await _invoke_graph(llm, tools, message, timeout=300)
        else:
            raise
```

`_is_quota_error`는 예외 메시지에서 키워드를 찾는다. Gemini API가 반환하는 `RESOURCE_EXHAUSTED` 상태와 HTTP 429 코드를 같이 확인한다. LangChain이 이 에러를 내부적으로 몇 번 재시도한 뒤 예외를 던지는데, 그 시점에 Ollama로 전환한다.

timeout은 Gemini 120초, Ollama 300초로 다르게 설정했다. Gemini는 빠르게 응답하므로 120초면 충분하고, Ollama는 처리가 느려서 여유를 줬다.

---

## 모델 선택 실수

처음에 `gemini-2.5-flash-lite`를 썼다. 이름에 "lite"가 붙어서 경량 모델일 거라 생각했는데, 실제로는 무료 티어 일 요청 한도가 20회다.

반면 `gemini-2.0-flash`는 1,500회/일이다.

| 모델 | 무료 티어 일 요청 한도 |
|---|---|
| gemini-2.5-flash-lite | 20회/일 |
| gemini-2.0-flash | 1,500회/일 |
| gemini-1.5-flash | 1,500회/일 |

최신 모델이라고 무조건 더 관대한 게 아니다. 신규 모델은 오히려 무료 티어가 제한적인 경우가 있다. 개인 프로젝트에서 무료로 쓰려면 모델별 할당량을 먼저 확인해야 한다.

---

## Gemini 도구 호출 문제

LLM을 바꾸면서 새로운 문제가 생겼다. Ollama에서 Gemini로 바꿨더니 에이전트가 도구를 거의 호출하지 않았다.

```
[agent] msg[0] type=HumanMessage
[agent] msg[1] type=AIMessage content_len=45 tool_calls=0
```

"안과 예약 추가해줘"라고 했는데 `add_calendar_event`를 호출하지 않고 "등록해드렸습니다"라는 거짓 응답을 했다.

원인은 시스템 프롬프트가 너무 약했다. "일정이 있으면 Calendar를 사용하세요" 정도의 부드러운 문장으로는 Gemini가 도구 호출 없이 텍스트로 답변하는 경우가 많았다.

강하게 바꿨다.

```python
def _build_system_prompt() -> str:
    today = date.today().isoformat()
    return f"""당신은 유능한 개인 비서입니다. 오늘 날짜: {today}. 항상 한국어로 답변하세요.

**도구 호출 규칙 — 반드시 준수:**

| 상황 | 호출할 도구 |
|---|---|
| 일정 추가·예약·약속 언급 | add_calendar_event |
| 일정 조회 요청 | list_calendar_events |
| 지출·비용·결제 언급 | add_expense |
| 알림·리마인더 설정 | set_reminder + add_todo (둘 다 호출) |
...

- 도구 없이 텍스트만 답변하지 마세요 (기억·기록이 필요한 요청은 반드시 도구 호출)"""
```

표 형식으로 "어떤 상황에 어떤 도구"를 명시하고, 마지막에 "도구 없이 텍스트만 답변하지 마세요"를 못 박았다. 이후 도구 호출 누락이 크게 줄었다.

LLM마다 프롬프트 민감도가 다르다. Ollama는 느슨하게 써도 도구를 잘 쓰던 게, Gemini에서 갑자기 안 쓰는 일이 생길 수 있다. LLM을 바꾸면 프롬프트도 다시 검증해야 한다.

---

## Ollama에서 think=False

```python
ChatOllama(model=settings.ollama_model, base_url=settings.ollama_host, think=False)
```

`think=False`는 extended thinking 모드를 끄는 옵션이다. `gemma4:12b`를 포함한 일부 모델은 기본적으로 내부 추론 과정을 출력한다. 응답 속도가 더 느려지고 `<thinking>...</thinking>` 태그가 응답에 포함되어 파싱을 복잡하게 만들었다. `think=False`로 끄면 최종 답변만 반환한다.

---

## 두 LLM의 동작 차이 요약

| 항목 | Gemini 2.0 Flash | Ollama gemma4:12b |
|---|---|---|
| 응답 속도 | 2~3초 | 20~30초 |
| 일 요청 한도 | 1,500회 | 무제한 (로컬) |
| 도구 호출 정확도 | 시스템 프롬프트 강도에 민감 | 상대적으로 관대 |
| 비용 | 무료 티어 초과 시 유료 | 전기료 |

일상 사용에서는 Gemini가 압도적으로 빠르다. Ollama는 할당량 소진 시 폴백 용도로 충분하다.
