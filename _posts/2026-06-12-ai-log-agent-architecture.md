---
layout: post
title: "AI 기반 서버 에러 자동 분석 시스템 — 전체 아키텍처와 핵심 기술"
date: 2026-06-12 13:00:00 +0900
categories: [AI, 프로젝트]
tags: [RAG, LLM, Agent, Graph, Memory, Judge, Gemini, ChromaDB, LangGraph, 아키텍처]
---

서버에서 에러가 발생하면 자동으로 원인을 분석하고, 코드 수정안을 만들어 GitHub PR까지 생성하는 AI 파이프라인을 구축했다. 이 포스트는 전체 시스템 구조와 각 단계에서 적용한 AI 기법들을 정리한다.

---

## 전체 파이프라인

```
서버 에러 발생
    ↓
Webhook 수신 (FastAPI)
    ↓
① Error Memory 검색 — 과거 유사 사례 조회
    ↓
② RAG 검색 — 관련 소스 파일 탐색
   (BM25 + 벡터 하이브리드 → RRF → Graph 확장)
    ↓
③ ReAct Agent 분석 (LangGraph + Ollama)
   — 과거 사례를 컨텍스트로 주입
   — search_files 툴로 소스 파일 직접 검색
    ↓
④ Self-reflection — before 코드 존재 여부 검증
    ↓
⑤ LLM Judge (Gemini) — 수정안 품질 자동 평가
    ↓
⑥ Slack 알림 — 분석 결과 + Judge 점수
    ↓
⑦ 승인 시 GitHub PR 자동 생성
```

---

## 시스템 구성도

```
┌─────────────────────────────────────────────────────┐
│                   서버 (Spring Boot 등)               │
│   에러 발생 → Webhook POST /api/v1/webhook/error      │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│              FastAPI Backend (Python)                 │
│                                                       │
│  ┌─────────────┐   ┌──────────────┐  ┌────────────┐ │
│  │ Error Memory│   │  RAG Engine  │  │ ReAct Agent│ │
│  │ ChromaDB    │   │  ChromaDB    │  │ LangGraph  │ │
│  │ (memory_*)  │   │  (server_*)  │  │ + Ollama   │ │
│  └─────────────┘   └──────────────┘  └────────────┘ │
│         │                │                  │         │
│         └────────────────┴──────────────────┘         │
│                          │                            │
│                  ┌───────▼───────┐                    │
│                  │  LLM Judge    │                    │
│                  │  Gemini API   │                    │
│                  └───────┬───────┘                    │
└──────────────────────────┼────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      Slack 알림      MySQL DB        GitHub PR
    (Judge 점수 포함)  (기록 저장)    (코드 수정 포함)
```

---

## 핵심 기술별 설명

### 1. 하이브리드 RAG — BM25 + 벡터 검색 + RRF

에러 스택 트레이스에는 `UserService`, `findById` 같은 **정확한 클래스명**이 포함된다. 벡터 검색만으로는 키워드 매칭이 약하고, BM25만으로는 의미 유사도를 못 잡는다.

두 검색 방식을 결합하고 **RRF(Reciprocal Rank Fusion)** 로 순위를 합산했다.

```python
# RRF 점수 계산 (k=60)
rrf[path] += 1 / (60 + rank + 1)
```

자세한 내용 → [BM25를 활용한 RAG 검색 품질 향상](/posts/bm25-hybrid-rag)

---

### 2. Graph RAG — import 기반 의존성 그래프

스택 트레이스에 `UserService`가 명시되어 있어도, 실제 원인은 `UserService`가 의존하는 `UserRepository`나 `DatabaseConfig`에 있을 수 있다.

인덱싱 시 언어별 import 구문을 파싱해 파일 간 의존성 그래프를 구성하고, 검색 결과를 그래프로 확장한다.

```
TestErrorController.java (검색 결과)
    ↓ Graph 확장 (depth=1)
EmailService.java, HealthCheckController.java (의존 파일 추가)
```

