---
title: "AI Log Analysis Agent 구축기 (5) — Docker & Jenkins CI/CD 배포"
date: 2026-06-05 04:00:00 +0900
categories: [Project, DevSecOps]
tags: [docker, jenkins, cicd, python, fastapi]
---

## Docker 구성

`python:3.11-slim-bookworm` + `uv` 패키지 매니저를 사용한다.
`uv`는 pip보다 10~100배 빠른 Python 패키지 설치 도구다.

```dockerfile
FROM python:3.11-slim-bookworm

WORKDIR /app
ENV PYTHONUNBUFFERED=1

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY app/ ./app/

EXPOSE 8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`--frozen` 플래그로 lock 파일 기준으로만 설치한다. 재현 가능한 빌드를 보장한다.

**주목할 점**: `apt-get`이 없다. git 바이너리를 설치하지 않는다.
Docker 빌드 환경에서 HTTP 포트 80이 차단되어 apt-get으로 패키지를 설치할 수 없었다.
`dulwich` (순수 Python git 구현체)를 도입해서 시스템 git 의존성을 완전히 제거했다.

---

## Jenkins Pipeline

```groovy
pipeline {
    agent any
    environment {
        IMAGE_NAME = "log-agent-backend"
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Learning-LLM/learning-server.git'
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
                }
            }
        }
        stage('Deploy') {
            steps {
                sh "docker stop ${IMAGE_NAME} || true"
                sh "docker rm ${IMAGE_NAME} || true"
                sh """
                    docker run -d --name ${IMAGE_NAME} \
                      --restart always \
                      -p 6001:8000 \
                      -v /var/lib/jenkins/repos:/app/repos \
                      -e DATABASE_URL=mysql+aiomysql://... \
                      -e OLLAMA_HOST=http://210.183.11.113:11434 \
                      -e SLACK_WEBHOOK_URL=... \
                      -e PUBLIC_URL=https://log-agent-api.sangkihan.co.kr \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }
        stage('Health Check') {
            steps {
                retry(5) {
                    sleep(3)
                    sh 'curl -sf http://localhost:6001/health | grep status'
                }
            }
        }
    }
    post {
        failure {
            sh """curl -s -X POST ${SLACK_WEBHOOK_URL} \
              -d '{"text": "❌ log-agent-backend 배포 실패"}'"""
        }
    }
}
```

`-v /var/lib/jenkins/repos:/app/repos` 볼륨 마운트로 git clone 데이터를 컨테이너 재시작 후에도 유지한다.

---

## 트러블슈팅 기록

### 1. apt-get 404 오류

Jenkins 빌드 서버에서 HTTP(80포트)가 차단되어 `deb.debian.org`와 `mirror.kakao.com` 모두 실패했다.

```
Err: http://deb.debian.org/debian bookworm Release
  404 Not Found [IP: 146.75.50.132 80]
```

**해결**: git 바이너리 설치를 포기하고 `dulwich` (순수 Python)로 대체.

### 2. dulwich SHA 이중 인코딩

```python
# 잘못된 방식
sha_bytes = repo.refs[ref_key]
return sha_bytes.hex()   # 80자 이중 인코딩 발생

# 올바른 방식
sha = repo.refs[ref_key]
return sha.decode("ascii")  # dulwich는 이미 hex 문자열을 bytes로 반환
```

dulwich의 `repo.refs[key]`는 20바이트 바이너리가 아니라 **40자 hex 문자열을 ASCII bytes**로 반환한다.
`.hex()`를 쓰면 이를 다시 hex로 변환해서 80자가 된다.

### 3. Ollama thinking 노출

```
# format: "json" 옵션 사용 시
{"thought": "The user wants..."}<|tool_response>
```

gemma4 모델에서 `format: "json"` 옵션이 thinking 과정을 노출시켰다.
이 옵션을 제거하고 시스템 프롬프트로 JSON 형식을 강제하는 방식으로 해결했다.

---

## 환경변수 구성

```
OLLAMA_HOST=http://210.183.11.113:11434
OLLAMA_MODEL=gemma4:12b
DATABASE_URL=mysql+aiomysql://root:***@210.183.11.113:3306/log-agent
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
REPOS_PATH=/app/repos
PUBLIC_URL=https://log-agent-api.sangkihan.co.kr
FRONTEND_URL=https://log-agent.sangkihan.co.kr
```

---

## 현재까지 구현된 기능 요약

```
✅ Spring Boot 글로벌 예외 핸들러 → 웹훅 발송
✅ FastAPI 웹훅 수신 → 즉시 200 반환 + BackgroundTask
✅ 중복 에러 60초 내 dedup
✅ server_ip 기반 서버 자동 매핑
✅ dulwich로 git clone/fetch (시스템 git 불필요)
✅ 스택 트레이스 파싱 → 소스 파일 추출
✅ Ollama gemma4:12b LLM 분석
✅ Slack Block Kit 알림 + 수락/거절 버튼
✅ 분석 이력 DB 저장
✅ React 프론트엔드 (서버 CRUD, 분석 이력 조회)
✅ Jenkins CI/CD + Docker 자동 배포
```

## 다음 단계

현재는 스택 트레이스에 명시된 파일만 LLM 컨텍스트로 전달한다.
다음 단계로 **RAG (Retrieval-Augmented Generation)** 를 도입해서 전체 소스코드를 벡터 DB로 인덱싱하고, 에러와 의미적으로 가장 관련된 파일을 자동 선별할 예정이다.
