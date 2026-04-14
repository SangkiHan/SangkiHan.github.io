---
title: AWS CodePipeline + EC2 CI/CD 파이프라인 구축 가이드
author: SangkiHan
date: 2026-03-17 10:00:00 +0900
categories: [DevOps, CI/CD]
tags: [DevOps, CI/CD, AWS, CodePipeline, EC2]
---

## 개요

PuppyNote 서버를 GitHub에 push하면 자동으로 EC2에 배포되는 CI/CD 파이프라인을 구축했습니다.

AWS CodePipeline, CodeBuild, CodeDeploy를 활용했고 서버는 t4g.small(ARM64) EC2 인스턴스를 사용합니다.

---

## 전체 아키텍처

```
GitHub (main push)
  → CodePipeline (자동 트리거)
    → CodeBuild (Gradle 빌드 + JAR 생성)
      → S3 (아티팩트 업로드)
        → CodeDeploy (EC2에 JAR 배포 + 재시작)
```

---

## 핵심 파일 3개

CI/CD 파이프라인은 아래 3개 파일로 동작합니다.

| 파일 | 역할 | 사용 주체 |
|------|------|----------|
| `buildspec.yml` | 빌드 명세 | CodeBuild |
| `appspec.yml` | 배포 명세 | CodeDeploy |
| `scripts/` | 배포 훅 스크립트 | CodeDeploy |

---

## buildspec.yml

CodeBuild가 읽는 빌드 설정 파일입니다.

SSM Parameter Store에서 환경변수를 읽어와 Gradle 빌드 시 주입합니다.

```yaml
version: 0.2

env:
  parameter-store:
    DB_HOST:         "/puppynote/prd/DB_HOST"
    DB_USERNAME:     "/puppynote/prd/DB_USERNAME"
    DB_PASSWORD:     "/puppynote/prd/DB_PASSWORD"
    JWT_SECRET_KEY:  "/puppynote/prd/JWT_SECRET_KEY"
    # ... 나머지 환경변수
    FIREBASE_CONFIG: "/puppynote/prd/FIREBASE_CONFIG"

phases:
  pre_build:
    commands:
      - echo "$FIREBASE_CONFIG" > src/main/resources/puppynote-firebase.json

  build:
    commands:
      - chmod +x gradlew
      # clean: QueryDSL Q클래스 충돌 방지
      - ./gradlew clean bootJar -x test -x asciidoctor --no-daemon

artifacts:
  files:
    - build/libs/*.jar
    - appspec.yml
    - scripts/**/*
  discard-paths: no

cache:
  paths:
    - '/root/.gradle/caches/**/*'
    - '/root/.gradle/wrapper/**/*'
```

### 빌드 환경 설정 (ARM64)

t4g.small은 ARM64 아키텍처이므로 CodeBuild 환경도 ARM으로 맞춥니다.

- OS: Amazon Linux 2023
- Image: `aws/codebuild/amazonlinux2023-aarch64-standard:1.0`
- Environment type: **ARM**

---

## appspec.yml

CodeDeploy가 읽는 배포 설정 파일입니다.

```yaml
version: 0.0
os: linux
files:
  - source: build/libs/
    destination: /home/ec2-user/app/
permissions:
  - object: /home/ec2-user/app/
    owner: ec2-user
    group: ec2-user
hooks:
  AfterInstall:
    - location: scripts/stop.sh
      timeout: 60
      runas: ec2-user
  ApplicationStart:
    - location: scripts/start.sh
      timeout: 120
      runas: ec2-user
  ValidateService:
    - location: scripts/health.sh
      timeout: 60
      runas: ec2-user
```

배포 순서는 다음과 같습니다.

```
JAR 파일 EC2 복사
  → stop.sh  (기존 프로세스 종료)
  → start.sh (새 JAR 실행)
  → health.sh (헬스체크 통과 확인)
```

---

## 배포 스크립트

### stop.sh

```bash
#!/bin/bash
PID=$(pgrep -f 'puppynote-server')
if [ -n "$PID" ]; then
  echo "기존 프로세스 종료: PID=$PID"
  kill -15 $PID
  sleep 5
fi
echo "종료 완료"
```

### start.sh

SSM Parameter Store에서 환경변수를 읽어 앱을 실행합니다.

```bash
#!/bin/bash
APP_DIR=/home/ec2-user/app
LOG_DIR=/home/ec2-user/logs
REGION="ap-northeast-2"
SSM_PREFIX="/puppynote/prd"

get_ssm() {
  aws ssm get-parameter \
    --name "${SSM_PREFIX}/$1" \
    --with-decryption \
    --region $REGION \
    --query "Parameter.Value" \
    --output text
}

export DB_HOST=$(get_ssm DB_HOST)
export DB_USERNAME=$(get_ssm DB_USERNAME)
# ... 나머지 환경변수

JAR_FILE=$(ls $APP_DIR/*.jar | head -1)

nohup java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -Dspring.profiles.active=prd \
  -jar $JAR_FILE \
  > $LOG_DIR/app.log 2>&1 &
```

