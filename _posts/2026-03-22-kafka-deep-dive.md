---
layout: post
title: "Kafka 동작 원리 Deep Dive - 브로커부터 복제까지"
date: 2026-03-22 18:00:00 +0900
categories: [Event, Kafka]
tags: [Architecture, Kafka, KRaft, Partition, Replication]
---

Kafka를 직접 구성하고 운영하면서 생긴 궁금증들을 파고든 내용을 정리했다.

---

## 1. 브로커 구성 - 왜 3대인가

Kafka 클러스터는 **과반수(Quorum)** 원칙으로 동작한다.

| 브로커 수 | 과반수 | 허용 장애 수 |
|:---:|:---:|:---:|
| 1대 | 1대 | 0대 |
| 2대 | 2대 | **0대** |
| 3대 | 2대 | **1대** |
| 5대 | 3대 | **2대** |

**2대가 1대보다 장애 내성이 낮다.** 브로커 2대 중 1대가 죽으면 남은 1대가 과반수(2/2)를 충족하지 못해 클러스터 전체가 멈춘다. 반면 브로커 1대는 혼자서 과반수(1/1)를 충족한다.

또한 짝수 구성은 **Split-Brain** 위험이 있다.

```
브로커 4대, 네트워크 단절로 2대씩 분리

그룹A(2대): "우리가 과반수야" → 독립 운영
그룹B(2대): "우리가 과반수야" → 독립 운영

→ 두 그룹이 동시에 다른 데이터 씀
→ 네트워크 복구 후 데이터 충돌
```

홀수이면 네트워크 단절 시 한쪽만 과반수를 달성할 수 있어 Split-Brain을 방지한다. **브로커는 항상 홀수(3, 5, 7대)로 구성한다.**

---

## 2. Controller 선출 - Raft 알고리즘

KRaft에서 브로커 3대 중 1대가 **Active Controller**가 된다. 선출 방식은 Raft 알고리즘이다.

### Term(임기)

모든 브로커는 Term 번호를 가진다. 새 Controller가 선출될 때마다 1씩 증가한다.

```
초기 상태:
브로커1: Term=1, Active Controller
브로커2: Term=1, Follower
브로커3: Term=1, Follower
```

### 장애 시 선출 과정

```
브로커1 장애 발생

① 브로커2, 브로커3이 heartbeat 끊김 감지
② election timeout 발동 (랜덤 150~300ms)
③ 먼저 timeout된 브로커2가 후보 선언
   Term=1 → Term=2로 올리고 자신에게 투표
   브로커3에게 투표 요청
④ 브로커3 판단:
   "요청 Term(2) > 내 Term(1)" → 수락
   "이번 Term에 아직 투표 안 함" → 수락
⑤ 브로커2 2표 획득 → 과반수(2/3) 달성
⑥ 브로커2가 Term=2의 Active Controller 선출
```

### election timeout을 랜덤으로 쓰는 이유

고정값이면 두 브로커가 동시에 후보가 되어 서로 투표 거절 → 영원히 선출 실패한다. 랜덤값으로 한 브로커가 먼저 선출 과정을 시작하게 만든다.

### 브로커1 복구 후

```
브로커1 재시작 → Term=1로 클러스터 합류
브로커2: "현재 Term=2, 내가 Controller"
브로커1: Term=1 < Term=2 → 자동으로 Follower 합류
```

복구된 브로커는 Term이 낮아서 자동으로 Follower가 된다. **한번 선출된 Controller는 장애가 나기 전까지 계속 유지된다.**

---

## 3. Controller가 하는 일

Controller는 **파티션 Leader 명단을 관리**한다.

```
파티션0 Leader = 브로커1
파티션1 Leader = 브로커2
파티션2 Leader = 브로커3
파티션3 Leader = 브로커1
파티션4 Leader = 브로커2
```

이 정보를 **메타데이터**라고 한다. Controller는 메타데이터를 최신 상태로 유지하고, Producer/Consumer가 요청하면 알려준다.

**데이터를 중간에서 받아서 분배하는 역할이 아니다.** 데이터 전송은 Producer가 메타데이터를 보고 각 브로커에 직접 한다.

---

## 4. Producer가 데이터를 분배하는 방식

