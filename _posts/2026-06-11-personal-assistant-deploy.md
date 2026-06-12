---
layout: post
title: "개인 비서 Docker 배포 & 운영 중 마주친 문제들"
date: 2026-06-11 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Docker, Jenkins, ChromaDB, Slack, Debugging, Python, DevOps]
---

기능 구현보다 배포와 운영에서 더 많은 시간을 썼다. 로컬에서 잘 되던 게 Docker 환경에서 안 되는 일이 반복됐다. 마주친 문제들을 정리했다.

---

## Jenkins CI/CD 파이프라인

```groovy
pipeline {
    environment {
        APP_NAME = 'personal-assistant'
        APP_PORT = '6002'
    }

    stages {
        stage('Set Environment Variables') {
            steps {
                script {
                    env.OLLAMA_HOST = 'http://192.168.x.x:11434'
                    env.GEMINI_API_KEY = '...'
                    env.GOOGLE_CREDENTIALS_JSON = '{ "type": "service_account", ... }'
                    // ...
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${env.APP_NAME}:${env.IMAGE_TAG} ."
            }
        }

        stage('Deploy') {
            steps {
                // credentials.json을 파일로 저장
                sh "echo '${env.GOOGLE_CREDENTIALS_JSON}' > ${env.DEPLOY_PATH}/credentials.json"
                sh """
                    cat > ${env.DEPLOY_PATH}/.env << EOF
GEMINI_API_KEY=${env.GEMINI_API_KEY}
...
EOF
                """
                sh "docker compose -f ${env.DEPLOY_PATH}/docker-compose.yml up -d"
            }
        }

        stage('Health Check') {
            steps {
                script {
                    retry(10) {
                        sleep 6
                        sh "curl -sf http://localhost:${env.APP_PORT}/health | grep 'status'"
                    }
                }
            }
        }
    }
}
```

Google 서비스 계정 JSON을 환경변수로 넣고 파일로 내려쓰는 이유가 있다. `.env` 파일에 멀티라인 값을 넣으면 파싱이 복잡해진다. JSON을 파일로 분리하고 Docker volume으로 마운트하면 훨씬 깔끔하다.

```yaml
# docker-compose.yml
volumes:
  - /var/lib/jenkins/personal-assistant/credentials.json:/app/credentials.json:ro
```

---

## ChromaDB Docker 네트워크 문제

ChromaDB를 `log-agent-backend` 프로젝트에서 이미 띄워두고 있었다. `personal-assistant` 컨테이너에서 이 ChromaDB에 접근하려면 같은 Docker 네트워크여야 한다.

처음 시도: `CHROMA_HOST=host.docker.internal`

```
Connection refused
```

`host.docker.internal`은 컨테이너에서 호스트 머신에 접근하는 특수 DNS다. Linux Docker에서는 기본적으로 동작하지 않는다. Mac에서는 잘 되는데 Linux에서 안 되는 이유가 이것이었다.

두 번째 시도: 호스트 머신 IP를 직접 지정

```
Connection timed out
```

Docker의 published port(`-p 8000:8000`)는 외부에서 호스트 IP로 접근하는 용도다. 컨테이너 내부에서 호스트 IP로 다시 들어오는 경로는 bridge 네트워크 구성에 따라 막힐 수 있다.

해결: 두 프로젝트의 컨테이너를 같은 Docker 네트워크에 붙이기

```yaml
# personal-assistant docker-compose.yml
networks:
  default:
  log-agent-net:
    name: log-agent_default
    external: true
```

`docker inspect chromadb`로 해당 컨테이너가 붙어있는 네트워크 이름을 확인했다.

```json
"Networks": {
    "log-agent_default": { ... }
}
```

이 네트워크를 `external: true`로 참조하면 같은 네트워크에 합류한다. 이후 `CHROMA_HOST=chromadb`(컨테이너 이름)으로 직접 접근 가능해졌다.

---

## Slack 봇 메시지 루프

봇이 보낸 메시지도 Slack Events API로 다시 들어온다. 처리하지 않으면 봇이 자신의 응답에 또 응답하는 무한루프가 된다.

```python
event = body.get("event", {})

# 봇 메시지 필터링
if event.get("bot_id"):
    return {"ok": True}

# 본인만 허용 (보안)
if event.get("user") != settings.slack_my_user_id:
    return {"ok": True}
```

