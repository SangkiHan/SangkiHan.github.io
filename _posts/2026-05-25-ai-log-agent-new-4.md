---
layout: post
title: "AI 오류 자동 분석 에이전트 구축기 (4) — Reranker 모델 도입"
date: 2026-05-25 13:00:00 +0900
categories: [Project, AI]
tags: [Reranker, RAG, ChromaDB, Python, LLM]
---

벡터 검색(코사인 유사도)만으로는 의미적으로 가장 관련된 파일을 정확히 골라내기 어렵다. Reranker가 그 한계를 보완한다.

## 벡터 검색의 한계

ChromaDB의 코사인 유사도 검색은 임베딩 벡터 방향의 유사도를 측정한다. 단어 빈도나 표면적 유사도에 강하고 빠르다.

하지만 문제가 있다.

**예시**: 에러가 `UserService`에서 발생했는데 벡터 검색 결과 상위에 `UserDto`, `UserController`가 올라오는 경우. 임베딩 공간에서 비슷한 단어가 많지만 실제 관련도는 낮다.

| 특성 | 벡터 검색 | Reranker |
|---|---|---|
| 속도 | 매우 빠름 | 느림 |
| 의미 이해 | 얕음 (표면 유사도) | 깊음 (cross-attention) |
| 대규모 탐색 | 적합 | 비적합 |
| 정밀 재순위화 | 부적합 | 적합 |

**전략**: 벡터 검색으로 후보 15~25개를 빠르게 가져온 뒤, Reranker로 실제 관련도 기준으로 재순위화해서 상위 3~5개만 사용한다.

---

## qllama/bge-reranker-v2-m3

Ollama에서 실행하는 rerank 전용 모델이다. 쿼리와 문서 쌍을 cross-encoder 방식으로 평가해서 `relevance_score`를 반환한다.

```
Reranker: query ↔ doc 쌍을 한번에 읽고 점수 계산
Embedder: query와 doc를 각각 독립적으로 벡터화
```

Cross-encoder는 쿼리와 문서를 동시에 보기 때문에 둘의 상호작용을 파악할 수 있다. 임베딩 기반 검색보다 정확하지만 전체 문서를 매번 읽어야 해서 느리다.

---

## 구현

### _rerank 함수

```python
# rag_service.py
async def _rerank(query: str, documents: list[str]) -> list[int]:
    """Reranker로 후보 문서 재순위화. 관련도 높은 순 index 목록 반환."""
    try:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.post(
                f"{settings.ollama_host}/api/rerank",
                json={
                    "model": settings.ollama_rerank_model,
                    "query": query,
                    "documents": documents,
                },
            )
            resp.raise_for_status()
            results = resp.json().get("results", [])
            # relevance_score 내림차순으로 index 목록 반환
            return [
                r["index"]
                for r in sorted(results, key=lambda x: x["relevance_score"], reverse=True)
            ]
    except Exception as e:
        logger.warning("[rag] rerank failed (fallback to vector order): %s", e)
        return list(range(len(documents)))  # 실패 시 원래 순서 유지
```

Reranker가 실패해도 폴백으로 벡터 검색 순서를 그대로 사용해서 시스템이 멈추지 않는다.

Ollama `/api/rerank` 응답 예시:
```json
{
  "results": [
    {"index": 2, "relevance_score": 0.94},
    {"index": 0, "relevance_score": 0.71},
    {"index": 1, "relevance_score": 0.35}
  ]
}
```

---

### search_relevant_files: 전체 흐름

```python
async def search_relevant_files(server_id: int, query: str, n_results: int = 5) -> list[str]:
    col = _collection(server_id)
    count = col.count()
    if count == 0:
        return []

    query_embeddings = await _embed([query[:2000]])

    # 1단계: 벡터 검색으로 후보 넉넉하게 (n_results * 5)
    candidates = min(n_results * 5, count)  # 최대 25개
    results = col.query(
        query_embeddings=query_embeddings,
        n_results=candidates,
    )

    all_docs = list(zip(results["metadatas"][0], results["documents"][0]))

    # 2단계: Reranker로 실제 관련도 재순위화
    ranked_indices = await _rerank(query, [doc for _, doc in all_docs])

    # 3단계: 파일 경로 기준 중복 제거 (청크 단위 → 파일 단위)
    seen: set[str] = set()
    paths: list[str] = []
    for idx in ranked_indices:
        path = all_docs[idx][0]["path"]
        if path not in seen:
            seen.add(path)
            paths.append(path)
        if len(paths) >= n_results:
            break

    return paths
```

한 파일이 여러 청크로 분할되어 있으므로 중복 제거 후 파일 경로만 반환한다.

---

## LangGraph 에이전트와의 연결

Reranker가 반환한 파일 경로를 LLM 에이전트의 `search_files` 도구가 활용한다.

```python
# ollama_service.py
def _make_search_tool(server_id: int, repo_path: Path):
    @langchain_tool
    async def search_files(query: str) -> str:
        """에러의 근본 원인과 관련된 소스 파일을 검색하고 내용을 반환합니다."""
        # RAG + Reranker로 정렬된 파일 경로 3개
        paths = await rag_service.search_relevant_files(server_id, query, n_results=3)
        results = {}
        for path in paths:
            content = (repo_path / path).read_text(encoding="utf-8", errors="replace")
            results[path] = content[:2000]
        return json.dumps(results, ensure_ascii=False)
    return search_files
```

LLM이 `search_files("UserService NullPointerException")`을 호출하면:
1. nomic-embed-text로 쿼리 임베딩
2. ChromaDB에서 코사인 유사도 상위 15개 청크 검색
3. bge-reranker-v2-m3로 실제 관련도 재순위화
4. 상위 3개 파일 경로 반환
5. 해당 파일 내용을 LLM에게 전달

---

## 통신 흐름 상세

```
[Before — Reranker 없는 RAG]

에러 쿼리
    │
    ▼
nomic-embed-text (임베딩)
    │
    ▼
ChromaDB 코사인 유사도 검색
    │ 상위 5개 파일 반환
    │ 표면적 유사도 기준 — 틀릴 수 있음
    ▼
LLM 분석

[After — Reranker 추가]

puppynote-server     FastAPI(노트북)      ChromaDB     Ollama(M4)
     │ 에러 발생        │                   │              │
     │──POST /error──▶ │                   │              │
     │                 │                   │              │
     │                 │  search_files 호출 (LangGraph tool call)
     │                 │──embed query─────▶ nomic-embed   │
     │                 │──cosine search───▶ │              │
     │                 │◀── 25개 후보 ───── │              │
     │                 │──rerank──────────────────────────▶│
     │                 │                           bge-reranker-v2-m3
     │                 │◀─ relevance_score 순 정렬 ────────│
     │                 │ 상위 3개 파일 내용을 LLM에 전달   │
     │                 │──────────────────────────────────▶│ gemma4:12b
     │                 │◀──────── JSON 분석 결과 ─────────-│
     │                 │──Slack 전송                        │

M4 Mac Mini에서 세 모델이 순차적으로 동작:
  1. nomic-embed-text: 쿼리/문서 임베딩
  2. qllama/bge-reranker-v2-m3: 후보 재순위화
  3. gemma4:12b: 에러 분석 및 수정 제안

ChromaDB + FastAPI는 개인 노트북 서버(4Core 8G, Docker Compose).
```
