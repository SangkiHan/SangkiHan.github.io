---
layout: post
title: "Terraform으로 AWS ECS 서비스 배포하기 — 시리즈 소개와 전체 그림"
date: 2026-10-06 09:00:00 +0900
categories: [DevOps, Terraform 구축]
tags: [Terraform, AWS, ECS, Fargate, ALB, IaC]
mermaid: true
---

Terraform을 제대로 써본 적이 없어서, 실제 MSA 구조를 흉내 낸 테스트 프로젝트를 만들고 AWS ECS(Fargate)에 올려보기로 했다. 코드로 인프라를 만들고, 서비스를 띄우고, 브라우저에서 응답을 확인하고, 다시 전부 지우는 것까지가 목표다. 이 포스트는 그 과정 전체를 요약하고, 시리즈 전체의 인덱스 역할을 한다.

---

## 무엇을 만들었나

서비스 6개짜리 가짜 MSA다. **로직은 전혀 없고**, 루트(`/`)로 접근하면 자기 이름만 텍스트로 돌려준다.

| 서비스 | 포트 | 응답 |
|---|---|---|
| gateway | 9000 | `gateway-service` |
| chatting | 8080 | `chatting-service` |
| session | 8081 | `session-service` |
| user | 8082 | `user-service` |
| push | 8084 | `push-service` |
| push-worker | 8085 | `push-worker-service` |

앱이 단순한 이유는 단 하나, **Terraform을 테스트하기 위해서**다. 앱 로직에 신경 쓰면 인프라 학습이 흐려진다.

그리고 gateway는 Spring Cloud Gateway라서 `/user`, `/chatting` 같은 경로를 각 서비스로 전달한다. 결국 브라우저에서 `http://<ALB 주소>/user`를 치면 `user-service`가 나오는 구조다.

---

## 전체 아키텍처

```mermaid
flowchart LR
    U[브라우저] -->|HTTP :80| ALB[ALB]
    ALB --> GW[gateway :9000]
    GW -->|Service Connect| US[user :8082]
    GW --> CH[chatting :8080]
    GW --> SE[session :8081]
    GW --> PU[push :8084]
    WK[push-worker :8085]
    subgraph VPC
      subgraph 퍼블릭 서브넷
        ALB
        NAT[NAT Gateway]
      end
      subgraph 프라이빗 서브넷
        GW
        US
        CH
        SE
        PU
        WK
      end
    end
    프라이빗 서브넷 -.->|이미지 pull / 로그| NAT
    ECR[(ECR)] -.-> 프라이빗 서브넷
```

- **외부에 노출되는 건 ALB뿐**이다. 서비스는 전부 프라이빗 서브넷에 있다.
- ALB는 gateway 하나만 바라본다. 나머지 서비스는 gateway가 **Service Connect**라는 ECS 기능으로 이름(`http://user:8082`)을 써서 호출한다.
- 컨테이너 이미지는 ECR에 두고, ECS가 NAT Gateway를 통해 받아온다.

---

## 시리즈 구성

1. **시리즈 소개와 전체 그림** (이 글)
2. [테스트용 MSA 앱 만들기 — 로직 없이 이름만 돌려주는 6개 서비스](/posts/terraform-ecs-app)
3. [Terraform 코드 구조 설계 — global, envs, modules를 나눈 이유](/posts/terraform-ecs-structure)
4. [Terraform 실행 준비 — AWS 로그인, state 버킷, global 적용](/posts/terraform-ecs-global-apply)
5. [이미지 올리고 서비스 띄우기 — dev 적용과 트러블슈팅](/posts/terraform-ecs-deploy)

각 글은 실제로 겪은 에러와 해결 과정을 그대로 담았다. 성공 경로만 정리한 글은 이미 많으니, 이 시리즈는 **어디서 막히는지**에 무게를 뒀다.

---

## 사용한 도구와 버전

| 항목 | 값 |
|---|---|
| Terraform | 1.10 이상 (S3 state 락에 필요) |
| AWS Provider | `~> 6.4` (실제 설치 6.67) |
| 앱 | Java 25, Spring Boot 4.1, Gradle 9.6 |
| 이미지 빌드 | Jib (Docker 없이 빌드·push) |
| 리전 | ap-northeast-2 (서울) |

---

## 결과 미리보기

모든 적용이 끝나고 ALB 주소로 접근하면 이렇게 나온다.

```
$ curl http://<ALB 주소>/
gateway-service
$ curl http://<ALB 주소>/user
user-service
$ curl http://<ALB 주소>/chatting
chatting-service
```

Terraform 코드는 총 약 67개의 AWS 리소스(VPC, 서브넷, NAT, ALB, 보안그룹, IAM, ECS 클러스터와 서비스 등)를 만든다.

> 이 구성은 NAT Gateway와 ALB가 켜져 있는 동안 계속 과금된다. 테스트가 끝나면 반드시 `terraform destroy`로 지운다. 이 부분은 마지막 글에서 다룬다.
{: .prompt-warning }

다음 글에서는 가장 먼저 테스트용 앱을 만든다.