자세한 내용 → [Graph RAG — import 기반 의존성 그래프](/posts/graph-rag)

---

### 3. Error Memory — 과거 사례 재활용

분석 결과를 ChromaDB에 벡터로 저장해두고, 다음 유사 에러 발생 시 검색해 프롬프트에 주입한다.

```
[과거 사례 1]
에러: NullPointerException at UserService.findById
분석: value 필드 null 체크 누락, Optional 처리 필요
```

에러가 반복될수록 분석 품질이 향상되는 구조다.

자세한 내용 → [Error Memory — 과거 에러 분석 사례 재활용](/posts/error-memory)

---

### 4. ReAct Agent — LangGraph + Ollama

단순 LLM 호출이 아닌 **도구를 사용하는 에이전트** 구조다. `search_files` 툴을 통해 실제 소스 파일을 검색하고 내용을 확인한 뒤 수정안을 작성한다.

```
[에이전트 루프]
1. 스택 트레이스에서 클래스명 추출
2. search_files("TestErrorController") 호출
3. 파일 내용 확인
4. 수정 코드 작성
5. JSON 형태로 최종 응답
```

`recursion_limit=30`으로 최대 반복 횟수를 제한하고, 120초 타임아웃 후 structured output fallback으로 안정성을 보장한다.

---

### 5. LLM Judge — Gemini로 외부 검증

분석 에이전트(Ollama)가 생성한 수정안을 **외부 LLM(Gemini)이 독립적으로 검토**한다. 같은 모델이 자기 출력을 검토하는 self-evaluation bias를 피하기 위해 다른 모델을 Judge로 활용했다.

![Gemini Judge API](/assets/img/post/2026-06-12-ai-log-agent/judge-result.png)

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
  ├── 추천 수정 코드
  ├── Gemini Judge 점수
  └── [✅ 수락 & Push] [❌ 거절] 버튼

✅ 수락 클릭
  → 새 브랜치 생성 (log-agent/fix-record-{id})
  → 수정 파일 커밋
  → GitHub PR 자동 생성
```

에러 감지부터 PR 생성까지 사람의 개입은 Slack 버튼 클릭 한 번뿐이다.

---

## 기술 스택

| 영역 | 기술 |
|---|---|
| 백엔드 | FastAPI, Python 3.11 |
| LLM (분석) | Ollama — gemma4:12b |
| LLM (Judge) | Google Gemini API |
| 에이전트 | LangGraph (ReAct) |
| 벡터 DB | ChromaDB |
| 키워드 검색 | rank-bm25 |
| DB | MySQL (aiomysql + SQLAlchemy) |
| 배포 | Docker, Jenkins CI/CD |
| 알림 | Slack Webhook + Interactive Actions |
| 코드 관리 | GitHub API (자동 PR 생성) |

---

## 관련 포스트

### 구축기 시리즈
- [AI 오류 자동 분석 에이전트 구축기 (1) — 도입 배경과 초기 세팅](/posts/ai-log-agent-new-1)
- [AI 오류 자동 분석 에이전트 구축기 (2) — FastAPI와 파이프라인 설계](/posts/ai-log-agent-new-2)
- [AI 오류 자동 분석 에이전트 구축기 (3) — Embedding 모델과 ChromaDB RAG 구축](/posts/ai-log-agent-new-3)

### AI 고급 기법
- [BM25를 활용한 RAG 검색 품질 향상](/posts/bm25-hybrid-rag)
- [Graph RAG — import 기반 의존성 그래프로 검색 결과 확장하기](/posts/graph-rag)
- [Error Memory — 과거 에러 분석 사례를 RAG로 재활용하기](/posts/error-memory)
- [LLM Judge — 외부 LLM으로 AI 분석 결과를 자동 검증하기](/posts/llm-judge)
- [LangChain Agent 도입기](/posts/langchain-agent-introduction)
- [RAG 평가 모듈 구축 (Hit Rate, MRR)](/posts/rag-evaluation)
