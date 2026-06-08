---
layout: post
title: "LangChain 도입기 - LLM 에이전트 루프를 직접 짜다가 생긴 일"
date: 2026-06-08 10:00:00 +0900
categories: [AI, LangChain]
tags: [LangChain, LLM, Ollama, FastAPI, Agent, Python]
---

서버 에러 로그를 LLM이 분석해서 Slack으로 원인과 수정 코드를 보내주는 시스템을 만들고 있었다. LLM이 단순히 로그를 받아서 답을 내는 게 아니라, 필요하면 직접 소스 파일을 검색해가며 분석하는 **에이전트** 구조였다. 처음엔 이 루프를 직접 구현했는데, 문제가 생겼다.

## 직접 구현한 에이전트 루프

LLM 에이전트의 동작 방식은 단순하다.

```
1. LLM에게 에러 로그를 준다
2. LLM이 "이 파일 검색해줘" 라고 요청하면 → 검색해서 돌려준다
3. LLM이 충분히 파악했다고 판단하면 → 최종 JSON 응답을 반환한다
```

이걸 처음에 직접 구현했다.

```python
_MAX_TOOL_ITERATIONS = 5

for iteration in range(_MAX_TOOL_ITERATIONS):
    response = await httpx.post(ollama_host + "/api/chat", json={
        "model": "gemma4:12b",
        "messages": messages,
        "tools": tools,
        "stream": False,
    })
    message = response.json().get("message", {})

    tool_calls = message.get("tool_calls") or []
    if not tool_calls:
        return message.get("content", "")  # 최종 응답

    for call in tool_calls:
        result = await search_files(call["function"]["arguments"]["query"])
        messages.append({"role": "tool", "content": result})
```

동작은 했다. 그런데 운영하면서 문제가 하나씩 드러났다.

---

## 문제 1: 5회 제한의 근거가 없다

`_MAX_TOOL_ITERATIONS = 5`는 그냥 "적당한 것 같다"는 감으로 정한 숫자였다. 간단한 에러는 2~3번이면 충분하지만, 복잡한 의존성을 추적해야 하는 에러는 5번으로 부족할 수 있다.

그렇다고 `while True`로 돌리면 안 된다. LLM이 루프에 빠질 수 있기 때문이다.

```
iteration 1 → search_files("TestErrorController")
iteration 2 → 파일 받고 나서 또 search_files("TestErrorController")
iteration 3 → 또 search_files("TestErrorController")
...무한반복
```

LLM이 같은 파일을 계속 보려고 하면 탈출 조건이 없다. 시간 제한도 없으니 Slack에는 영원히 응답이 안 온다.

---

## 문제 2: 에러 로그가 있는데 LLM이 아무것도 못 찾으면?

`max_iterations`에 도달했을 때 코드는 `_fallback_analyze()`를 호출하는 것으로 처리했다. 빈 응답을 반환하거나 에러를 던지는 것보다는 낫지만, "지금까지 수집한 정보로 최선의 답을 내줘"를 요청하는 게 더 자연스럽다. 이미 검색까지 다 해놓고 그냥 버리는 셈이었다.

---

## 문제 3: 스택 트레이스 파싱이 Java/Python 전용

에이전트가 검색을 시작하기 전에 스택 트레이스에서 파일 경로를 정규식으로 뽑아서 LLM에게 미리 전달하는 로직이 있었다.

```python
java_pattern = re.compile(r"at ([\w$.]+)\.\w+\((\w+\.(?:java|kt)):\d+\)")
python_pattern = re.compile(r'File "([^"]+\.py)", line \d+')
# TypeScript, Go는 패턴 없음
```

TypeScript나 Go 프로젝트를 연결하면 이 단계에서 아무것도 못 가져온다. 반면 RAG 인덱싱은 `.ts`, `.go` 등 여러 언어를 지원하고 있었으니 앞뒤가 맞지 않는 구조였다. 그리고 애초에 LLM이 스택 트레이스를 보고 스스로 `search_files`를 호출하면 되는데, 왜 별도로 정규식까지 짜서 미리 주는 걸까 싶었다.

---

## LangChain이란

LLM 애플리케이션을 만들기 위한 Python 프레임워크다. "LLM + 도구 + 루프 제어"라는 반복되는 패턴을 추상화해서 제공한다.

핵심 개념은 세 가지다.

**Chain**: LLM 호출과 후처리를 파이프라인으로 연결한다.

**Tool**: LLM이 호출할 수 있는 함수를 정의한다. `@tool` 데코레이터로 감싸면 LangChain이 LLM에게 함수 설명을 자동으로 전달한다.

**Graph (create_react_agent)**: 에이전트 루프 자체를 담당한다. Tool 호출 → 결과 전달 → 반복의 흐름을 그래프 노드로 표현하며, `recursion_limit`(스텝 제한)으로 무한루프를 방지한다. LangChain 1.x부터는 내부적으로 LangGraph 위에서 동작한다.

직접 구현했던 for 루프가 하는 일을 `create_react_agent`가 대신한다.

---

## 도입 후 코드

기존에는 for 루프 안에서 Ollama API를 직접 호출하고, tool_calls를 수동으로 파싱하고, 메시지 배열을 직접 관리했다. 바뀐 코드는 다음과 같다.