### health.sh

```bash
#!/bin/bash
for i in {1..12}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health-check)
  if [ "$STATUS" = "200" ]; then
    echo "헬스체크 성공"
    exit 0
  fi
  echo "대기 중... ($i/12)"
  sleep 5
done
echo "헬스체크 실패"
exit 1
```

---

## AWS 콘솔 구축 순서

### 1. IAM 역할 생성

| 역할명 | Trusted Entity | 용도 |
|--------|---------------|------|
| `ecsTaskExecutionRole` | EC2 | ECS Task 실행 |
| `puppynote-ec2-role` | EC2 | EC2 인스턴스 (SSM, S3 접근) |
| `puppynote-codebuild-role` | CodeBuild | 빌드 서비스 |
| `puppynote-codedeploy-role` | CodeDeploy | 배포 서비스 |

EC2 역할에는 아래 정책이 필요합니다.
- `AmazonSSMManagedInstanceCore`
- `AmazonS3ReadOnlyAccess`
- SSM Parameter Store 접근 인라인 정책

### 2. SSM Parameter Store 환경변수 등록

민감한 정보는 모두 **SecureString** 타입으로 저장합니다.

```
/puppynote/prd/DB_HOST
/puppynote/prd/DB_PASSWORD
/puppynote/prd/JWT_SECRET_KEY
... (총 18개)
```

Firebase 설정 파일은 JSON을 한 줄로 압축해서 저장합니다.

```bash
aws ssm put-parameter \
  --name "/puppynote/prd/FIREBASE_CONFIG" \
  --value "$(cat puppynote-firebase.json | tr -d '\n')" \
  --type "SecureString"
```

### 3. S3 아티팩트 버킷 생성

CodePipeline이 빌드 산출물을 저장하는 버킷입니다.

- Bucket name: `puppynote-pipeline-artifacts-{계정ID}`
- Versioning: Enable
- Block public access: 전부 체크

### 4. EC2 인스턴스 설정

```bash
# CodeDeploy 에이전트 설치
sudo dnf install -y ruby wget
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x install
sudo ./install auto
sudo systemctl enable codedeploy-agent

# Java 17 설치
sudo dnf install -y java-17-amazon-corretto
```

### 5. GitHub 연결 (CodeStar Connections)

**Developer Tools → Settings → Connections**에서 GitHub 연결을 생성합니다.

생성 후 **반드시 콘솔에서 수동 승인(Pending → Available)** 이 필요합니다.

### 6. CodeBuild 프로젝트 생성

- Source: GitHub 연결
- Environment: ARM / Amazon Linux 2023
- Buildspec: `buildspec.yml` 자동 인식

### 7. CodeDeploy 설정

- Application: EC2/On-premises 플랫폼
- Deployment group: EC2 인스턴스 태그로 타겟 지정

### 8. CodePipeline 생성

```
Source (GitHub) → Build (CodeBuild) → Deploy (CodeDeploy)
```

---

## 트러블슈팅

### asciidoctor 빌드 실패

```
property '$3' specifies directory 'build/generated-snippets' which doesn't exist
```

`bootJar`가 `asciidoctor`에 의존하는데, CI 환경에서는 테스트 스니펫이 없어서 실패합니다.

**해결**: `-x asciidoctor` 옵션으로 스킵

### CodeBuild SSM 접근 거부

```
AccessDeniedException: not authorized to perform ssm:GetParameters
```

**해결**: CodeBuild 역할에 SSM 인라인 정책 추가

```json
{
  "Effect": "Allow",
  "Action": ["ssm:GetParameters", "ssm:GetParameter"],
  "Resource": "arn:aws:ssm:ap-northeast-2:계정ID:parameter/puppynote/*"
}
```

### stop.sh가 프로세스를 못 찾는 경우

JAR 파일명으로 `pgrep`을 해야 합니다. `app.jar` 같은 고정 이름이 아닌 실제 JAR 파일명 패턴을 사용합니다.

```bash
# 잘못된 예
PID=$(pgrep -f 'app.jar')

# 올바른 예
PID=$(pgrep -f 'puppynote-server')
```

---

## 서버 자동 복구 설정

EC2 재시작 시에도 앱이 자동으로 올라오도록 systemd 서비스로 등록합니다.

```bash
sudo tee /etc/systemd/system/puppynote.service << 'EOF'
[Unit]
Description=PuppyNote Server
After=network-online.target

[Service]
Type=forking
User=ec2-user
ExecStart=/home/ec2-user/scripts/start.sh
ExecStop=/home/ec2-user/scripts/stop.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable puppynote.service
```

---

