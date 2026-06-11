---
layout: post
title: "Error Memory — 과거 에러 분석 사례를 RAG로 재활용하기"
date: 2026-06-11 12:00:00 +0900
categories: [AI, RAG]
tags: [RAG, ChromaDB, Python, LLM, Memory, 에러분석]
---

같은 서버에서 비슷한 에러가 반복되는 상황은 흔하다. 배포 직후 설정 오류가 여러 번 발생하거나, 특정 트래픽 패턴에서 항상 같은 NPE가 터지는 식이다.

현재 RAG 파이프라인은 매번 처음부터 분석한다. 과거에 같은 에러를 분석했다는 사실을 모른다.

**Error Memory**는 분석 결과를 벡터 DB에 저장해두고, 다음 유사 에러 발생 시 과거 사례를 LLM 프롬프트에 주입하는 구조다.

---

## 저장: 분석 완료 후 ChromaDB에 기록

기존 소스 인덱스(`server_{id}` 컬렉션)와 별개로 `memory_{id}` 컬렉션을 만들어 에러 로그와 분석 결과를 함께 저장한다.

```python
async def store_error_memory(server_id: int, error_query: str, analysis: str) -> None:
    embeddings = await _embed([error_query[:2000]])
    col = _memory_collection(server_id)
    doc_id = f"mem_{int(time.time() * 1000)}"
    col.upsert(
        ids=[doc_id],
        documents=[error_query[:2000]],    # 에러 로그 원문
        embeddings=embeddings,
        metadatas=[{"analysis": analysis[:3000]}],  # 분석 결과 JSON
    )
```

`analyze_log`가 유효한 JSON을 반환했을 때만 저장한다. 폴백 결과나 파싱 실패는 저장하지 않는다.

---

## 검색: 새 에러 발생 시 유사 사례 조회

에러 로그를 임베딩해서 메모리 컬렉션에서 코사인 유사도 검색을 한다.

```python
async def search_error_memory(server_id: int, query: str, n_results: int = 3) -> list[dict]:
    col = _memory_collection(server_id)
    if col.count() == 0:
        return []
    embeddings = await _embed([query[:2000]])
    results = col.query(query_embeddings=embeddings, n_results=min(n_results, col.count()))
    return [
        {"error": doc, "analysis": meta.get("analysis", "")}
        for doc, meta in zip(results["documents"][0], results["metadatas"][0])
    ]
```

---

## 주입: 과거 사례를 시스템 프롬프트에 추가

`analyze_log` 진입 시 메모리를 먼저 검색하고, 결과가 있으면 시스템 프롬프트 끝에 붙인다.

```python
past_cases = await rag_service.search_error_memory(server_id, raw_log[:1000])
system_prompt = SYSTEM_PROMPT
if past_cases:
    cases_text = "\n\n".join(
        f"[과거 사례 {i+1}]\n에러: {c['error'][:300]}\n분석 요약: {c['analysis'][:500]}"
        for i, c in enumerate(past_cases)
    )
    system_prompt = f"{SYSTEM_PROMPT}\n\n=== 과거 유사 에러 사례 (참고용) ===\n{cases_text}"
```

LLM은 과거 사례를 보고 "이 서버에서 비슷한 에러가 있었고 Redis 설정이 원인이었다"는 힌트를 얻어 분석 방향을 잡을 수 있다.

---

## 구조 정리

```
에러 발생
    ↓
search_error_memory() → 유사 과거 사례 3개
    ↓
시스템 프롬프트에 사례 주입
    ↓
LLM 분석 (소스 파일 검색 포함)
    ↓
유효한 결과면 store_error_memory() 저장
```

---

## 컬렉션을 분리한 이유

소스 인덱스(`server_{id}`)와 메모리(`memory_{id}`)를 같은 컬렉션에 넣으면 소스 재인덱싱 시 메모리까지 날아간다. 커밋이 바뀌면 소스 컬렉션을 통째로 삭제·재생성하는 구조이기 때문에 메모리는 반드시 별도로 관리해야 한다.

---

## 이후 개선 여지

현재는 유사도만으로 과거 사례를 가져온다. 동일 서버에서 동일 에러 클래스가 발생했다는 메타데이터 필터를 추가하면 노이즈를 줄일 수 있다. 또한 사용자가 "이 분석이 틀렸다"는 피드백을 주면 해당 메모리를 삭제하거나 가중치를 낮추는 방향으로 발전시킬 수 있다.
