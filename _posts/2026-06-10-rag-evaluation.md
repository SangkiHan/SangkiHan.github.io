---
layout: post
title: "RAG 검색 품질을 어떻게 측정할까 — Hit Rate, MRR, 평가 시스템 구축"
date: 2026-06-10 10:00:00 +0900
categories: [AI, RAG]
tags: [RAG, 평가, HitRate, MRR, ChromaDB, Python]
---

BM25 하이브리드 검색을 도입하고 나서 자연스럽게 궁금증이 생겼다. **"검색이 실제로 잘 되고 있는 걸까?"** 로그를 보면 파일 목록이 반환되는 건 확인되지만, 그게 맞는 파일인지 판단할 기준이 없었다.

RAG 품질을 수치로 측정하는 평가 시스템을 만든 이유다.

---

## 무엇을 측정하나

RAG 평가에서 핵심은 **"정답 파일이 검색 결과에 포함됐는가"**다. 여기서 두 가지 지표를 사용한다.

### Hit Rate@K

상위 K개 결과 안에 정답 파일이 있으면 1, 없으면 0.

```
Hit Rate@1 = 1위가 정답인 비율
Hit Rate@3 = 상위 3개 안에 정답이 있는 비율
Hit Rate@5 = 상위 5개 안에 정답이 있는 비율
```

K가 클수록 관대한 기준이다. LLM에게 파일 3개를 줄 거라면 Hit Rate@3이 가장 중요한 지표가 된다.

### MRR — Mean Reciprocal Rank

정답이 몇 위에 있는지를 수치화한다.

```
RR = 1 / 정답_순위

1위 = 1/1 = 1.000
2위 = 1/2 = 0.500
3위 = 1/3 = 0.333
5위 = 1/5 = 0.200
없음 = 0.000

MRR = 전체 케이스의 RR 평균
```

Hit Rate는 "있냐 없냐"만 보는데, MRR은 "얼마나 위에 있냐"까지 본다. 1위와 5위는 Hit Rate@5 기준으로는 같지만 MRR은 다르다.

---

## 평가 시스템 구조

```
TestCase
  ├── stack_trace (검색 쿼리)
  └── expected_files (정답 파일 경로)

평가 실행
  → stack_trace로 RAG 검색
  → 검색 결과 vs expected_files 비교
  → Hit Rate@K, RR 계산
  → DB 저장 (eval_runs, eval_cases, eval_retrieved)
```

DB에는 검색 결과 파일 내용까지 저장한다. 나중에 "이 평가에서 어떤 파일이 몇 위로 검색됐는지"를 다시 확인할 수 있다.

```python
# rag_eval.py 핵심
async def evaluate(server_id, test_cases, n_results=5):
    for tc in test_cases:
        retrieved = await search_relevant_files(server_id, tc.stack_trace, n_results)
        
        result = CaseResult(
            hit_at_1 = _hit_at_k(retrieved, tc.expected_files, 1),
            hit_at_3 = _hit_at_k(retrieved, tc.expected_files, 3),
            hit_at_5 = _hit_at_k(retrieved, tc.expected_files, 5),
            rr       = _reciprocal_rank(retrieved, tc.expected_files),
        )
    
    # DB 저장 (eval_runs → eval_cases → eval_retrieved)
    ...
```

API 3개로 구성했다:

| 엔드포인트 | 역할 |
|---|---|
| `POST /api/v1/eval/rag/{server_id}` | 평가 실행 + DB 저장 |
| `GET /api/v1/eval/rag/{server_id}/history` | 실행 이력 조회 |
| `GET /api/v1/eval/rag/run/{run_id}` | 특정 실행 상세 조회 |

---

## 테스트 케이스

평가에는 **정답지**가 필요하다. 스택 트레이스와 그 에러가 발생한 실제 파일 경로를 쌍으로 정의한다.

