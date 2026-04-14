---
layout: post
title: "Eureka 없이 Spring Cloud Gateway에서 로드밸런싱하기"
date: 2026-04-14 13:00:00
categories: [Architecture, Spring Cloud]
tags: [Architecture, Spring Cloud, Gateway, LoadBalancing]
---

Spring Cloud Eureka를 도입했다가 제거하고 Gateway에서 직접 VM을 관리하는 방식으로 변경했다. 이유와 설정 방법을 정리한다.

---

## Eureka를 제거한 이유

Eureka는 서비스가 많고 인스턴스가 동적으로 늘어나는 환경에서 진가를 발휘한다. 새 VM이 뜨면 Eureka에 자동 등록되고, Gateway는 그걸 감지해서 자동으로 라우팅 대상에 포함시킨다.

하지만 현재 구조는 다르다.

- 서비스당 VM이 1~2개로 고정
- VM이 동적으로 추가되는 일이 거의 없음
- Eureka 서버 자체도 하나의 운영 포인트가 됨 (Eureka가 죽으면 라우팅 불가)

이 상황에서 Eureka를 유지하는 건 운영 복잡도만 올라가고 실제 이점이 없었다. **VM 목록이 고정적이라면 Gateway에서 직접 관리하는 게 더 단순하고 안정적이다.**

---

## 변경 전/후 구조

**변경 전 (Eureka 사용)**
```
클라이언트
    ↓
Gateway :8080
    ↓ Eureka에서 인스턴스 조회
Eureka Server :8761
    ↓
data-api VM
```

**변경 후 (SimpleDiscoveryClient 사용)**
```
클라이언트
    ↓
Gateway :8080
    ↓ yml에 정의된 정적 인스턴스 목록 조회
data-api VM1 / VM2 (round-robin)
```

Eureka 서버가 없어도 `lb://` 로드밸런싱이 된다. Gateway 내부에서 `SimpleDiscoveryClient`가 yml에 정의된 인스턴스 목록을 관리하고, Spring Cloud LoadBalancer가 round-robin으로 분배한다.

---

## build.gradle

eureka-client 대신 loadbalancer만 있으면 된다.

```gradle
ext {
    set('springCloudVersion', '2022.0.4')
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

---

## application-dev.yml

로컬 개발 환경은 인스턴스 1개만 등록한다.

```yaml
server:
  port: 8080

