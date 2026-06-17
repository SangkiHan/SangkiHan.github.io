---
layout: post
title: "Slack으로 부리는 개인 비서 — FastAPI + LangGraph로 만든 AI 에이전트"
date: 2026-05-28 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Slack, FastAPI, LangGraph, LLM, Ollama, Python, PersonalAssistant]
---

log-agent를 운영하면서 "에러 분석 에이전트를 만드는 데 쓴 구조를 일상에도 쓸 수 있지 않을까"는 생각이 들었다. 일정 관리, 지출 기록, 할 일 목록, 리마인더 같은 것들을 하나의 Slack 채널에서 말로 처리하는 개인 비서다.

이 포스트는 전체 구축 과정을 한눈에 볼 수 있는 인덱스다. 각 주제별 상세 글로 이동할 수 있다.

---

## 전체 구성

| 기능 | 수단 |
|---|---|
| 일정 추가·조회 | Google Calendar API |
| 지출 기록·조회 | Google Sheets API |
| 할 일 추가·완료 | MySQL |
| 리마인더 설정·취소 | APScheduler + MySQL |
| 장기 기억 | ChromaDB + LlamaIndex |
| 웹 검색 | DuckDuckGo |
| 파일 저장·검색·카테고리 관리 | 로컬 디스크 + ChromaDB + Slack Files API |

"다음주 화요일 오후 2시 치과 예약해줘", "스타벅스 6500원", "30분 후에 약 먹으라고 알려줘" — 이 세 문장이 각각 Calendar, Sheets, APScheduler로 연결된다.

---

## 아키텍처

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
    ├── ChromaDB + LlamaIndex (장기 기억 + Knowledge Graph)
    ├── MySQL (Todo / Reminder)
    └── DuckDuckGo (웹 검색)
    ↓
Slack DM 응답
```

핵심은 LangGraph의 ReAct 에이전트다. 사용자 메시지를 받으면 어떤 도구를 쓸지 LLM이 판단하고, 도구를 실행한 결과를 다시 LLM에게 넘겨 최종 답변을 만든다.

---

## 구축기 시리즈

### 1. 기초 구조 — FastAPI + LangGraph + Slack

→ **[현재 포스트]**

Slack Events API 연동, LangGraph ReAct 에이전트 구조, 도구 정의 방식, 시스템 프롬프트 설계를 다룬다.

주요 포인트:
- Slack은 3초 안에 200 응답을 요구 → `BackgroundTasks`로 즉시 반환 후 백그라운드 처리
- `bot_id` 체크 없으면 봇 응답에도 이벤트가 발생해 무한루프
- `@langchain_tool` 데코레이터로 도구 정의, `user_id`는 클로저로 바인딩
- Slack은 표준 마크다운이 아닌 mrkdwn 형식 — 시스템 프롬프트에 명시 필수

```python
@router.post("/events")
async def slack_events(request: Request, background_tasks: BackgroundTasks):
    body = await request.json()
    event = body.get("event", {})
    if event.get("type") == "message" and not event.get("bot_id"):
        background_tasks.add_task(handle_message, user_id, text, channel)
    return {"ok": True}
```

---

### 2. 장기 기억 — ChromaDB + Ollama 임베딩

→ **[개인 비서에 장기 기억 달기](/posts/personal-assistant-memory)**

ChromaDB를 이용한 벡터 기반 장기 기억 구현을 다룬다.

주요 포인트:
- 임베딩: Ollama `nomic-embed-text` 모델, `/api/embed` 엔드포인트 직접 호출
- ChromaDB HttpClient를 싱글턴으로 관리 — 매 요청마다 생성 시 `ResourceWarning` 발생
- `min(n, count)` 없으면 저장된 문서보다 많은 n_results 요청 시 ChromaDB 에러
- `_prefetch_memory()`: 매 대화 전에 자동 검색 후 메시지 앞에 주입 — LLM이 도구를 호출할지 말지 판단하는 것보다 확실함
- Docker 컨테이너 간 통신: `host.docker.internal` 대신 같은 네트워크에 붙여서 컨테이너 이름으로 접근

```python
async def _prefetch_memory(user_id: str, message: str) -> str:
    memories = await memory_service.search_memory(user_id, message)
    if not memories:
        return message
    context = "\n".join(f"- {m}" for m in memories)
    return f"[기억된 정보 (자동 조회)]\n{context}\n\n[사용자 메시지]\n{message}"
```

---

### 3. Google Calendar·Sheets 연동

→ **[개인 비서에 Google Calendar·Sheets 연동하기](/posts/personal-assistant-google)**

서비스 계정 방식의 Google API 연동과 삽질 기록이다.

주요 포인트:
- 서비스 계정 선택 이유: 서버 자동화에서 OAuth2 토큰 갱신 관리가 불필요
- `calendarId="primary"` → 서비스 계정 자체 캘린더에 이벤트 생성, 내 캘린더에 접근하려면 Gmail 주소 지정 필요
- Groovy 단일 따옴표 문자열에서 `\n`은 리터럴 → `json.loads(raw, strict=False)` 로 파싱
- Google API 클라이언트는 동기 라이브러리 → `run_in_executor`로 스레드풀에서 실행

```python
@langchain_tool
async def add_calendar_event(title: str, date: str, time: str) -> str:
    loop = asyncio.get_event_loop()
    return await asyncio.wait_for(
        loop.run_in_executor(None, lambda: calendar_service.add_event(title, date, time)),
        timeout=45,
    )