`bot_id`가 있으면 봇이 보낸 메시지다. 바로 `{"ok": True}`를 반환해서 처리하지 않는다.

`slack_my_user_id` 체크는 보안 목적이다. 봇을 Slack 워크스페이스에 설치하면 워크스페이스 멤버 누구나 DM을 보낼 수 있다. 자신의 Slack User ID만 허용하도록 필터링했다.

---

## 응답이 비어서 Slack 에러

```
slack_sdk.errors.SlackApiError: The request to the Slack API failed.
(url: https://slack.com/api/chat.postMessage)
The server responded with: {'ok': False, 'error': 'no_text'}
```

에이전트 응답을 추출하는 코드가 단순했다.

```python
reply = result["messages"][-1].content
```

Gemini는 `content`가 문자열이 아니라 리스트로 올 때가 있다.

```python
# content가 list인 경우
[{"type": "text", "text": "실제 응답 텍스트"}]
```

그냥 `content`를 Slack에 넘기면 `[{'type': 'text', ...}]` 문자열이 전달되거나 빈 문자열이 된다.

```python
def _extract_text(content) -> str:
    if isinstance(content, list):
        return " ".join(
            b["text"] for b in content
            if isinstance(b, dict) and b.get("type") == "text"
        )
    return content or ""

reply = _extract_text(result["messages"][-1].content)
if not reply.strip():
    # 마지막 메시지가 비어있으면 이전 메시지에서 찾기
    for msg in reversed(result["messages"][:-1]):
        reply = _extract_text(msg.content)
        if reply.strip():
            break
    else:
        reply = "처리가 완료되었습니다."
```

마지막 메시지가 비어있는 경우도 처리했다. 에이전트 그래프의 마지막 스텝이 도구 실행으로 끝났을 때 최종 AIMessage가 없는 경우가 있다. 역순으로 스캔해서 내용 있는 메시지를 찾는다.

---

## 리소스 경고 억제

운영 로그에 반복적으로 나타나는 경고가 있었다.

```
DeprecationWarning: create_react_agent is deprecated
ResourceWarning: unclosed <ssl.SSLSocket ...>
```

첫 번째는 LangGraph 버전 업데이트로 인한 API 변경 경고다. 코드를 바꿀 수 없는 상황이면 억제한다.

```python
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning, module="langgraph")
warnings.filterwarnings("ignore", category=ResourceWarning)
```

`ResourceWarning`은 ChromaDB 클라이언트를 매 요청마다 새로 만들어서 TCP 연결이 닫히지 않아 발생했다. 싱글턴으로 바꾸자 사라졌다.

---

## Slack DM 메시지 수신

Slack 채널에서 봇을 멘션(`@personal-assistant`)해야 하는 방식은 불편하다. DM으로 그냥 말하는 게 자연스럽다.

Slack 앱 설정에서 `message.im` 이벤트를 구독하면 DM 수신이 가능하다. 데스크톱 앱에서 "이 앱으로 메시지를 보내는 기능이 꺼져 있습니다"라는 문구가 뜨는데, 이는 앱의 DM 메시지 허용 설정 문제다. Slack 앱 관리 페이지의 "App Home" 탭에서 "Allow users to send Slash commands and messages from the messages tab"을 활성화하면 해결된다.

---

## 운영하면서 추가한 것들

기능을 완성하고 나서 실제로 써보니 예상 못했던 니즈가 나왔다.

**리마인더 조회·취소**: 처음엔 `set_reminder`만 있었다. "오늘 알림 뭐가 있지?" → "없습니다" (DB 확인 안 함). 조회(`list_reminders`)와 취소(`cancel_reminder`) 도구를 나중에 추가했다.

**할 일 완료 표시**: 처음엔 미완료 항목만 반환했다. 오늘 완료된 것들도 포함해서 ✅/⬜로 구분하는 게 더 유용하다는 걸 사용하면서 깨달았다.

**메모리 자동 주입**: `search_memory` 도구를 등록했는데 LLM이 개인 정보 질문에서 도구를 호출하지 않고 그냥 "모릅니다"라고 답하는 경우가 많았다. 매 요청마다 자동으로 ChromaDB를 검색해서 관련 기억을 메시지 앞에 주입하는 방식으로 바꿨다.

이런 것들은 만들기 전에 예측하기 어렵다. 직접 써봐야 알게 된다.
