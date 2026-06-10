---
layout: post
title: "BM25를 활용한 RAG 검색 품질 향상 — 벡터 검색의 한계와 하이브리드 접근"
date: 2026-06-08 10:00:00 +0900
categories: [AI, RAG]
tags: [BM25, RAG, ChromaDB, Python, LLM, 검색]
---

개인 프로젝트에서 RAG(Retrieval-Augmented Generation)를 구축하면서 벡터 검색만으로는 해결되지 않는 문제를 겪었다. LLM이 에러 스택 트레이스를 분석할 때 `UserService`, `findById` 같은 **정확한 클래스명·메서드명**을 언급하는데, 벡터 검색이 그 파일을 상위에 올려놓지 못하는 경우가 있었다.

이 문제를 BM25와 벡터 검색을 결합한 하이브리드 검색으로 해결했다.

---

## 벡터 검색의 한계

벡터 검색은 텍스트를 고차원 벡터로 변환하고 **코사인 유사도**로 비교한다.

```
"NullPointerException in UserService.findById"
→ [0.23, -0.81, 0.45, ...] (768차원 벡터)
→ ChromaDB에서 가장 가까운 벡터를 가진 문서 반환
```

의미적으로 유사한 문서를 잘 찾는다. "사용자 조회 실패"와 "UserService findById error"가 의미적으로 같다는 걸 벡터로 표현할 수 있다.

**그런데 문제가 있다.**

`UserService`라는 단어가 벡터 공간에서 반드시 `UserService.java`와 가장 가까운 건 아니다. 임베딩 모델은 단어의 의미를 통계적으로 학습한 것이라, **정확히 그 단어가 포함된 파일**을 보장하지 않는다.

예를 들어:
- 쿼리: `"UserService에서 NullPointerException 발생"`
- 벡터 검색 1위: `OrderService.java` (비슷한 패턴의 서비스 파일)
- 벡터 검색 3위: `UserService.java` (실제로 필요한 파일)

키워드 `UserService`가 그대로 들어있는 파일인데도 순위가 밀리는 상황이다.

---

## BM25란

BM25(Best Match 25)는 **키워드 빈도 기반 검색 알고리즘**이다. Elasticsearch, 구글 검색 초기 버전 등 전통적인 검색 엔진의 핵심 알고리즘으로, 특정 단어가 문서에 얼마나 많이 등장하는지(TF)와 그 단어가 전체 문서 중 얼마나 희귀한지(IDF)를 결합해 점수를 매긴다.

### TF — Term Frequency (단어 빈도)

단어가 문서에 많이 나올수록 관련도가 높다. 그러나 단순 빈도를 쓰면 긴 문서가 유리해진다. BM25는 이를 보정한다.

```
TF(t, d) = f(t,d) / (f(t,d) + k1 * (1 - b + b * |d| / avgdl))
```

- `f(t, d)`: 단어 t가 문서 d에 등장한 횟수
- `k1`: 빈도 포화 파라미터 (보통 1.2~2.0). 값이 클수록 빈도의 영향이 더 오래 지속됨
- `b`: 문서 길이 정규화 계수 (보통 0.75). 1이면 완전 정규화, 0이면 정규화 없음
- `|d|`: 문서 길이
- `avgdl`: 전체 문서 평균 길이

핵심은 **포화(saturation)** 개념이다. 단어가 1번 나오는 것과 10번 나오는 것은 다르지만, 100번과 200번의 차이는 거의 없다. BM25는 빈도가 늘어날수록 점수 증가폭이 줄어드는 구조다.

### IDF — Inverse Document Frequency (역문서 빈도)

```
IDF(t) = log((N - n(t) + 0.5) / (n(t) + 0.5) + 1)
```

- `N`: 전체 문서 수
- `n(t)`: 단어 t가 등장하는 문서 수

**흔한 단어는 가중치를 낮추고, 희귀한 단어는 가중치를 높인다.**

예시:
- `"the"`, `"is"`, `"import"` → 모든 문서에 등장 → IDF 낮음 → 검색에 별 도움 안 됨
- `"UserService"`, `"findById"` → 특정 파일에만 등장 → IDF 높음 → 검색에 크게 기여

### 최종 점수

```
BM25(t, d) = IDF(t) * TF(t, d)
```

쿼리에 여러 단어가 있으면 각 단어의 BM25 점수를 합산한다.

---

## 왜 BM25 + 벡터를 같이 써야 하나

| 상황 | 벡터 검색 | BM25 |
|---|---|---|
| `"UserService 오류"` → `UserService.java` 찾기 | 보통 (의미 유사한 다른 파일도 상위) | 강함 (정확한 키워드 매칭) |
| `"사용자 조회 실패"` → `UserService.java` 찾기 | 강함 (의미 이해) | 약함 ("사용자", "조회"가 없으면 못 찾음) |
| 오타나 변형 | 강함 (임베딩이 흡수) | 약함 (다른 토큰으로 처리) |
| 정확한 메서드명 매칭 | 약함 | 강함 |

두 방식은 서로 보완적이다. 하나가 놓치는 걸 다른 하나가 잡는다.

---

## RRF — Reciprocal Rank Fusion

두 검색 결과를 합칠 때 문제가 생긴다. **점수 스케일이 다르다.**

- 벡터 검색: 코사인 유사도 (0~1)
- BM25: 빈도 기반 점수 (범위 없음, 문서 집합마다 다름)

