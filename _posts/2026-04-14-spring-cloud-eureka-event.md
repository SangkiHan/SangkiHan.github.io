---
layout: post
title: "Spring Cloud Eureka 서비스 등록/해제 이벤트 감지"
date: 2026-04-14 12:00:00 +0900
categories: [Architecture, Spring]
tags: [Spring, Eureka, SpringCloud, Event, Alert]
---

Eureka 서버에 서비스가 올라오거나 내려갈 때 알람을 보내고 싶었다. Spring Cloud Eureka Server는 서비스 등록/해제 시 ApplicationEvent를 발행하기 때문에 `@EventListener`로 간단하게 감지할 수 있다.

---

## 사용 이벤트

| 이벤트 | 발생 시점 |
|---|---|
| `EurekaInstanceRegisteredEvent` | 서비스가 Eureka에 처음 등록될 때 |
| `EurekaInstanceCanceledEvent` | 서비스가 정상 종료되거나 heartbeat 타임아웃으로 제거될 때 |
| `EurekaInstanceRenewedEvent` | 서비스가 heartbeat를 보낼 때 (30초마다) |

heartbeat 이벤트는 너무 자주 발생해서 알람 용도로는 사용하지 않는다.

---

## 구현

```java
@Slf4j
@Component
public class EurekaInstanceEventListener {

    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @EventListener
    public void onInstanceRegistered(EurekaInstanceRegisteredEvent event) {
        InstanceInfo instanceInfo = event.getInstanceInfo();
        String appName = instanceInfo.getAppName();
        String instanceId = instanceInfo.getInstanceId();
        String ipAddr = instanceInfo.getIPAddr();
        int port = instanceInfo.getPort();
        String time = LocalDateTime.now().format(FORMATTER);

        log.info("[EUREKA] 서비스 등록 - appName={}, instanceId={}, ip={}:{}, time={}",
                appName, instanceId, ipAddr, port, time);

        String message = String.format("[Eureka] 서비스 등록\n앱: %s\n인스턴스: %s\n주소: %s:%d\n시각: %s",
                appName, instanceId, ipAddr, port, time);

        sendAlert(message);
    }

    @EventListener
    public void onInstanceCanceled(EurekaInstanceCanceledEvent event) {
        String appName = event.getAppName();
        String serverId = event.getServerId();
        String time = LocalDateTime.now().format(FORMATTER);

        log.warn("[EUREKA] 서비스 해제 - appName={}, serverId={}, time={}",
                appName, serverId, time);

        String message = String.format("[Eureka] 서비스 해제\n앱: %s\n인스턴스: %s\n시각: %s",
                appName, serverId, time);

        sendAlert(message);
    }

    private void sendAlert(String message) {
        // TODO: 알람 전송 구현
        // Slack 예시:
        // slackNotifier.send(message);

        // 카카오 알림톡 예시:
        // kakaoNotifier.send(message);

        // 이메일 예시:
        // emailNotifier.send("Eureka 서비스 변경 알람", message);
    }
}
```

`EurekaInstanceRegisteredEvent`에서는 `getInstanceInfo()`로 상세 정보(IP, 포트 등)를 가져올 수 있다. `EurekaInstanceCanceledEvent`는 이미 해제된 인스턴스라 `getAppName()`과 `getServerId()`만 제공한다.

---

## 로그 출력 예시

서비스 등록 시:
```
[EUREKA] 서비스 등록 - appName=DATA-API, instanceId=192.168.0.10:data-api:8081, ip=192.168.0.10:8081, time=2026-04-14 10:00:00
```

서비스 해제 시:
```
[EUREKA] 서비스 해제 - appName=DATA-API, serverId=192.168.0.10:data-api:8081, time=2026-04-14 10:05:00
```

---

## 주의사항

서비스가 정상 종료(`/actuator/shutdown` 또는 프로세스 종료)되면 `EurekaInstanceCanceledEvent`가 즉시 발생한다.

반면 서비스가 갑자기 죽어서 heartbeat가 끊기는 경우에는 Eureka의 eviction 주기(기본 60초)가 지난 후에 이벤트가 발생한다. `enable-self-preservation: false`로 설정해두어야 eviction이 정상적으로 동작한다.
