---
layout: post
title: "테스트용 MSA 앱 만들기 — 로직 없이 이름만 돌려주는 6개 서비스"
date: 2026-10-05 10:00:00 +0900
categories: [DevOps, Terraform 구축]
tags: [Terraform, Spring Boot, Gradle, SpringCloudGateway, Jib]
---

[지난 포스트](/posts/terraform-ecs-intro)에서 전체 그림을 봤다. 이번엔 Terraform이 올릴 **대상 앱**을 만든다. 목적이 인프라 테스트라서 앱은 최대한 단순하게 만든다.

---

## 프로젝트 구조

실제로 운영 중인 MSA 프로젝트와 같은 모양으로, Gradle 멀티모듈로 서비스를 나눴다.

```
terraform-study/
├── settings.gradle.kts
├── build.gradle.kts
├── apps/
│   ├── gateway/        # 9000
│   ├── chatting/       # 8080
│   ├── session/        # 8081
│   ├── user/           # 8082
│   ├── push/           # 8084
│   └── push-worker/    # 8085
└── infra/terraform/    # 다음 글부터
```

```kotlin
// settings.gradle.kts
rootProject.name = "terraform-study"

include(":apps:gateway", ":apps:chatting", ":apps:session", ":apps:user", ":apps:push", ":apps:push-worker")
```

각 서비스 안은 헥사고날 구조의 흔적만 남겼다. 컨트롤러는 `adapter/in/web` 아래에 둔다.

```
apps/user/src/main/java/com/genesisnest/user/
├── UserApplication.java
└── adapter/in/web/HomeController.java
```

---

## HomeController: 이름만 돌려준다

서비스 6개가 전부 같은 모양이고 반환값만 다르다.

```java
@RestController
public class HomeController {

    @GetMapping("/")
    public String home() {
        return "user-service";
    }
}
```

각 서비스 의존성은 딱 두 개다.

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

`actuator`를 넣은 이유는 **헬스체크** 때문이다. ALB와 ECS가 `/actuator/health`를 호출해서 서비스가 살아 있는지 판단한다. 이게 없으면 인프라가 서비스를 정상으로 인식하지 못한다.

---

## gateway: 설정만으로 라우팅

gateway는 Spring Cloud Gateway를 쓴다. 코드는 한 줄도 없고, `application.yml`에 경로를 적은 게 전부다.

```yaml
server:
  port: 9000

# 업스트림은 환경변수로 덮어쓴다 (ECS Service Connect DNS 등)
gateway:
  upstreams:
    chatting: ${CHATTING_URL:http://localhost:8080}
    session: ${SESSION_URL:http://localhost:8081}
    user: ${USER_URL:http://localhost:8082}
    push: ${PUSH_URL:http://localhost:8084}

spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: user
              uri: ${gateway.upstreams.user}
              predicates: [Path=/user/**]
              filters: ["RewritePath=/user(?<segment>/?.*), /${segment}"]
            # chatting, session, push도 같은 모양
```

이 설정에서 중요한 점은 두 가지다.

**1. 업스트림 주소가 환경변수다.** 로컬에서는 기본값(`localhost:8082`)을 쓰고, ECS에서는 환경변수로 서비스 이름(`http://user:8082`)을 넣어 덮어쓴다. 이걸 빼먹으면 gateway가 **자기 자신의 localhost를 호출**한다. 기동은 정상으로 되고 라우팅만 조용히 깨지는 최악의 버그다.

**2. `RewritePath`로 접두사를 뗀다.** `/user`로 들어온 요청을 user 서비스의 `/`로 보낸다. user 서비스에는 `/user`라는 경로가 없기 때문이다.

로컬에서 gateway, user, chatting을 같이 띄워서 확인했다.

```
/         → gateway-service
/user     → user-service
/chatting → chatting-service
```

---

## 이미지 빌드: Jib

컨테이너 이미지는 Docker 없이 **Jib**로 만든다. Gradle 플러그인이 이미지 빌드와 레지스트리 push를 한 번에 해준다.

```kotlin
jib {
    from {
        image = "amazoncorretto:25"
        platforms {
            platform {
                architecture = "arm64"
                os = "linux"
            }
        }
    }
    to.image = "terraform-study/user"
    container {
        ports = listOf("8082")
        mainClass = "com.genesisnest.user.UserApplication"
        format = com.google.cloud.tools.jib.api.buildplan.ImageFormat.OCI
        jvmFlags = listOf("-XX:MaxRAMPercentage=75.0")
    }
}
```

- **arm64**로 빌드한다. Fargate에서 ARM(Graviton)이 더 저렴하고, 뒤에서 만들 ECS 태스크도 ARM64로 맞춘다. 이미지 아키텍처와 태스크 아키텍처가 다르면 `exec format error`가 난다.
- `mainClass`를 직접 지정한 이유는 5편에서 다룬다. 처음엔 빼고 썼다가 빌드가 실패했다.

---

## 다음 글

앱이 준비됐으니 이제 이걸 올릴 인프라를 설계한다. 다음 글에서는 Terraform 코드를 어떻게 나눌지 다룬다.

➡️ 시리즈 인덱스: [Terraform으로 AWS ECS 서비스 배포하기](/posts/terraform-ecs-intro)
