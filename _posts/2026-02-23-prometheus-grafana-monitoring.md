---
title: Spring Boot + Prometheus + Grafana 모니터링 구축기 (별도 서버 분리)
author: SangkiHan
date: 2026-02-19 20:00:00 +0900
categories: [Monitoring, Prometheus]
tags: [Monitoring, Prometheus, Grafana, Spring Boot]
---

## 개요

운영 중인 Spring Boot 서버가 저사양이라 Prometheus + Grafana를 동일 서버에 올리기 어려웠습니다.
Prometheus와 Grafana는 합쳐서 400~800MB의 메모리를 사용하기 때문에 별도 모니터링 서버로 분리하는 것이 필수였습니다.

---

## 전체 아키텍처

```
[앱 서버 (저사양)]               [모니터링 서버 (별도)]
  Spring Boot App      ←──────   Prometheus (메트릭 수집)
  /actuator/prometheus            Grafana (시각화)
  (메트릭 노출)
```

- **앱 서버**: Spring Boot Actuator로 `/actuator/prometheus` 엔드포인트 노출
- **모니터링 서버**: Prometheus가 주기적으로 앱 서버에서 메트릭을 수집하고, Grafana가 이를 시각화

---

## Step 1. Spring Boot 앱 설정

### 1-1. build.gradle 의존성 추가

```gradle
// Monitoring (Prometheus + Actuator)
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

- `spring-boot-starter-actuator`: `/actuator/prometheus` 엔드포인트를 제공
- `micrometer-registry-prometheus`: JVM, HTTP, DB 커넥션 등의 메트릭을 Prometheus 포맷으로 변환

### 1-2. application.yml 설정

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
  metrics:
    tags:
      application: addiction-prd   # 환경별로 구분 (dev/prd)
```

### 1-3. Spring Security 설정

Spring Security가 적용된 프로젝트라면 두 곳을 수정해야 합니다.

**SecurityConfig.java** - 인가(Authorization) 예외 처리:

```java
private RequestMatcher[] getPublicMatchers() {
    return new RequestMatcher[] {
        new AntPathRequestMatcher("/api/v1/jwt/**"),
        new AntPathRequestMatcher("/api/v1/auth/**"),
        // ...
        new AntPathRequestMatcher("/actuator/**")   // 추가
    };
}
```

**JwtAuthenticationFilter.java** - 인증(Authentication) 필터 예외 처리:

```java
private static final String[] excludePath = {
    "/api/v1/auth",
    "/docs",
    "/health-check",
    "/actuator"     // 추가
};
```

> `SecurityConfig`의 `permitAll()`만 추가해서는 안 됩니다. JWT 필터는 Spring Security의 인가 체크보다 먼저 실행되므로, `shouldNotFilter`에도 `/actuator` 경로를 반드시 추가해야 합니다.

---

## Step 2. 모니터링 서버 설정

### 디렉토리 구조

```
/home/ubuntu/monitoring/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.external-url=https://도메인/prometheus'
      - '--web.route-prefix=/'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123!
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://도메인/grafana
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

nginx의 서브패스(`/prometheus`, `/grafana`)를 통해 접근할 경우 반드시 아래 옵션이 필요합니다.
- Prometheus: `--web.external-url`, `--web.route-prefix=/`
- Grafana: `GF_SERVER_ROOT_URL`, `GF_SERVER_SERVE_FROM_SUB_PATH=true`

### prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'addiction-dev'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['dev서버IP:8080']
        labels:
          env: 'dev'

  - job_name: 'addiction-prd'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['prd서버IP:8080']
        labels:
          env: 'prd'
```

### 실행

```bash
# Docker 설치
sudo apt-get update
sudo apt-get install -y docker.io docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
newgrp docker

# 실행
cd /home/ubuntu/monitoring
docker-compose up -d

# 상태 확인
docker-compose ps
```

---

## Step 3. 앱 서버 방화벽 설정

Prometheus가 앱 서버의 8080 포트로 메트릭을 수집할 수 있도록, 모니터링 서버 IP에서만 접근을 허용합니다.

```bash
# 앱 서버에서 실행 (dev, prd 각각)
sudo ufw allow from 모니터링서버IP to any port 8080
```

AWS EC2라면 Security Group 인바운드 규칙에도 `TCP 8080 / 모니터링서버IP/32`를 추가해야 합니다.

---

## Step 4. nginx 설정 (서브패스 프록시)

```nginx
location /grafana {
    proxy_pass http://localhost:3000/;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

location /prometheus {
    proxy_pass http://localhost:9090/;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

---

## Step 5. Grafana 대시보드 설정

1. `https://도메인/grafana` 접속 후 로그인
2. **Connections > Data sources > Add new data source** → Prometheus 선택
3. URL 입력 후 **Save & Test**
4. **Dashboards > Import** → ID `19004` 입력 → Load → Import

> Grafana에서 Prometheus 데이터소스 URL은 Grafana 컨테이너 내부에서 사용하므로, nginx를 거치지 않고 컨테이너 간 직접 통신하는 `http://prometheus:9090`을 사용하는 것이 권장됩니다.

---

## 트러블슈팅

### 1. prometheus.yml이 디렉토리로 마운트되는 오류

**오류 메시지:**
```
Are you trying to mount a directory onto a file (or vice-versa)?
Check if the specified host path exists and is the expected type
```

**원인:**
`prometheus/prometheus.yml` 파일이 존재하지 않는 상태에서 `docker-compose up`을 실행하면 Docker가 해당 경로를 파일이 아닌 디렉토리로 생성합니다.

