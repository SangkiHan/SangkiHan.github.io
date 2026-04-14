---
layout: post
title: "Spring Cloud Eureka 서버 구축 및 서비스 등록"
date: 2026-04-14 10:00:00 +0900
categories: [Architecture, Spring Cloud]
tags: [Architecture, Spring Cloud, Eureka, ServiceDiscovery]
---

마이크로서비스 구조에서 서비스 디스커버리를 위해 Spring Cloud Netflix Eureka를 도입했다. Gateway가 각 서비스의 IP를 직접 하드코딩하는 게 아니라, Eureka를 통해 동적으로 서비스를 발견하고 로드밸런싱까지 가능하게 하는 게 목표였다.

---

## 전체 구조

```
[platform-data-api] ──등록──▶ [Eureka Server :8761]
[platform-be-gateway] ──조회──▶ [Eureka Server :8761]
                                        │
                        lb://data-api 로 동적 라우팅
```

서비스들이 Eureka에 자신을 등록하면, Gateway는 Eureka에서 서비스 목록을 조회해서 자동으로 라우팅한다. 나중에 VM이 추가되더라도 새 인스턴스가 Eureka에 등록만 하면 자동으로 로드밸런싱 대상에 포함된다.

---

## build.gradle

Spring Boot 3.0.11, Spring Cloud 2022.0.4 기준이다.

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.0.11'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.eureka'
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
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
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

**주의: `io.spring.dependency-management` 버전은 반드시 `1.1.7` 이상으로 맞춰야 한다.**

Gradle 9.x에서 1.1.3 버전을 사용하면 아래 에러가 발생한다.

```
NoSuchMethodError: LenientConfiguration.getArtifacts(Ljava/util/Set;)Ljava/util/Set;
```

---

## 메인 클래스

```java
@EnableEurekaServer
@SpringBootApplication
public class PlatformBeEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(PlatformBeEurekaApplication.class, args);
    }
}
```

`@EnableEurekaServer` 하나만 붙이면 된다.

---

## application-dev.yml

```yaml
server:
  port: 8761

spring:
  application:
    name: platform-be-eureka

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false  # 서버 자신은 Eureka에 등록하지 않음
    fetch-registry: false        # 다른 서비스 목록을 가져오지 않음
    service-url:
      defaultZone: http://localhost:8761/eureka/
  server:
    wait-time-in-ms-when-sync-empty: 0  # 시작 시 불필요한 대기 제거
    enable-self-preservation: false     # dev: 서비스 수가 적어 임계값 미달 경고 방지
```

---

## application-prd.yml

```yaml
server:
  port: 8761

spring:
  application:
    name: platform-be-eureka

eureka:
  instance:
    hostname: ${EUREKA_HOSTNAME:localhost}
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${EUREKA_HOSTNAME:localhost}:8761/eureka/
  server:
    wait-time-in-ms-when-sync-empty: 0
    enable-self-preservation: false
```

prd에서도 서비스 수가 많지 않으면 self-preservation을 꺼두는 게 낫다. 켜두면 아래 경고가 계속 뜬다.

```
EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT.
RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.
```

---

## Self-Preservation Mode란

Eureka는 등록된 인스턴스 수의 **85%** 이상이 30초마다 heartbeat를 보내야 정상으로 판단한다.

```
임계값 = 등록된 인스턴스 수 × 2 × 0.85
```

서비스가 1~2개뿐이면 분당 heartbeat 횟수가 임계값에 못 미쳐서 자기 보존 모드에 진입한다. 이 모드에서는 서비스가 응답 없어도 강제로 제거하지 않는다. 네트워크 이슈로 인한 대량 해제를 방지하기 위한 안전장치인데, dev 환경에서는 그냥 꺼두면 된다.

---

## 클라이언트 서비스 등록 (data-api 예시)

### build.gradle

```gradle
// Eureka Client
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:4.0.3'
```

### application-dev.yml

```yaml
spring:
  application:
    name: data-api  # Eureka 등록 이름 = Gateway 라우팅 경로

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

`spring.application.name`이 Eureka에 등록되는 서비스 ID가 된다. Gateway는 이 이름으로 라우팅한다.

`@EnableDiscoveryClient` 어노테이션은 Spring Boot 3 + Spring Cloud 2022 조합에서는 붙이지 않아도 자동으로 등록된다.

---

## 확인

Eureka 서버 기동 후 `http://localhost:8761` 접속하면 대시보드에서 등록된 서비스를 확인할 수 있다.

```
Instances currently registered with Eureka
Application    AMIs    Availability Zones    Status
DATA-API        n/a         (1)            UP (1) - ...
```

서비스 이름은 대문자로 표시된다. Gateway의 discovery locator에서 `lower-case-service-id: true` 설정을 하면 `/data-api/**` 경로로 라우팅된다.

---

## 실행 순서

1. **Eureka Server** 먼저 기동 → `http://localhost:8761` 대시보드 확인
2. **platform-data-api** 기동 → 대시보드에 `DATA-API` 등록 확인
3. **platform-be-gateway** 기동 → `PLATFORM-BE-GATEWAY` 등록 확인
