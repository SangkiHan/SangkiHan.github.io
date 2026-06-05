---
title: "AI Log Analysis Agent 구축기 (1) — 아키텍처 설계와 Event-Driven 파이프라인"
date: 2026-06-05 00:00:00 +0900
categories: [Project, DevSecOps]
tags: [fastapi, spring-boot, ollama, slack, devops, ai, python]
---

## 개요

운영 중인 서비스에서 에러가 발생하면 개발자가 직접 로그를 확인하고 원인을 분석해야 한다.
이 과정을 자동화하기 위해 **AI Log Analysis Agent**를 구축했다.

에러 발생 → 자동 감지 → LLM 분석 → Slack 알림 → 개발자 승인 의 흐름을 전부 자동화한다.

---

## 시스템 구성

| 컴포넌트 | 역할 |
|---|---|
| **puppynote-server** (Spring Boot) | 에러 감지 & 웹훅 발송 |
| **log-agent-backend** (FastAPI) | 웹훅 수신, 분석 파이프라인 실행 |
| **Ollama gemma4:12b** | 에러 원인 분석 & 수정 제안 |
| **Slack** | 분석 결과 알림, 수락/거절 버튼 |
| **GitHub** | 소스코드 저장소 (분석 컨텍스트 제공) |
| **MySQL** | 서버 정보, 분석 이력 저장 |

---

## 전체 흐름 (Event-Driven)

```
[puppynote-server] 예외 발생
        ↓ POST /api/v1/webhook/error
[log-agent-backend] 웹훅 수신 → 200 OK 즉시 반환
        ↓ BackgroundTask
[Git Service] fetch → 원격 브랜치 HEAD 커밋 조회 → 스택 트레이스에서 소스파일 추출
        ↓
[Ollama] gemma4:12b → 원인 분석 + 수정 제안 JSON 반환
        ↓
[MySQL] AnalysisRecord 저장
        ↓
[Slack] 분석 결과 + [✅ 수락] [❌ 거절] 버튼 전송
```

핵심 설계 원칙: **웹훅 수신과 분석 파이프라인을 분리**한다.
웹훅 엔드포인트는 즉시 200을 반환하고, 분석은 `BackgroundTask`로 비동기 실행한다.
이렇게 하면 Ollama가 느려도 Spring Boot 쪽에 타임아웃이 발생하지 않는다.

---

## 설계 결정 — git 로직은 log-agent가 전담

초기에는 Spring Boot 쪽에서 commit hash를 직접 포함해 보낼까 고민했다.
하지만 그렇게 하면 **모든 연동 서비스에 git 로직이 들어가야 한다**는 문제가 생긴다.

최종 결정: **외부 서비스는 에러 페이로드만 보내고, git은 log-agent가 전담**한다.

```
# 외부 서비스가 보내는 것 (단순)
{
  "server_name": "puppynote",
  "server_ip": "3.39.16.28",
  "error_type": "NullPointerException",
  "message": "Cannot invoke String.toUpperCase() because value is null",
  "stack_trace": "...",
  "request_method": "GET",
  "request_url": "/test/npe"
}
```

log-agent는 server_ip로 DB에서 서버를 조회하고, 해당 서버의 git 정보를 이용해 소스를 가져온다.

---

## 서버 등록 구조

서버별로 git 저장소와 GitHub 토큰을 관리한다.
멀티 계정을 지원하기 위해 **GITHUB_TOKEN은 전역 환경변수가 아닌 서버별로 저장**한다.

```python
class Server(Base):
    __tablename__ = "servers"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    git_repo_url: Mapped[str] = mapped_column(String(500), nullable=False)
    git_branch: Mapped[str] = mapped_column(String(100), default="main")
    github_token: Mapped[str] = mapped_column(String(200), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    hosts: Mapped[list["ServerHost"]] = relationship(back_populates="server", cascade="all, delete-orphan")
```

`ServerHost` 테이블에 서버의 IP를 저장해두고, 웹훅이 들어오면 `server_ip`로 매핑한다.

---

## 다음 글

- [2편] Spring Boot 글로벌 예외 핸들러 & 웹훅 연동
- [3편] FastAPI 파이프라인 & Git 서비스 핵심 로직
- [4편] Ollama LLM 연동 & Slack 알림