**해결:**
```bash
# 잘못 생성된 디렉토리 제거
rm -rf /home/ubuntu/monitoring/prometheus/prometheus.yml

# 파일 직접 생성
cat > /home/ubuntu/monitoring/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  ...
EOF

docker-compose down && docker-compose up -d
```

---

### 2. prometheus.yml YAML 파싱 오류

**오류 메시지:**
```
yaml: unmarshal errors:
  line 10: field labels not found in type config.ScrapeConfig
```

**원인:**
`labels`의 위치가 잘못됐습니다. `labels`는 `static_configs` 내부의 각 타겟 항목 아래에 위치해야 합니다.

**AS-IS (잘못된 구조):**
```yaml
scrape_configs:
  - job_name: 'addiction-dev'
    static_configs:
      - targets: ['서버IP:8080']
    labels:          # ← 잘못된 위치 (ScrapeConfig 레벨)
      env: 'dev'
```

**TO-BE (올바른 구조):**
```yaml
scrape_configs:
  - job_name: 'addiction-dev'
    static_configs:
      - targets: ['서버IP:8080']
        labels:     # ← 올바른 위치 (targets 항목 아래)
          env: 'dev'
```

---

### 3. Grafana 데이터소스 - 502 Bad Gateway

**오류 메시지:**
```
502 Bad Gateway - There was an error returned querying the Prometheus API.
```

**원인:**
nginx는 `/prometheus` → `http://localhost:9090`으로 프록시 설정이 되어 있었지만, Prometheus 컨테이너가 YAML 파싱 오류로 계속 재시작(`Restarting`) 상태였기 때문입니다.

**진단:**
```bash
docker ps -a          # Restarting 상태 확인
docker logs prometheus # 오류 원인 확인
curl http://localhost:9090/-/healthy
```

**해결:** prometheus.yml의 YAML 구조 오류를 수정 후 재시작

---

### 4. Grafana 데이터소스 - 404 Not Found

**오류 메시지:**
```
404 Not Found - There was an error returned querying the Prometheus API.
```

**원인:**
Grafana 컨테이너 내부에서 `http://prometheus:9090`으로 연결 시 Docker 내부 DNS(127.0.0.11)가 제대로 동작하지 않았습니다.

**해결:**
Docker 컨테이너 간 직접 통신 대신, 호스트의 Docker 브리지 IP를 사용합니다.

```bash
# Docker 네트워크 게이트웨이 IP 확인
docker network inspect monitoring_default | grep Gateway
```

데이터소스 URL을 `http://172.18.0.1:9090` 형태로 설정하거나, Prometheus가 정상 기동된 후 다시 `http://prometheus:9090`을 시도합니다.

---

### 5. actuator 엔드포인트 401 Unauthorized

**오류 메시지:**
```json
{"statusCode":401,"httpStatus":"UNAUTHORIZED","message":"JWT토큰이 수신되지 않았거나 형식이 맞지않습니다."}
```

**원인:**
Spring Security의 `SecurityConfig`에 `/actuator/**`를 `permitAll()`로 추가했지만, JWT 인증 필터(`JwtAuthenticationFilter`)가 Spring Security의 인가 체크보다 먼저 실행되어 401을 반환했습니다.

`JwtAuthenticationFilter`는 `OncePerRequestFilter`를 상속하며, `shouldNotFilter` 메서드로 특정 경로를 필터에서 제외할 수 있습니다.

**AS-IS:**
```java
private static final String[] excludePath = {
    "/api/v1/auth",
    "/docs",
    "/health-check"
    // /actuator 누락!
};
```

**TO-BE:**
```java
private static final String[] excludePath = {
    "/api/v1/auth",
    "/docs",
    "/health-check",
    "/actuator"    // 추가
};
```

> Spring Security에서 `permitAll()`은 인가(Authorization) 레이어를 우회하지만, 그 이전에 실행되는 커스텀 필터는 별도로 처리해야 합니다.

---

### 6. prd 서버 도메인 타겟 스크랩 실패

**원인:**
Blue/Green 배포 환경에서 포트가 변경되므로 IP:port 대신 도메인을 타겟으로 설정했는데, `targets`에 포트를 명시하지 않으면 Prometheus가 기본 포트인 `80`으로 접근을 시도합니다. HTTPS 도메인이므로 443으로 연결해야 합니다.

**AS-IS (잘못된 설정):**
```yaml
- job_name: 'addiction-prd'
  scheme: https
  static_configs:
    - targets: ['api.quitmate.co.kr']   # 포트 미지정 → 80 포트로 시도
```

**TO-BE (올바른 설정):**
```yaml
- job_name: 'addiction-prd'
  scheme: https
  static_configs:
    - targets: ['api.quitmate.co.kr:443']   # 443 포트 명시
```

> Blue/Green 배포처럼 앱 서버의 포트가 동적으로 변경되는 환경에서는 nginx가 라우팅을 담당하므로, IP:port 대신 도메인:443으로 설정하는 것이 올바른 방법입니다.

---

## 최종 확인

```bash
# 앱 서버에서 메트릭 정상 노출 확인
curl http://localhost:8080/actuator/prometheus
# → jvm_memory_used_bytes, http_server_requests_seconds 등 메트릭 출력

# Prometheus 타겟 상태 확인
https://도메인/prometheus/targets
# → addiction-dev, addiction-prd 모두 UP 상태 확인
```

Grafana 대시보드(ID: 19004)에서 JVM 메모리, CPU, HTTP 요청 수, HikariCP 커넥션 현황 등을 실시간으로 확인할 수 있습니다.