```
① Producer 시작 시 딱 한번
   → 아무 브로커에 메타데이터 요청
   → "파티션0=브로커1, 파티션1=브로커2..." 응답 수신
   → 로컬에 캐싱

② 17,000개 메시지 전송 시
   → Controller 거치지 않음
   → 캐싱된 메타데이터 보고 각 브로커에 직접 전송

siteId=00001 → 파티션0 → 브로커1 직접 전송
siteId=00002 → 파티션1 → 브로커2 직접 전송
siteId=00003 → 파티션2 → 브로커3 직접 전송
```

Controller는 교통 신호등(어디로 가라 알려주는 것)이고, 실제 데이터는 신호등을 보고 직접 이동한다.

---

## 5. 파티션 구조

파티션 5개, replication-factor 3이면:

```
파티션 5개 × 복제본 3개 = 총 15개 복사본

파티션0: 브로커1(Leader), 브로커2(Follower), 브로커3(Follower)
파티션1: 브로커2(Leader), 브로커1(Follower), 브로커3(Follower)
파티션2: 브로커3(Leader), 브로커1(Follower), 브로커2(Follower)
파티션3: 브로커1(Leader), 브로커2(Follower), 브로커3(Follower)
파티션4: 브로커2(Leader), 브로커1(Follower), 브로커3(Follower)
```

각 브로커는 파티션 5개씩 가지며, 일부는 Leader로 실제 처리하고 나머지는 Follower로 복제한다.

### 파티션 Leader 선정 기준

**최초 토픽 생성 시**: 라운드로빈으로 균등 배분

**장애 시**: ISR(In-Sync Replica) 목록에서 선정

ISR은 Leader와 동기화가 완료된 Follower 목록이다. 동기화가 밀린 브로커는 ISR에서 제외된다. 장애 시 ISR 중 첫 번째 브로커가 Leader로 승격한다.

```
파티션0 ISR: [브로커1, 브로커2, 브로커3]
브로커1 장애 → 브로커2 Leader 승격
```

ISR 외의 브로커는 최신 데이터가 없어서 Leader 후보가 될 수 없다.

---

## 6. Follower 복제 과정

Follower가 Leader한테 **주기적으로 당겨옵니다 (Pull 방식).**

```
브로커2(Follower): "파티션0 offset=5 이후 데이터 줘"
브로커1(Leader): "offset=6, 7, 8 여기있어"
브로커2: 저장
브로커2: "offset=8 이후 데이터 줘"
브로커1: "없어 (아직 안 들어옴)"
브로커2: 대기 후 다시 요청
```

### Producer acks 설정

```yaml
spring:
  kafka:
    producer:
      acks: all  # ISR 전체 복제 완료 후 응답 (가장 안전)
      acks: 1    # Leader 저장 완료 후 바로 응답 (빠르지만 유실 가능)
      acks: 0    # 응답 안 기다림 (가장 빠르지만 유실 가능성 높음)
```

Follower가 30초 이상 복제 못 하면 ISR에서 제외된다. 따라잡으면 다시 합류한다.

---

## 7. Kafka 시작 시 역할 결정 순서

```
docker-compose up
    ↓
브로커 3대 시작
    ↓
Raft 투표 → Controller 선출
    ↓
kafka-init 실행 → 토픽 생성 요청
    ↓
Controller가 파티션 Leader/Follower 배정 (라운드로빈)
    ↓
각 브로커에 역할 통보
    ↓
Follower들 Leader한테 Pull 시작
    ↓
Producer/Consumer 연결 준비 완료
```

이후 역할이 바뀌는 경우:
- 브로커 장애 → Controller가 Leader 재배정
- 파티션 수 변경 → Controller가 재배분

---

## 8. Group Coordinator

Consumer 파티션 배정은 **Group Coordinator**가 담당한다. Controller와 다른 역할이다.

| 역할 | 담당 | 하는 일 |
|---|---|---|
| Controller | 브로커 중 1대 | 파티션 Leader 관리 |
| Group Coordinator | 브로커 중 1대 | Consumer 파티션 배정 관리 |

Group Coordinator는 group-id 해시값으로 결정된다.

```
"mongo-consumer".hashCode() % 브로커 수 = 담당 브로커
```