spring:
  application:
    name: platform-be-gateway
  cloud:
    discovery:
      client:
        simple:
          instances:
            data-api:
              - uri: http://localhost:8081
    gateway:
      routes:
        - id: data-api
          uri: lb://data-api
          predicates:
            - Path=/data-api/**

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns:
              - http://localhost:3000
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 3600
```

---

## application-prd.yml

prd는 서비스별로 VM IP를 등록한다. VM이 추가되면 `instances` 목록에 항목만 추가하면 된다.

```yaml
server:
  port: 8080

spring:
  application:
    name: platform-be-gateway
  cloud:
    discovery:
      client:
        simple:
          instances:
            data-api:
              - uri: http://VM1_IP  # VM1 data-api
              - uri: http://VM2_IP  # VM2 data-api (추가 시)
            mobile-api:
              - uri: http://VM1_IP  # VM1 mobile-api
    gateway:
      routes:
        - id: data-api
          uri: lb://data-api
          predicates:
            - Path=/data-api/**
        - id: mobile-api
          uri: lb://mobile-api
          predicates:
            - Path=/api/**

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
```

---

## 핵심 포인트

**`spring.cloud.discovery.client.simple.instances`**

Eureka 없이 서비스 인스턴스를 정적으로 정의하는 설정이다. 서비스 이름(`data-api`, `mobile-api`)이 키가 되고, `uri` 목록이 로드밸런싱 대상이 된다.

**`lb://data-api`**

`lb://`는 Spring Cloud LoadBalancer에게 해당 서비스 이름으로 등록된 인스턴스들을 round-robin으로 분배하라는 의미다. Eureka가 없어도 `SimpleDiscoveryClient`가 인스턴스 목록을 제공하면 동일하게 동작한다.

**새 서비스 추가 시**

라우트와 인스턴스를 각각 추가하면 된다.

```yaml
instances:
  new-service:
    - uri: http://VM_IP

routes:
  - id: new-service
    uri: lb://new-service
    predicates:
      - Path=/new-service/**
```

---

## 장애 시 자동 failover (Retry 필터)

VM1이 죽었을 때 Gateway가 자동으로 VM2로 재시도하게 하려면 `Retry` 필터를 추가한다.

```yaml
gateway:
  default-filters:
    - name: Retry
      args:
        retries: 1                                               # 실패 시 다른 인스턴스로 1회 재시도
        statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE, GATEWAY_TIMEOUT  # 재시도할 HTTP 상태
        methods: GET                                             # GET만 재시도 (POST는 중복 처리 위험)
        backoff:
          firstBackoff: 10ms
          maxBackoff: 100ms
          factor: 2
          basedOnPreviousValue: false
```

**동작 흐름:**
```
VM1 요청 → 502 BAD_GATEWAY
    ↓ Retry 필터 동작
VM2 요청 → 200 OK  (클라이언트는 오류 없이 응답 받음)
```

**`methods: GET`만 재시도하는 이유**

POST/PUT/DELETE는 같은 요청이 두 번 처리될 수 있다. 예를 들어 VM1에서 데이터 저장이 완료됐는데 응답 도중 연결이 끊긴 경우, Retry로 VM2에 재시도하면 데이터가 중복 저장된다. 멱등성이 보장되는 API라면 추가할 수 있다.

---

## 오류 발생 VM 알람

Retry가 동작해서 클라이언트는 정상 응답을 받더라도, 어떤 VM에서 오류가 났는지 운영자는 알아야 한다. `LoadBalancerLifecycle`을 구현하면 LoadBalancer가 인스턴스를 선택하고 요청을 완료할 때 훅이 호출된다.

```java
@Slf4j
@Component
public class LoadBalancerFailureListener implements LoadBalancerLifecycle<RequestDataContext, ResponseData, ServiceInstance> {

    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @Override
    public boolean supports(Class requestContextClass, Class responseClass, Class serverTypeClass) {
        return ServiceInstance.class.isAssignableFrom(serverTypeClass);
    }

    @Override
    public void onStart(Request<RequestDataContext> request) {}

    @Override
    public void onStartRequest(Request<RequestDataContext> request, Response<ServiceInstance> lbResponse) {}

    @Override
    public void onComplete(CompletionContext<ResponseData, ServiceInstance, RequestDataContext> completionContext) {
        CompletionContext.Status status = completionContext.status();

        if (status == CompletionContext.Status.FAILED || status == CompletionContext.Status.DISCARD) {
            ServiceInstance instance = completionContext.getLoadBalancerResponse().getServer();
            String serviceId = instance.getServiceId();
            String host = instance.getHost();
            int port = instance.getPort();
            String time = LocalDateTime.now().format(FORMATTER);
            Throwable cause = completionContext.getThrowable();

            log.warn("[GATEWAY] 인스턴스 오류 감지 - serviceId={}, host={}:{}, status={}, cause={}",
                    serviceId, host, port, status, cause != null ? cause.getMessage() : "unknown");

            String message = String.format(
                    "[Gateway] 인스턴스 오류 감지\n서비스: %s\nVM: %s:%d\n상태: %s\n원인: %s\n시각: %s",
                    serviceId, host, port, status,
                    cause != null ? cause.getMessage() : "unknown",
                    time
            );

            sendAlert(message);
        }
    }

    private void sendAlert(String message) {
        // TODO: 알람 전송 구현
        // Slack 예시:
        // slackNotifier.send(message);

        // 카카오 알림톡 예시:
        // kakaoNotifier.send(message);

        // 이메일 예시:
        // emailNotifier.send("Gateway 인스턴스 오류 알람", message);
    }
}
```

**`CompletionContext.Status` 종류:**

| 상태 | 의미 |
|---|---|
| `SUCCESSFULL` | 요청 성공 |
| `FAILED` | 연결 실패, 타임아웃 등 네트워크 오류 |
| `DISCARD` | 인스턴스 선택 불가 (등록된 인스턴스 없음) |

Retry와 함께 동작하면 아래처럼 흐른다.

```
VM1 요청 → 실패
    ↓ onComplete(FAILED) → 로그 + 알람 발송 (VM1 IP 포함)
    ↓ Retry 필터가 VM2로 재시도
VM2 요청 → 성공
    ↓ onComplete(SUCCESSFULL) → 알람 없음
```

클라이언트는 정상 응답을 받지만, 운영자는 VM1에서 오류가 발생했다는 알람을 받는다.

---

## Eureka가 필요한 시점

지금 구조에서는 필요 없지만, 아래 상황이 되면 다시 고려할 수 있다.

- VM이 오토스케일링으로 동적으로 추가/제거되는 경우
- 서비스가 10개 이상으로 늘어나서 yml 관리가 어려워지는 경우
- 서비스가 직접 헬스체크 기반으로 라우팅 대상에서 빠져야 하는 경우
