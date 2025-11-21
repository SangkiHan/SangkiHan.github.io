---
layout: post
title:  "Azure Function App 비용 80% 절감하기 - EP3에서 EP1으로 Scale Down"
date:   2025-11-01 10:00:00 +0900
categories: [Cloud, Cost Optimization]
tags: [Azure, Function App, Cost, Performance, Cloud]
---

회사에서 Azure Function App을 사용한 서버리스 아키텍처를 운영하고 있었다.   
인프라는 관리 하지 않았지만 인프라 담당자의 퇴사로 인프라 관리까지 업무를 맡게 되었다.  
어느 날 Azure 청구서를 확인하고 깜짝 놀랐다.

"이렇게 비용이 많이 나온다고...?"

## 문제 발견: 과도한 스펙

### 9월 청구 내역
![9월 요금](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/9월 요금.png)

9월 Function App 비용을 확인해보니 **상당한 금액**이 청구되고 있었다.  
9월 Function App VM의 사용료만 약 1500달러...  
실제 사용량을 분석해보기로 했다.

### 리소스 사용률 분석

#### CPU 사용량
![기존 CPU 사용량](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/기존 CPU 사용량.png)

EP3 스펙에서 CPU 사용률을 확인해보니 **평균 3%** 수준. 피크 타임에도 5%를 넘지 않았다.

#### 메모리 사용량
![기존 메모리 사용량](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/기존 메모리 사용량.png)

메모리도 마찬가지였다. 할당된 메모리의 절반도 사용하지 않고 있었다.

#### 인스턴스 수
![인스턴스 수](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/인스턴스 수.png)

자동 스케일링 설정으로 20개까지 늘어날 수 있었지만, 실제로는 **1개**만 사용하고 있었다.

## Premium Plan 가격 비교

![Function App 프리미엄 플랜 가격](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/Function app 프리미엄 플랜 가격.png)

Azure Function App Premium Plan의 가격표를 확인해봤다:

| Plan | vCPU | 메모리 | 예상 월 비용 |
|------|------|--------|--------------|
| **EP1** | 1 | 3.5 GB | ~$150 |
| **EP2** | 2 | 7 GB | ~$300 |
| **EP3** | 4 | 14 GB | ~$600 |

현재 사용 중인 EP3는 4 vCPU, 14GB 메모리를 제공하지만, 실제로는 그 성능의 10%도 사용하지 않고 있었다.

## 해결책: EP1으로 Scale Down

### 결정 과정

1. **리소스 사용률 분석**
   - CPU: 평균 3~5%, 최대 5%
   - 메모리: 30% 미만 사용
   - 결론: EP1 (1 vCPU, 3.5GB)로도 충분
   - 부족하더라도 CPU가 70% 이상 사용 시 Function App에서 자동으로 Scale Out 가능

2. **성능 영향 검토**
   - 예상 영향: EP1으로 변경해도 성능 저하 거의 없을 것으로 판단
   - 만약 부족하면 EP2로 다시 올리면 됨

3. **비용 절감 효과**
   - EP3: 약 $600/월
   - EP1: 약 $150/월
   - **절감액: 약 $450/월 (75% 절감)**

### Scale Down 실행
Azure Portal에서:
1. Function App → Configuration → Scale up
2. EP3 → **EP1** 선택
3. Apply

## 결과: 비용 37% 절감 달성

### 10월 청구 내역
![10월 요금](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/10월 요금.png)

EP1으로 변경한 후 10월 청구 내역을 확인해보니 **비용이 대폭 감소**했다!   
작년과 비교하면 1100달러 한화 약 140만원이 절감되었다.

### 성능 모니터링

#### EP1 변경 후 CPU 사용량
![EP1 변경 후 CPU 사용량](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/EP1 변경 후 CPU사용량.png)

EP1으로 변경 후에도 CPU 사용률은 **평균 5~15%** 수준으로 안정적으로 운영되고 있다.

#### EP1 변경 후 메모리 사용량
![EP1 변경 후 메모리 사용량](/assets/img/post/2025-10-20-Azure-FunctionApp-Cost/EP1 변경 후 CPU 메모리 사용량.png)

메모리도 3.5GB 중 **충분한 여유**를 가지고 운영 중이다.

## 교훈 및 권장사항

### 1. 클라우드 비용은 정기적으로 점검하자

개발 환경에서 설정한 스펙을 그대로 프로덕션에 올리면 **과도한 비용**이 발생할 수 있다. 최소 월 1회는 비용 리포트를 확인하고, 리소스 사용률을 분석해야 한다.

### 2. 실제 사용률 기반으로 스펙 결정

Azure Monitor, CloudWatch 등의 모니터링 도구로 **실제 CPU/메모리 사용률**을 확인하고, 그에 맞는 적절한 스펙을 선택하자.

### 3. 스케일링 전략 수립

Premium Plan의 장점은 **자동 스케일링**이다. 하지만 최소 인스턴스 수가 많으면 기본 비용도 증가한다.

최소 인스턴스를 1~2개로 설정하고, 부하 시에만 자동으로 증가하도록 구성하면 비용을 더 절감할 수 있다.

### 4. 단계적 Scale Down

한 번에 EP3 → EP1으로 변경하는 것이 불안하다면:

1. EP3 → **EP2**로 먼저 변경
2. 1주일간 모니터링
3. 문제 없으면 EP2 → **EP1**로 추가 변경

이런 식으로 단계적으로 Scale Down하면 리스크를 최소화할 수 있다.

## 마무리

클라우드는 **유연성**이 장점이지만, 방치하면 **비용 폭탄**이 될 수 있다. 이번 경험을 통해 다음을 깨달았다:

1. 🔍 **정기적인 모니터링**이 필수
2. 📊 **실제 사용률 기반 의사결정**
3. 💰 **비용 대비 성능 최적화**
4. 🚀 **Scale Down도 하나의 최적화 전략**

EP3 → EP1 변경으로 **월 $1100 (약 140만원)** 절감에 성공했다. 연간으로 계산하면 **약 1,680만원**이다.