**Tool 정의**

```python
from langchain_core.tools import tool as langchain_tool

def _make_search_tool(server_id: int, repo_path: Path):
    @langchain_tool
    async def search_files(query: str) -> str:
        """에러의 근본 원인과 관련된 소스 파일을 검색하고 내용을 반환합니다."""
        paths = await rag_service.search_relevant_files(server_id, query, n_results=3)
        results = {}
        for path in paths:
            try:
                content = (repo_path / path).read_text(encoding="utf-8", errors="replace")
                results[path] = content[:2000]
            except FileNotFoundError:
                pass
        return json.dumps(results, ensure_ascii=False)
    return search_files
```

`@langchain_tool` 데코레이터를 붙이면 함수의 docstring을 LLM에게 도구 설명으로 자동 전달한다. `server_id`와 `repo_path`는 클로저로 캡처해서 매 요청마다 동적으로 생성한다.

**에이전트 그래프 설정**

```python
import asyncio
from langchain_ollama import ChatOllama
from langgraph.errors import GraphRecursionError
from langgraph.prebuilt import create_react_agent

llm = ChatOllama(
    model=settings.ollama_model,
    base_url=settings.ollama_host,
    think=False,
)

tools = [_make_search_tool(server_id, repo_path)]
graph = create_react_agent(model=llm, tools=tools, prompt=SYSTEM_PROMPT)

try:
    result = await asyncio.wait_for(
        graph.ainvoke(
            {"messages": [("user", f"다음 에러 로그를 분석해줘:\n\n```\n{raw_log}\n```")]},
            config={"recursion_limit": 30},
        ),
        timeout=120,
    )
    raw = result["messages"][-1].content
except GraphRecursionError:
    raw = await _fallback_analyze(raw_log)  # 스텝 한계 도달
except asyncio.TimeoutError:
    raw = await _fallback_analyze(raw_log)  # 시간 초과
```

`recursion_limit=30`인 이유가 있다. LangGraph는 "LLM 호출 1번 + 툴 실행 1번"을 각각 하나의 스텝으로 센다. 즉 툴 호출 1회 = 2 스텝이므로, `recursion_limit=30`은 실질적으로 툴 15회 호출에 해당한다.

시간 제한은 `asyncio.wait_for(timeout=120)`으로 감싸서 처리한다. 한계 도달 시에는 `GraphRecursionError`가 발생하는데, 이를 잡아서 `_fallback_analyze()`로 넘긴다. fallback은 툴 없이 단순 one-shot 프롬프트로 재시도한다.

---

## 전후 비교

| 항목 | 직접 구현 | LangChain + LangGraph |
|---|---|---|
| 횟수 제한 | 5회 (임의) | recursion_limit=30 (툴 15회) |
| 시간 제한 | 없음 | asyncio.wait_for(timeout=120) |
| 루프 감지 | 없음 | GraphRecursionError 자동 발생 |
| 한계 도달 시 | fallback 재시도 | GraphRecursionError catch → fallback |
| 스택 트레이스 파싱 | Java/Python 전용 정규식 | 제거 (LLM이 직접 검색) |
| Tool 관리 | 수동 JSON 파싱 | `@langchain_tool` 자동 처리 |
| 코드 라인 수 | ~90줄 | ~40줄 |

---

## FastAPI와 LangChain을 같이 쓰는 이유

둘 다 프레임워크인데 동시에 쓰는 게 어색하게 느껴질 수 있다. 그런데 역할이 완전히 다른 계층이라 충돌하지 않는다.

```
HTTP 요청
    ↓
FastAPI        ← 웹 프레임워크 (라우팅, 요청/응답)
    ↓
LangChain      ← LLM 오케스트레이션 프레임워크 (에이전트 루프, 툴 관리)
    ↓
Ollama         ← LLM 런타임
```

FastAPI는 "누가 왔고 어느 함수를 실행할지"를 담당하고, LangChain은 그 함수 안에서 LLM과 대화하는 복잡한 로직을 담당한다. `analyze_log()` 함수 하나로만 연결되어 있다.

실무에서 `FastAPI + LangChain`, `FastAPI + LlamaIndex`, `Django + LangChain` 조합은 표준적인 구성이다.

---

## 정리

에이전트 루프를 직접 구현하는 건 처음엔 단순해 보인다. 그런데 횟수 제한, 시간 제한, 루프 감지, 한계 도달 시 처리 같은 엣지 케이스들이 쌓이면 결국 프레임워크가 하는 일을 다시 만드는 셈이 된다.

LangChain의 `create_react_agent`는 이 반복되는 패턴을 추상화해놓은 것이다. 내부적으로 LangGraph의 그래프 실행 엔진 위에서 동작하며, 직접 구현의 유연함이 필요한 시점이 오기 전까지는 프레임워크에 맡기는 게 낫다.

한 가지 주의할 점은 LangChain의 버전 변화가 빠르다는 것이다. 처음 `AgentExecutor`로 구현했다가 langchain 1.x에서 해당 클래스가 제거되어 `create_react_agent`로 교체해야 했다. LLM 생태계는 아직 빠르게 변하고 있으니 버전 고정과 업그레이드 시점에 주의가 필요하다.