```

---

### 4. 할 일 목록과 리마인더

→ **[개인 비서 — 할 일 목록과 리마인더 구현](/posts/personal-assistant-todo-reminder)**

APScheduler 기반 리마인더와 Todo CRUD 구현을 다룬다.

주요 포인트:
- Calendar(날짜 이벤트) vs Todo(완료 처리 작업) vs Reminder(시간 기반 알림) — 세 가지 구분이 중요
- `fired=True`로 취소 처리 — 삭제하면 "아까 설정한 리마인더 취소해줘" 요청 시 목록 제공 불가
- `[AGENT_ONLY]` 섹션: 에이전트에게 DB PK를 전달하되 사용자 화면에는 미노출
- `recursion_limit=6`에서 리마인더 수정(list→cancel→set, 3번 연속 도구 호출) 시 한도 초과 → 20으로 조정
- 1분 polling 방식 — 개인 비서 용도로 충분한 정밀도

```python
def start_scheduler() -> None:
    _scheduler = AsyncIOScheduler(timezone="Asia/Seoul")
    _scheduler.add_job(_fire_reminders, "interval", minutes=1)
    _scheduler.start()
```

---

### 5. LLM 전략 — Gemini 우선, Ollama 폴백

→ **[개인 비서 LLM 전략 — Gemini 우선, Ollama 폴백](/posts/personal-assistant-llm-strategy)**

LLM을 Ollama에서 Gemini로 바꾼 이유와 폴백 전략을 다룬다.

주요 포인트:
- Ollama `gemma4:12b` 응답: 20~30초 / Gemini: 2~3초 — 개인 비서에서 30초는 너무 느림
- `gemini-2.5-flash-lite`는 무료 20회/일, `gemini-2.0-flash`는 1,500회/일 — 최신 모델이 항상 관대하지 않음
- Gemini로 바꾼 뒤 도구 호출 누락 → 시스템 프롬프트를 표 형식으로 강화, "도구 없이 텍스트만 답변하지 마세요" 추가
- `think=False`: Ollama 일부 모델의 내부 추론 출력 제거

```python
def _is_quota_error(e: Exception) -> bool:
    s = str(e).lower()
    return any(k in s for k in ("quota", "429", "resource exhausted", "rate limit"))

async def chat(user_id: str, message: str) -> str:
    try:
        result = await _invoke_graph(_make_gemini(), tools, message, timeout=120)
    except Exception as e:
        if _is_quota_error(e):
            result = await _invoke_graph(_make_ollama(), tools, message, timeout=300)
```

---

### 6. 기억 시스템 디버깅

→ **[개인 비서 기억 시스템 디버깅 — 고유명사 검색 실패, 데이터 오염, 확인 UX](/posts/personal-assistant-memory-debug)**

운영하면서 터진 기억 시스템 버그들의 디버깅 기록이다.

주요 포인트:
- "또리" 같은 고유명사는 임베딩 모델이 의미를 학습하지 못해 벡터 유사도 낮음 → BM25 + 벡터 RRF 하이브리드로 해결
- `auto_save_memory` 프롬프트 끝에 `핵심 정보 (없으면 빈 문자열):`를 붙이자 LLM이 그대로 복사 → ChromaDB 오염
- "(내용 없음)" 8글자 → `len > 5` 필터를 통과해 저장 → 쓰레기 데이터가 top-3 점령
- 오염 데이터로 에이전트가 기억을 불신 → 시스템 프롬프트에 "신뢰하고 '없다'고 하지 마" 추가
- Slack Block Kit으로 저장 전 사용자 확인 UX 구현

```
루트 원인 체인:
auto_save_memory 프롬프트 잘못됨
    → 쓰레기 데이터 ChromaDB 저장
    → top-3 점령
    → 실제 데이터 밀려남
    → 에이전트가 "없다"고 답변
```

---

### 7. BM25 한국어 복합어 문제

→ **[개인 비서 BM25 한국어 복합어 문제 — 형태소 분석기 없이 해결하기](/posts/personal-assistant-korean-bm25)**

"내차가"를 "차"로 매칭하지 못하는 BM25 한국어 문제와 해결법을 다룬다.

주요 포인트:
- `[가-힣]+` 패턴은 "내차가"를 하나의 토큰으로 → "차" 매칭 불가
- 형태소 분석기(`kiwipiepy`) 없이 해결: 2-gram + 음절 단위 토큰 추가
- "내차가" → ["내차가", "내차", "차가", "내", "차", "가"] → "차" 매칭 성공
- BM25 IDF가 "이", "의", "는" 같은 공통 음절의 가중치를 자동으로 낮춤
- `kiwipiepy` 미도입 이유: Docker 이미지 크기 증가, arm64/amd64 호환성 문제

```python
def _tokenize(text: str) -> list[str]:
    tokens = re.findall(r"[A-Za-z가-힣0-9]+", text.lower())
    korean_words = re.findall(r"[가-힣]+", text)
    extra = []
    for word in korean_words:
        for i in range(len(word) - 1):
            extra.append(word[i:i+2])   # 2-gram
        extra.extend(list(word))         # 음절
    return tokens + extra
