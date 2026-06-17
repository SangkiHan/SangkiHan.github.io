---
layout: post
title: "AI 기반 서버 에러 자동 분석 시스템 — 전체 아키텍처와 핵심 기술"
date: 2026-06-12 13:00:00 +0900
categories: [AI, 프로젝트]
tags: [LLM, Agent, Graph, Memory, Judge, Gemini, ChromaDB, LangGraph, 아키텍처]
---

서버에서 에러가 발생하면 자동으로 원인을 분석하고, 코드 수정안을 만들어 GitHub PR까지 생성하는 AI 파이프라인을 구축했다. 이 포스트는 전체 시스템 구조와 각 단계에서 적용한 AI 기법들을 정리한다.

---

## 전체 파이프라인

```
서버 에러 발생
    ↓
Webhook 수신 (FastAPI)
    ↓
① Git fetch — 최신 소스코드 로컬 동기화
    ↓
② Error Memory 검색 — 과거 유사 사례 조회 (ChromaDB)
    ↓
③ ReAct Agent 분석 (LangGraph + Ollama)
   — 과거 사례를 컨텍스트로 주입
   — grep_files / read_file / list_directory 툴로 소스 파일 직접 탐색
    ↓
④ LLM Judge (Gemini) — 수정안 품질 자동 평가
    ↓
⑤ Slack 알림 — 분석 결과 + Judge 점수 + Before/After 코드
    ↓
⑥ 수락 클릭 → Gemini가 파일 수정 적용 → GitHub PR 생성
```

---

## 시스템 구성도

```
┌──────────────────────────────────────────────────────┐
│                  서버 (Spring Boot 등)                 │
│   에러 발생 → Webhook POST /api/v1/webhook/error       │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────────────┐
│  서버                                                                   │
│                                                                           │
│  ┌──────────────────────────────────────┐   ┌─────────────────────────┐ │
│  │      FastAPI Backend (Python)        │   │       Mac Mini          │ │
│  │                                      │   │                         │ │
│  │  ┌─────────────┐  ┌───────────────┐  │   │  ┌───────────────────┐  │ │
│  │  │ Error Memory│  │ 로컬 레포지토리│  │──►│  │  Ollama           │  │ │
│  │  │ ChromaDB    │  │ grep_files    │  │   │  │  gemma4:12b       │  │ │
│  │  │ (memory_*)  │  │ read_file     │  │◄──│  │  (LLM 추론)       │  │ │
│  │  └─────────────┘  └───────────────┘  │   │  └───────────────────┘  │ │
│  │                                      │   └─────────────────────────┘ │
│  │  LLM Judge (Gemini) · 파일 수정 (Gemini)                             │
│  └──────────────────────┬───────────────┘                               │
└─────────────────────────┼─────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
     Slack 알림       MySQL DB        GitHub PR
   (Judge 점수 포함)  (기록 저장)  (Before/After 포함)
```

---

## 핵심 기술별 설명

### 1. 로컬 파일 직접 탐색 — Claude Code 방식

초기에는 RAG(ChromaDB)로 소스코드를 청킹·임베딩해 벡터 검색으로 관련 파일을 찾았다. 그러나 운영 중 여러 한계를 마주했다.

- 청크 경계에서 코드가 잘려 컨텍스트가 유실됨
- 매 에러마다 전체 파일 인덱싱으로 오버헤드 발생
- `TestErrorController` 같은 정확한 식별자 검색에 벡터 유사도가 부적합

RAG를 제거하고 LLM에게 직접 파일을 탐색할 수 있는 도구를 제공했다.

```python
def _make_file_tools(repo_path: Path):
    @langchain_tool
    async def grep_files(pattern: str, path: str = "", file_extension: str = "") -> str:
        """레포지토리에서 클래스명·메서드명·패턴을 검색합니다."""
        ...

    @langchain_tool
    async def read_file(path: str) -> str:
        """파일 전체 내용을 읽습니다."""
        ...

    @langchain_tool
    async def list_directory(path: str = "") -> str:
        """디렉토리 구조를 탐색합니다."""
        ...
```

에이전트가 스택 트레이스에서 클래스명을 추출해 `grep_files`로 파일을 찾고, `read_file`로 실제 코드를 읽은 뒤 수정안을 작성한다. RAG 인덱싱 없이 항상 최신 코드를 본다.

자세한 내용 → [RAG 기반 코드 검색에서 로컬 파일 직접 조회로 전환](/posts/log-agent-rag-to-local-file)

---

### 2. Error Memory — 과거 사례 재활용

분석 결과를 ChromaDB에 벡터로 저장해두고, 다음 유사 에러 발생 시 검색해 프롬프트에 주입한다.