group-id 이름에 따라 결정되므로 사실상 랜덤이다. 부하가 거의 없어서 어떤 브로커가 담당하든 성능에 영향이 없다.

---

## 9. Consumer Group

```
group-id: mongo-consumer
파티션 5개, Consumer 3대

Consumer1: 파티션0, 파티션1 담당
Consumer2: 파티션2, 파티션3 담당
Consumer3: 파티션4 담당
```

같은 group-id 안에서 파티션은 Consumer에게 **1:1로 배정**된다. 같은 파티션을 두 Consumer가 동시에 읽는 일이 없어 중복 처리가 없다.

Consumer가 파티션보다 많으면:

```
파티션 5개, Consumer 6대
→ Consumer 1대는 배정받을 파티션 없어서 대기
```

group-id가 다르면 완전히 독립된 offset을 가진다.

```
group-id: mongo-consumer   → 파티션0~4 독립적으로 읽음
group-id: table-consumer   → 파티션0~4 독립적으로 읽음
```

같은 데이터를 각자의 목적으로 처리할 수 있다.

---

## 10. Offset

**"어디까지 읽었는지 표시하는 번호"** 다.

```
파티션0:
offset:  0      1      2      3      4
메시지: [msg0] [msg1] [msg2] [msg3] [msg4]

Consumer가 offset=2까지 읽고 커밋
→ 죽었다 살아나도 offset=3부터 이어서 읽음
```

offset은 `__consumer_offsets` 토픽에 저장된다. 각 브로커는 자신이 Leader인 파티션의 offset을 관리하고, 장애 시 Follower가 Leader로 승격하면서 offset도 이어받는다.

### auto-offset-reset

Consumer가 처음 시작할 때 어디서 읽을지 결정한다.

```yaml
auto-offset-reset: earliest  # 처음부터 읽기 (기존 데이터 전부 처리)
auto-offset-reset: latest    # 지금부터 들어오는 것만 읽기
```

### At-Least-Once

```
offset=3 읽음 → MongoDB 저장 성공
offset 커밋 직전 → Consumer 죽음

재시작 후 offset=3 다시 읽음 → 중복 처리 가능
```

이를 방지하기 위해 `rowKey`를 MongoDB `@Id`로 사용해 중복 저장 시 upsert 처리한다.

---

## 11. 스케일아웃 공식

```
실제 병렬 처리 수 = min(파티션 수, 브로커 수, concurrency)
```

셋 중 하나라도 병목이 되면 나머지를 아무리 늘려도 효과가 없다.

```
파티션 5개, 브로커 10대, concurrency 5
→ 실제 병렬 처리 = 5

브로커 아무리 늘려도 파티션이 5개면 5대만 Leader
나머지 5대는 Follower(복제)만
```

올바른 스케일아웃:
```
브로커 추가 → 파티션 수 증가 → concurrency 증가
셋을 항상 같이 맞춰야 효과가 난다
```

---

## 12. 실측 성능 비교

17,000개 메시지, MongoDB single node 기준:

| 파티션 수 | elapsed | 개선율 |
|:---:|---:|---:|
| 1 | 18,587ms | - |
| 3 | 7,548ms | -59% |
| 4 | 6,223ms | -17% |
| **5** | **5,590ms** | **-10%** |
| 6 | 5,125ms | -8% |

5→6 구간에서 개선율이 8%로 수렴한다. MongoDB single node가 병목이라 파티션을 더 늘려도 효과가 줄어드는 것이다. **파티션 5개가 현재 인프라의 최적점이다.**

---

## 정리

| 개념 | 한줄 요약 |
|---|---|
| Controller | 파티션 Leader 명단 관리자 |
| Leader | 실제 읽기/쓰기 처리 |
| Follower | Leader 데이터 Pull 방식으로 복제 |
| ISR | Leader와 동기화 완료된 Follower 목록 |
| Group Coordinator | Consumer 파티션 배정 관리 |
| Offset | 어디까지 읽었는지 기록하는 번호 |
| Raft | 과반수 투표로 Controller 선출하는 알고리즘 |
| Split-Brain | 네트워크 단절로 두 그룹이 동시에 운영되는 현상 |
