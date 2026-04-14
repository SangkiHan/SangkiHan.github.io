---
title: EC2 단일 서버 Blue/Green 무중단 배포 (Nginx + 포트 스위칭)
author: SangkiHan
date: 2026-03-17 11:00:00 +0900
categories: [DevOps, CI/CD]
tags: [DevOps, CI/CD, AWS, EC2, BlueGreen, Nginx]
---

## 개요

ALB나 Auto Scaling Group 없이 **EC2 단일 서버에서 Blue/Green 무중단 배포**를 구현했습니다.

Nginx의 `proxy_pass` 포트를 배포 시마다 교체하는 방식으로, 추가 인프라 비용 없이 다운타임 제로 배포가 가능합니다.

---

## 기존 In-place 배포의 문제

```
stop.sh  → 기존 프로세스 종료   ← 이 순간 서비스 중단
start.sh → 새 버전 실행
```

앱 재시작 시간(보통 30초~1분) 동안 503 에러가 발생합니다.

---

## Blue/Green 포트 스위칭 방식

```
EC2 한 대에서 두 포트를 번갈아 사용

[현재 상태]  Nginx :443 → 8081 (v1 운영 중)
[배포 시작]  8080 포트에 v2 조용히 실행
[헬스체크]   8080 헬스체크 통과 확인
[트래픽 전환] Nginx proxy_pass: 8081 → 8080 (무중단)
[구버전 종료] 8081 프로세스 종료

[다음 배포]  역할 교체: 8081에 v3 배포 → 8080 → 8081 전환
```

사용자 입장에서는 트래픽 전환 순간 끊김이 없습니다.

---

## 전체 구조

```
appspec.yml
  ├── AfterInstall  → stop.sh   (비활성 포트 정리)
  ├── ApplicationStart → start.sh  (비활성 포트에 새 버전 실행)
  └── ValidateService
        ├── health.sh  (비활성 포트 헬스체크)
        └── switch.sh  (Nginx 트래픽 전환 + 구버전 종료)
```

---

## appspec.yml

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
    - location: scripts/switch.sh
      timeout: 30
      runas: root
```

---

## 배포 스크립트

### stop.sh

배포 전 비활성 포트의 기존 프로세스를 정리합니다.

```bash
#!/bin/bash
INACTIVE_PORT=$(cat /home/ec2-user/app/inactive_port 2>/dev/null || echo "8081")

PID=$(lsof -ti:$INACTIVE_PORT 2>/dev/null)
if [ -n "$PID" ]; then
  echo "비활성 포트($INACTIVE_PORT) 프로세스 종료: PID=$PID"
  kill -15 $PID
  sleep 3
fi
echo "정리 완료"
```

### start.sh

`active_port` 파일을 읽어 현재 운영 포트를 확인하고, 반대 포트에 새 버전을 실행합니다.

```bash
#!/bin/bash
APP_DIR=/home/ec2-user/app
LOG_DIR=/home/ec2-user/logs

# 현재 활성 포트 확인 → 반대 포트에 배포
ACTIVE_PORT=$(cat $APP_DIR/active_port 2>/dev/null || echo "8080")
if [ "$ACTIVE_PORT" = "8080" ]; then
  INACTIVE_PORT="8081"
else
  INACTIVE_PORT="8080"
fi

echo "활성 포트: $ACTIVE_PORT → 배포 대상 포트: $INACTIVE_PORT"
echo $INACTIVE_PORT > $APP_DIR/inactive_port

# SSM Parameter Store에서 환경변수 로드
get_ssm() {
  aws ssm get-parameter \
    --name "/puppynote/prd/$1" \
    --with-decryption \
    --region ap-northeast-2 \
    --query "Parameter.Value" \
    --output text
}

export DB_HOST=$(get_ssm DB_HOST)
export DB_USERNAME=$(get_ssm DB_USERNAME)
export DB_PASSWORD=$(get_ssm DB_PASSWORD)
# ... 나머지 환경변수

JAR_FILE=$(ls $APP_DIR/*.jar | head -1)

nohup java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -Dspring.profiles.active=prd \
  -Dserver.port=$INACTIVE_PORT \
  -jar $JAR_FILE \
  > $LOG_DIR/app-$INACTIVE_PORT.log 2>&1 &

echo "앱 시작됨: PID=$! / 포트: $INACTIVE_PORT"
```

### health.sh

비활성 포트에서 새 버전이 정상 기동됐는지 확인합니다.

```bash
#!/bin/bash
INACTIVE_PORT=$(cat /home/ec2-user/app/inactive_port 2>/dev/null || echo "8081")

echo "헬스체크 대상 포트: $INACTIVE_PORT"

for i in {1..12}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:$INACTIVE_PORT/health-check)
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

### switch.sh

헬스체크 통과 후 Nginx의 `proxy_pass` 포트를 교체하고 구버전을 종료합니다.

```bash
#!/bin/bash
APP_DIR=/home/ec2-user/app
INACTIVE_PORT=$(cat $APP_DIR/inactive_port)
ACTIVE_PORT=$(cat $APP_DIR/active_port 2>/dev/null || echo "8080")

echo "Nginx 트래픽 전환: $ACTIVE_PORT → $INACTIVE_PORT"

# nginx.conf의 proxy_pass 포트 교체
sudo sed -i "s/proxy_pass http:\/\/localhost:$ACTIVE_PORT/proxy_pass http:\/\/localhost:$INACTIVE_PORT/" /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx

# 구버전 프로세스 종료
OLD_PID=$(lsof -ti:$ACTIVE_PORT 2>/dev/null)
if [ -n "$OLD_PID" ]; then
  echo "구버전 종료: 포트=$ACTIVE_PORT PID=$OLD_PID"
  kill -15 $OLD_PID
fi

# 활성 포트 업데이트
echo $INACTIVE_PORT > $APP_DIR/active_port
echo "전환 완료: 현재 활성 포트 → $INACTIVE_PORT"
```

---

## Nginx 설정

`proxy_pass`에 포트를 직접 명시합니다. switch.sh의 `sed`가 이 포트 숫자를 교체합니다.

```nginx
location / {
    proxy_pass http://localhost:8081;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

---

## EC2 초기 설정 (1회)

```bash
# 초기 활성 포트 설정 (nginx의 현재 proxy_pass 포트와 일치)
echo "8081" > /home/ec2-user/app/active_port

# switch.sh가 sudo nginx 명령을 실행할 수 있도록 권한 부여
echo "ec2-user ALL=(ALL) NOPASSWD: /usr/sbin/nginx, /bin/systemctl reload nginx, /usr/bin/sed" \
  | sudo tee /etc/sudoers.d/puppynote
```

---

## In-place vs Blue/Green 비교

| 항목 | In-place | Blue/Green (포트 스위칭) |
|------|----------|------------------------|
| 다운타임 | 있음 (재시작 시간) | 없음 |
| 롤백 | 재배포 필요 | 구버전 프로세스 살아있으면 즉시 |
| 메모리 | 1개 프로세스 | 배포 중 잠시 2개 동시 실행 |
| 추가 인프라 | 불필요 | 불필요 |
| 복잡도 | 단순 | 다소 복잡 |

---

## 마무리

ALB, Auto Scaling Group 없이도 Nginx 포트 스위칭만으로 무중단 배포가 가능합니다.

소규모 서비스에서 인프라 비용을 최소화하면서 무중단 배포를 원할 때 효과적인 방법입니다.
