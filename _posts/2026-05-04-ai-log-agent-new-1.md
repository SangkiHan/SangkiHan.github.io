---
layout: post
title: "AI 오류 자동 분석 에이전트 구축기 (1) — 도입 배경과 초기 세팅"
date: 2026-05-04 10:00:00 +0900
categories: [Project, AI]
tags: [FastAPI, Ollama, Slack, LLM, Python, DevOps]
---

개인 프로젝트 puppynote-server를 운영하면서 반복적으로 겪는 불편함이 있었다. 에러가 발생해도 즉시 확인하기 어렵다는 것이다.

## 왜 만들었나

개인 스터디 프로젝트는 24시간 모니터링하는 팀이 없다. 배포하고 자리를 뜨면 에러가 났는지조차 모른다. Slack에 에러 알림이 와도 "나중에 봐야지"가 반복된다. 원인 파악에 시간이 걸리고, 이미 컨텍스트가 날아간 상태라 스택 트레이스만 보고 빠르게 수정하기 어렵다.

**해결하고 싶었던 것**: 에러가 발생하면 Slack에 단순 알림이 아니라 **원인 분석 + 수정 코드 + GitHub PR**까지 자동으로 만들어주는 시스템.

---

## 하드웨어

로컬에서 LLM을 직접 돌린다. 외부 API(OpenAI 등)는 비용이 발생하고 코드가 외부로 나가는 게 불편했다.

| 항목 | 사양 |
|---|---|
| 기기 | Apple Mac Mini M4 |
| RAM | 24GB (통합 메모리) |
| LLM 런타임 | Ollama |
| 분석 모델 | gemma4:12b |
| 임베딩 모델 | nomic-embed-text |

M4의 Neural Engine이 LLM 추론을 처리한다. 24GB 통합 메모리 덕분에 12B 파라미터 모델을 빠르게 돌릴 수 있다.

---

## 전체 흐름

```
puppynote-server (Spring Boot)
    │ 에러 발생 → ControllerAdvice 감지
    │ POST /api/v1/webhook/error
    ▼
log-agent-backend (FastAPI, 개인 노트북 서버 4Core 8G)
    │ 웹훅 수신 → BackgroundTask로 파이프라인 시작
    │ git clone → RAG 인덱싱 → Ollama 분석
    ▼
Ollama (Mac Mini M4 24G)
    │ gemma4:12b로 에러 분석
    │ search_files 도구로 소스 파일 검색
    ▼
ChromaDB + nomic-embed-text
    │ 소스코드 벡터 DB
    │ 의미 기반 파일 검색
    ▼
Slack
    │ 원인 / 수정 코드 / 수락·거절 버튼
    ▼
GitHub API (수락 시)
    브랜치 생성 → 파일 수정 → PR 자동 생성
```

---

## 사용 모델과 역할

**gemma4:12b** — 에러 로그를 받아 원인을 분석하고 수정 코드를 JSON으로 반환한다. tool calling을 지원해서 필요한 소스 파일을 직접 검색해가며 분석한다.

**nomic-embed-text** — 소스 파일을 벡터로 변환해서 ChromaDB에 저장한다. 텍스트를 숫자 배열로 변환하는 임베딩 전용 모델로, 분석 LLM과는 별개로 동작한다.

---

## Slack 웹훅 설정

분석 결과는 Slack Incoming Webhook으로 전송한다. 단순 텍스트가 아니라 Block Kit으로 구성해서 수락/거절 버튼을 함께 표시한다.

```python
# slack_service.py 핵심
{
    "type": "actions",
    "elements": [
        {
            "type": "button",
            "text": {"type": "plain_text", "text": "✅ 수락 & Push"},
            "style": "primary",
            "action_id": "approve_fix",
            "value": str(record.id),   # DB record ID
        },
        {
            "type": "button",
            "text": {"type": "plain_text", "text": "❌ 거절"},
            "style": "danger",
            "action_id": "reject_fix",
            "value": str(record.id),
        },
    ],
}
```

버튼 클릭은 Slack이 FastAPI `/api/v1/slack/actions` 엔드포인트로 POST 요청을 보내는 방식이다. 링크 버튼(브라우저가 열리는 방식)이 아니라 Slack 내부에서 처리된다.

---

## 통신 흐름 상세

```
[Before — 에러 발생 시]

puppynote-server                  개발자
     │  에러 발생                    │
     │──────────────────────────────▶│  Slack 단순 알림
     │                               │  "나중에 봐야지..."
     │                               │  (컨텍스트 전환 비용 높음)

[After — 에이전트 도입 후]

puppynote-server      FastAPI        Ollama(M4)      Slack
     │ 에러 발생          │               │              │
     │──POST webhook──▶  │               │              │
     │                   │──git clone──▶ 로컬           │
     │                   │──embed──────▶ nomic          │
     │                   │──analyze────▶ gemma4:12b     │
     │                   │               │ search_files │
     │                   │◀─────────────  JSON 응답     │
     │                   │──────────────────────────────▶│
     │                   │                              │ 원인+수정코드+버튼
     │                   │                         개발자가 수락 클릭
     │                   │◀──────── POST /slack/actions──│
     │                   │──GitHub PR 생성               │
     │                   │──────────────────────────────▶│ PR URL 전송
```

개발자는 Slack에서 수락 버튼 하나만 누르면 된다.