```
[과거 사례 1]
에러: NullPointerException at UserService.findById
분석: value 필드 null 체크 누락, Optional 처리 필요
```

에러가 반복될수록 분석 품질이 향상되는 구조다.

자세한 내용 → [Error Memory — 과거 에러 분석 사례 재활용](/posts/error-memory)

---

### 3. ReAct Agent — LangGraph + Ollama

단순 LLM 호출이 아닌 **도구를 사용하는 에이전트** 구조다. 파일 탐색 도구를 통해 실제 소스 파일을 직접 읽고 수정안을 작성한다.

```
[에이전트 루프]
1. 스택 트레이스에서 클래스명 추출
2. grep_files("TestErrorController") 호출
   → src/main/java/.../TestErrorController.java 발견
3. read_file("src/main/java/.../TestErrorController.java") 호출
   → 파일 전체 내용 확인
4. 수정 코드 작성 (before/after)
5. JSON 형태로 최종 응답
```

`recursion_limit=30`으로 최대 반복 횟수를 제한하고, 300초 타임아웃 후 structured output fallback으로 안정성을 보장한다.

---

### 4. LLM Judge — Gemini로 외부 검증

분석 에이전트(Ollama)가 생성한 수정안을 **외부 LLM(Gemini)이 독립적으로 검토**한다. 같은 모델이 자기 출력을 검토하는 self-evaluation bias를 피하기 위해 다른 모델을 Judge로 활용했다.

```
🤖 Gemini Judge
⭐⭐⭐⭐⭐ (5/5)  🟢 high
제시된 수정안은 NullPointerException의 근본 원인을 정확히 파악하고,
null 값에 대한 방어 로직을 추가하여 에러를 효과적으로 해결합니다.
```

자세한 내용 → [LLM Judge — 외부 LLM으로 AI 분석 결과 자동 검증](/posts/llm-judge)

---

## Slack → GitHub 자동화 워크플로우

```
Slack 메시지 수신
  ├── 에러 원인
  ├── 병목 지점
  ├── 추천 수정 코드 (Before / After)
  ├── Gemini Judge 점수
  └── [✅ 수락 & Push] [❌ 거절] 버튼

✅ 수락 클릭
  → 새 브랜치 생성 (log-agent/fix-record-{id})
  → Gemini가 로컬 파일을 읽고 Before→After 적용
  → 성공: 수정된 파일 커밋
  → 실패: 빈 커밋 (PR 본문에 Before/After 수동 적용 안내)
  → GitHub PR 생성 (Before/After 코드 항상 포함)
```

에러 감지부터 PR 생성까지 사람의 개입은 Slack 버튼 클릭 한 번뿐이다.

---

## 기술 스택

| 영역 | 기술 | 실행 환경 |
|---|---|---|
| 백엔드 | FastAPI, Python 3.11 | 서버 |
| LLM (분석) | Ollama — gemma4:12b | Mac Mini |
| LLM (Judge / 파일 수정) | Google Gemini API (gemini-3.1-flash-lite) | 서버 (API 호출) |
| 에이전트 | LangGraph (ReAct) | 서버 |
| 파일 탐색 | 로컬 파일시스템 직접 접근 (grep_files, read_file, list_directory) | 서버 |
| Error Memory | ChromaDB (벡터 DB) | 서버 |
| DB | MySQL (aiomysql + SQLAlchemy) | 서버 |
| 배포 | Docker, Jenkins CI/CD | 서버 |
| 알림 | Slack Webhook + Interactive Actions | 서버 |
| 코드 관리 | GitHub API (자동 PR 생성) | 서버 |

---

## 관련 포스트

### 구축기 시리즈
- [AI 오류 자동 분석 에이전트 구축기 (1) — 도입 배경과 초기 세팅](/posts/ai-log-agent-new-1)
- [AI 오류 자동 분석 에이전트 구축기 (2) — FastAPI와 파이프라인 설계](/posts/ai-log-agent-new-2)
- [AI 오류 자동 분석 에이전트 구축기 (3) — Embedding 모델과 ChromaDB RAG 구축](/posts/ai-log-agent-new-3)
- [AI 오류 자동 분석 에이전트 구축기 (4) — RAG 검색에서 로컬 파일 직접 탐색으로 전환](/posts/log-agent-rag-to-local-file)

### AI 고급 기법
- [RAG 기반 코드 검색에서 로컬 파일 직접 조회로 전환](/posts/log-agent-rag-to-local-file)
- [Error Memory — 과거 에러 분석 사례를 RAG로 재활용하기](/posts/error-memory)
- [LLM Judge — 외부 LLM으로 AI 분석 결과를 자동 검증하기](/posts/llm-judge)
- [LangChain Agent 도입기](/posts/langchain-agent-introduction)
