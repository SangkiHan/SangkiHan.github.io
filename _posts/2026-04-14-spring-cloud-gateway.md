---
layout: post
title: "Spring Cloud Gateway 구축 및 Eureka 연동"
date: 2026-04-14 11:00:00 +0900
categories: [Architecture, Spring]
tags: [Spring, Gateway, SpringCloud, Eureka, LoadBalancing, WebFlux]
---

기존에는 각 서비스를 직접 호출하거나 Nginx로 라우팅했는데, 서비스가 늘어날수록 관리가 어려워져서 Spring Cloud Gateway를 도입했다. Eureka와 연동해서 서비스가 자동으로 라우팅되고 로드밸런싱까지 되게 구성했다.

---

## 전체 구조

```
클라이언트
    │
    ▼
[Gateway :8080]  ──Eureka 조회──▶  [Eureka :8761]
    │                                     │
    │ lb://data-api                       │ 등록된 인스턴스 정보
    ▼                                     │
[data-api :8081] ◀────────────────────────┘
```

Gateway는 Eureka에서 서비스 목록을 주기적으로 받아온다. 클라이언트가 `/data-api/**`로 요청하면 Eureka에서 `data-api` 서비스의 실제 인스턴스 주소를 찾아 로드밸런싱해서 전달한다.

---

## build.gradle

Spring Boot 3.0.11, Spring Cloud 2022.0.4 기준이다.

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.0.11'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.gateway'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

ext {
    set('springCloudVersion', '2022.0.4')
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

jar {
    enabled = false
}
```

Spring Cloud Gateway는 WebFlux 기반이라 `spring-boot-starter-web`이 들어가면 충돌한다. web 의존성은 추가하지 않는다.

---

## application-dev.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: platform-be-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true              # Eureka 등록 서비스 자동 라우팅
          lower-case-service-id: true  # /data-api처럼 소문자 경로 사용
          filters:
            - StripPrefix=0          # context-path 보존 (아래 설명 참고)

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns:
              - http://localhost:3000
              - https://example.com
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 3600

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

---

## application-prd.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: platform-be-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
          filters:
            - StripPrefix=0

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns:
              - https://example.com
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 3600

eureka:
  client:
    service-url:
      defaultZone: http://${EUREKA_HOSTNAME:localhost}:8761/eureka/
  instance:
    prefer-ip-address: true
```

---

## Discovery Locator 동작 방식

`discovery.locator.enabled: true`를 설정하면 수동으로 라우트를 작성하지 않아도 된다. Eureka에 등록된 서비스를 자동으로 발견해서 라우트를 생성한다.

```
Eureka에 "data-api" 등록
    ↓
Gateway가 자동으로 /data-api/** → lb://data-api 라우트 생성
```

새 서비스가 추가될 때 Gateway 설정을 건드릴 필요 없이, 새 서비스가 Eureka에 등록만 하면 자동으로 라우팅된다.

---

## StripPrefix 이슈

Discovery Locator는 기본적으로 `StripPrefix=1`을 적용한다. 서비스 ID 경로를 제거하고 전달하는 것인데, 서비스에 `context-path`가 설정되어 있으면 문제가 된다.

**예시: `platform-data-api`의 context-path가 `/data-api`인 경우**

| | 경로 |
|---|---|
| 클라이언트 요청 | `/data-api/sems/api/inference/store-risk` |
| StripPrefix=1 (기본) | `/data-api` 제거 → `/sems/api/inference/store-risk` 전달 → **404** |
| StripPrefix=0 (수정) | 그대로 → `/data-api/sems/api/inference/store-risk` 전달 → **200** |

라우팅 결정(어느 서비스로 갈지)은 여전히 첫 번째 경로 세그먼트(`/data-api`)로 한다. StripPrefix는 그 이후 백엔드에 전달할 경로만 제어한다.

```
/data-api/sems/api/...
    ↓ 첫 세그먼트로 data-api 서비스 식별
lb://data-api 로 라우팅
    ↓ StripPrefix=0이므로 경로 그대로 전달
http://실제IP:8081/data-api/sems/api/...  ← context-path와 일치
```

---

## 요청 로깅 필터

어떤 서비스의 어떤 API를 호출했는지 Gateway 레벨에서 로그를 남긴다. 요청/응답 바디는 남기지 않고 메타정보만 기록한다.

```java
@Slf4j
@Component
public class LoggingGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        ServerHttpRequest request = exchange.getRequest();

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = System.currentTimeMillis() - startTime;
            Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
            HttpStatusCode status = exchange.getResponse().getStatusCode();

            log.info("[GATEWAY] {} {} → [{}] {} | status={} | {}ms",
                    request.getMethod(),
                    request.getURI().getPath(),
                    route != null ? route.getId() : "unknown",
                    route != null ? route.getUri() : "unknown",
                    status != null ? status.value() : "-",
                    elapsed
            );
        }));
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

로그 출력 예시:
```
[GATEWAY] GET /data-api/sems/api/inference/store-risk → [ReactiveCompositeDiscoveryClient_data-api] lb://data-api | status=200 | 142ms
```

---

## 인증 처리 방침

Gateway에서 토큰 검증을 하고 하위 서비스로 사용자 정보를 헤더로 넘기는 방식도 고려했으나, 웹 API와 모바일 API의 인증 방식이 달라서 Gateway에서 통일된 검증이 어려웠다. 결국 **각 서비스가 직접 인증을 처리**하고, Gateway는 순수하게 라우팅과 로깅만 담당하도록 했다.

Gateway는 `AuthToken` 헤더를 그대로 하위 서비스에 전달한다.

---

## 정리

| 설정 | 설명 |
|------|------|
| `discovery.locator.enabled: true` | Eureka 등록 서비스 자동 라우팅 |
| `lower-case-service-id: true` | 서비스 ID를 소문자 URL 경로로 사용 |
| `filters: StripPrefix=0` | context-path 있는 서비스 라우팅 시 필수 |
| `globalcors` | WebFlux 기반 Gateway의 CORS 설정 |
| `prefer-ip-address: true` | hostname 대신 IP로 Eureka 등록 |