```

---

### 8. LlamaIndex 도입 — 기억 검색 + Knowledge Graph

→ **[AI 개인 비서에 LlamaIndex 도입 — 기억 검색부터 Knowledge Graph까지](/posts/personal-assistant-llamaindex)**

memory_service를 LlamaIndex로 교체하고, 대화 히스토리 압축과 Knowledge Graph를 추가한 기록이다.

주요 포인트:
- `OllamaEmbedding` + `VectorStoreIndex` + `ChromaVectorStore`로 수동 httpx 임베딩 호출 대체
- `QueryFusionRetriever(mode="reciprocal_rerank")`로 BM25 + 벡터 RRF를 한 줄로
- 대화 히스토리 압축: 5턴 초과 시 오래된 대화를 LLM으로 요약, 최근 3턴 원문 유지
- `SimplePropertyGraphStore`: "또리 생일" → `(또리) -[생일]→ (10월 27일)` 그래프 저장
- `query_knowledge_graph` 도구로 엔티티 중심 쿼리 지원

```python
retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=n,
    mode="reciprocal_rerank",
    use_async=True,
)
nodes = await retriever.aretrieve(query)
```

---

### 9. Slack 파일 저장소 — 업로드·카테고리·번들·시맨틱 검색

→ **[개인 비서에 Slack 파일 저장소 추가](/posts/personal-assistant-file-storage)**

Slack DM으로 보낸 파일을 저장하고, 나중에 말로 찾아오는 기능을 구현한 기록이다.

주요 포인트:
- Slack `url_private`는 bot token 인증 없이 접근하면 403 — `files:read` 스코프 필수
- 파일 내용이 아닌 **파일명 + 카테고리 + 날짜**를 ChromaDB에 임베딩 — 이진 파일 파싱 불필요
- "회의록 파일 줘"처럼 대략적으로 말해도 벡터 유사도로 찾아 전송
- 파일과 함께 보낸 짧은 텍스트(15자 이하)를 카테고리로 자동 인식 ("업무" → category=업무)
- 여러 파일 동시 전송 → FileBundle로 묶어 저장, 카테고리 조회 시 zip으로 일괄 전송
- Docker 볼륨 미설정 시 컨테이너 재시작에 파일 소실 — `docker-compose.yml` 볼륨 마운트 필수

```python
def _make_embed_text(filename, mimetype, dt, category):
    parts = [f"파일명: {filename}", f"종류: {mimetype}", f"날짜: {dt.strftime('%Y-%m-%d')}"]
    if category:
        parts.insert(1, f"카테고리: {category}")
    return " | ".join(parts)
```

---

## 기술 스택

| 영역 | 기술 |
|---|---|
| 백엔드 | FastAPI, Python 3.11 |
| LLM (기본) | Google Gemini API (gemini-2.0-flash) |
| LLM (폴백) | Ollama — gemma4:12b |
| 에이전트 | LangGraph (ReAct) |
| 기억 저장소 | ChromaDB + LlamaIndex VectorStoreIndex |
| Knowledge Graph | LlamaIndex SimplePropertyGraphStore |
| DB | MySQL (aiomysql + SQLAlchemy) |
| 스케줄러 | APScheduler |
| 외부 연동 | Google Calendar API, Google Sheets API |
| 파일 저장 | 로컬 디스크 + ChromaDB (메타데이터 임베딩) |
| 알림 | Slack SDK + Block Kit |

---

## 전체 데이터 흐름

```
사용자 Slack DM
    ↓
FastAPI /events (즉시 200 반환 + BackgroundTask)
    ↓
_prefetch_memory() — ChromaDB 자동 검색 후 메시지 앞에 주입
    ↓
_get_history_context() — 대화 히스토리 (5턴 초과 시 LLM 압축)
    ↓
LangGraph ReAct 에이전트 (Gemini → 할당량 초과 시 Ollama)
    ├── search_memory → ChromaDB 하이브리드 검색 (벡터 + BM25 RRF)
    ├── query_knowledge_graph → SimplePropertyGraphStore 엔티티 조회
    ├── save_memory → Slack Block Kit 확인 버튼 → ChromaDB + Knowledge Graph 저장
    ├── add_calendar_event → Google Calendar API
    ├── add_expense → Google Sheets API
    ├── set_reminder → MySQL + APScheduler
    ├── add_todo / complete_todo → MySQL
    ├── web_search → DuckDuckGo
    └── find_file / find_files_by_category → 로컬 디스크 + ChromaDB + Slack Files API
    ↓
Slack DM 응답
```