단순히 더할 수 없다. 코사인 유사도 0.95와 BM25 점수 12.3을 어떻게 비교하나?

**RRF는 점수 대신 순위(rank)를 사용한다.**

```
RRF(d) = Σ 1 / (k + rank_i(d))
```

- `rank_i(d)`: i번째 검색 방식에서 문서 d의 순위 (1위, 2위, 3위...)
- `k`: 상위 문서 편향 방지 상수 (보통 60)

1위 문서: `1 / (60 + 1) ≈ 0.0164`
10위 문서: `1 / (60 + 10) ≈ 0.0143`
100위 문서: `1 / (60 + 100) ≈ 0.0063`

벡터 검색과 BM25 각각에서 순위를 매기고, 두 점수를 합산한다. **양쪽 검색 모두에서 상위에 오른 문서가 최종 상위에 위치한다.**

예시:

| 파일 | 벡터 순위 | BM25 순위 | RRF 점수 |
|---|---|---|---|
| `UserService.java` | 3위 | 1위 | 1/(61+3) + 1/(61+1) = 0.0156 + 0.0164 = **0.0320** |
| `OrderService.java` | 1위 | 8위 | 1/(61+1) + 1/(61+8) = 0.0164 + 0.0145 = **0.0309** |
| `BaseService.java` | 2위 | 20위 | 1/(61+2) + 1/(61+20) = 0.0159 + 0.0123 = **0.0282** |

`UserService.java`가 BM25에서 1위를 했기 때문에 최종 1위가 된다.

`k=60`인 이유는 상위 문서들의 점수 차이를 완화하기 위해서다. k가 작으면 1위와 2위 차이가 너무 크고, 크면 순위가 거의 의미 없어진다. 60은 실험적으로 검증된 기본값이다.

---

## 구현

```python
# rag_service.py
from rank_bm25 import BM25Okapi

def _tokenize(text: str) -> list[str]:
    """영문·한글·숫자 토큰 추출 (소문자 정규화)."""
    return re.findall(r"[A-Za-z가-힣0-9]+", text.lower())


async def search_relevant_files(server_id: int, query: str, n_results: int = 5) -> list[str]:
    col = _collection(server_id)
    count = col.count()
    if count == 0:
        return []

    candidates = min(n_results * 4, count)

    # 1단계: 벡터 검색
    query_embeddings = await _embed([query[:2000]])
    vec_results = col.query(
        query_embeddings=query_embeddings,
        n_results=candidates,
    )
    vec_items = list(zip(vec_results["metadatas"][0], vec_results["documents"][0]))

    # 2단계: BM25 검색
    all_data = col.get(limit=min(count, 2000), include=["documents", "metadatas"])
    all_docs = all_data["documents"]
    all_metas = all_data["metadatas"]

    tokenized_corpus = [_tokenize(doc) for doc in all_docs]
    bm25 = BM25Okapi(tokenized_corpus)
    bm25_scores = bm25.get_scores(_tokenize(query))
    bm25_top_idx = sorted(range(len(bm25_scores)), key=lambda i: bm25_scores[i], reverse=True)[:candidates]
    bm25_items = [(all_metas[i], all_docs[i]) for i in bm25_top_idx]

    # 3단계: RRF 융합 (k=60)
    K = 60
    rrf: dict[str, float] = {}
    for rank, (meta, _) in enumerate(vec_items):
        path = meta["path"]
        rrf[path] = rrf.get(path, 0.0) + 1 / (K + rank + 1)
    for rank, (meta, _) in enumerate(bm25_items):
        path = meta["path"]
        rrf[path] = rrf.get(path, 0.0) + 1 / (K + rank + 1)

    sorted_paths = sorted(rrf, key=lambda p: rrf[p], reverse=True)

    seen: set[str] = set()
    paths: list[str] = []
    for path in sorted_paths:
        if path not in seen:
            seen.add(path)
            paths.append(path)
        if len(paths) >= n_results:
            break

    return paths
```

### 포인트

**`candidates = n_results * 4`**: 두 검색 모두 충분한 후보를 가져와야 RRF가 의미 있다. 최종 5개가 필요하면 각 검색에서 20개씩 가져온다.

**`col.get(limit=2000)`**: BM25는 전체 코퍼스를 메모리에 올려야 하므로 상한을 둔다. 소스코드 2000 청크면 일반적인 프로젝트를 커버하기 충분하다.

**`_tokenize`**: 정규식으로 영문·한글·숫자만 추출한다. `{`, `(`, `.` 같은 특수문자는 제거한다. `UserService.findById`가 들어오면 `["userservice", "findbyid"]`가 된다.

---

## 결과

하이브리드 검색 도입 전후 차이는 스택 트레이스에 정확한 클래스명이 포함된 경우에서 두드러진다.

- **전**: 벡터 검색이 의미적으로 유사한 다른 서비스 파일을 1~2위로 올리고, 정작 필요한 파일이 3~5위에 위치
- **후**: BM25가 정확한 클래스명으로 해당 파일을 1위로 올리고, RRF가 이를 최종 결과에 반영

추상적인 에러 메시지("사용자를 찾을 수 없습니다")는 여전히 벡터 검색이 잘 처리하고, 구체적인 식별자(`UserService`, `findById`, `user_id`)는 BM25가 잡아낸다. 두 방식이 서로의 약점을 보완하는 구조다.