```python
_TEST_CASES = [
    TestCase(
        name="NullPointerException in TestErrorController",
        stack_trace=(
            "java.lang.NullPointerException: Cannot invoke \"String.toUpperCase()\" because \"value\" is null\n"
            "\tat com.puppynoteserver.global.TestErrorController.triggerNpe(TestErrorController.java:21)"
        ),
        expected_files=["src/main/java/com/puppynoteserver/global/TestErrorController.java"],
    ),
    TestCase(
        name="NoSuchElementException in UserServiceImpl",
        stack_trace=(
            "java.util.NoSuchElementException: User not found\n"
            "\tat com.puppynoteserver.user.users.service.impl.UserServiceImpl.findById(UserServiceImpl.java:34)"
        ),
        expected_files=["src/main/java/com/puppynoteserver/user/users/service/impl/UserServiceImpl.java"],
    ),
    TestCase(
        name="DataIntegrityViolationException in CommunityPostWriteService",
        stack_trace=(
            "org.springframework.dao.DataIntegrityViolationException: could not execute statement\n"
            "\tat com.puppynoteserver.community.post.service.impl.CommunityPostWriteServiceImpl.create(CommunityPostWriteServiceImpl.java:52)"
        ),
        expected_files=["src/main/java/com/puppynoteserver/community/post/service/impl/CommunityPostWriteServiceImpl.java"],
    ),
]
```

처음에는 패키지 경로를 추측해서 넣었다가 0점이 나왔다. 실제 repo 구조를 확인하고 나서야 정확한 경로로 수정할 수 있었다. **테스트 케이스 품질이 평가 품질을 결정한다.**

---

## 실제 측정 결과

```json
{
  "hit_rate@1": 0.667,
  "hit_rate@3": 0.667,
  "hit_rate@5": 1.000,
  "mrr": 0.733
}
```

케이스별로 보면:

| 케이스 | 정답 파일 | 검색 순위 | RR |
|---|---|---|---|
| TestErrorController NPE | TestErrorController.java | **1위** | 1.000 |
| UserServiceImpl NSEE | UserServiceImpl.java | **5위** | 0.200 |
| CommunityPostWriteService DIV | CommunityPostWriteServiceImpl.java | **1위** | 1.000 |

**케이스 1, 3**: 1위로 정확히 찾았다. BM25가 클래스명을 키워드로 직접 매칭한 덕분이다.

**케이스 2**: 정답 파일이 5위에 있다. 스택 트레이스에 `UserServiceImpl`이 명시되어 있는데도 `UserReadService`가 1위로 올라왔다. 이는 puppynote-server의 구조상 실제 `findById` 로직이 `UserReadService` 계열에 분산되어 있어서 벡터 유사도가 더 높게 측정되기 때문이다. `UserServiceImpl`이 5위 안에는 들어오므로 Hit Rate@5는 100%가 된다.

---

## 이 숫자가 의미하는 것

**Hit Rate@5 = 1.0**: 상위 5개 결과 안에 항상 정답 파일이 포함된다. LLM에게 5개 파일을 주는 현재 설정에서 정답 파일이 누락되는 경우는 없다.

**Hit Rate@1 = 0.667**: 1위가 정답인 경우는 2/3. LLM이 1번만 검색하고 끝낸다면 1/3 확률로 핵심 파일을 바로 못 찾는다.

**MRR = 0.733**: 평균적으로 정답 파일이 1~2위 사이에 있다.

현재 에이전트는 `n_results=3`으로 검색한다. 이 설정이라면 Hit Rate@3 = 0.667이 실질적인 성능 지표다. 케이스 2가 3위 안에 들어오지 못하는 게 개선 포인트다.

---

## 평가 시스템이 있으면 무엇이 달라지나

숫자가 있으면 변경 전후 비교가 가능하다.

```
BM25 도입 전  → Hit Rate@5: ?  (측정 안 함)
BM25 도입 후  → Hit Rate@5: 1.0

n_results 3→5 변경 시  → Hit Rate@3이 어떻게 달라지는지 측정 가능
청크 크기 변경 시       → MRR이 올라가는지 확인 가능
```

"더 좋아진 것 같은데"가 아니라 "0.667에서 0.833으로 올랐다"고 말할 수 있게 된다.
