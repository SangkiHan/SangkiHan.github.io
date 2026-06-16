---
title: "AI 개인 비서에 LlamaIndex 도입 — 기억 검색부터 Knowledge Graph까지"
author: SangkiHan
date: 2026-06-16 20:00:00 +0900
categories: [AI, 프로젝트]
tags: [LlamaIndex, LangGraph, ChromaDB, RAG, KnowledgeGraph, 개인비서]
---

Slack 기반 AI 개인 비서에 LlamaIndex를 도입했다. 기억 저장/검색 파이프라인을 교체하고, 대화 히스토리 압축과 Knowledge Graph를 추가했다.

---

## 왜 LlamaIndex인가

기존 `memory_service.py`는 LlamaIndex 없이 직접 구현되어 있었다.

```python
# AS-IS: 수동 구현
async def _embed(texts):
    async with httpx.AsyncClient() as client:
        resp = await client.post(f"{ollama_host}/api/embed", json={"model": ..., "input": texts})
        return resp.json()["embeddings"]

# 검색도 ChromaDB 직접 호출 + rank_bm25 직접 초기화 + 수동 RRF 계산
```

동작은 했지만 코드가 길었다. LlamaIndex는 이 파이프라인을 컴포넌트 조합으로 단순화해준다. 또 Knowledge Graph나 대화 압축 같은 기능을 추가할 때 바퀴를 다시 발명하지 않아도 된다.

---

## 1. memory_service — VectorStoreIndex + QueryFusionRetriever

### AS-IS

```python
# 임베딩: httpx로 Ollama 직접 호출
async def _embed(texts: list[str]) -> list[list[float]]:
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(f"{settings.ollama_host}/api/embed", ...)
        return resp.json()["embeddings"]

# BM25: rank_bm25 직접 초기화
tokenized_corpus = [_tokenize(doc) for doc in all_docs]
bm25 = BM25Okapi(tokenized_corpus)
bm25_scores = bm25.get_scores(_tokenize(query))

# RRF: 수동 계산
K = 60
rrf: dict[str, float] = {}
for rank, (doc_id, doc) in enumerate(zip(vec_ids, vec_docs)):
    rrf[doc_id] = rrf.get(doc_id, 0.0) + 1 / (K + rank + 1)
for rank, idx in enumerate(bm25_top_idx):
    rrf[doc_id] = rrf.get(doc_id, 0.0) + 1 / (K + rank + 1)
```

### TO-BE

```python
# LlamaIndex 전역 임베딩 설정 (한 번만)
from llama_index.core import Settings as LlamaSettings
from llama_index.embeddings.ollama import OllamaEmbedding

LlamaSettings.embed_model = OllamaEmbedding(
    model_name=settings.ollama_embed_model,
    base_url=settings.ollama_host,
)
LlamaSettings.llm = None  # 임베딩만 사용

# ChromaVectorStore + VectorStoreIndex
vector_store = ChromaVectorStore(chroma_collection=collection)
index = VectorStoreIndex.from_vector_store(vector_store, storage_context=storage_context)

# 검색: QueryFusionRetriever가 BM25 + 벡터 RRF를 대신 처리
retriever = QueryFusionRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    similarity_top_k=n,
    mode="reciprocal_rerank",  # RRF
    use_async=True,
)
nodes = await retriever.aretrieve(query)
```

`rank_bm25` 직접 호출, 수동 RRF 계산, httpx 임베딩 호출이 전부 사라졌다. 한글 2-gram 토크나이저는 `BM25Retriever`의 `tokenizer` 파라미터로 그대로 주입했다.

### 저장

```python
# AS-IS
doc_id = f"mem_{int(time.time() * 1000)}"
col.upsert(ids=[doc_id], documents=[text], embeddings=embeddings, metadatas=[...])

# TO-BE
doc = Document(text=text, id_=doc_id, metadata={"timestamp": ..., "user_id": user_id})
index.insert(doc)  # 임베딩 + 저장 자동 처리
```

### 실제 저장 예시

Slack에서 "결혼정장 업체명: 슈트패브릭, 전화번호: 0507-1370-9606, 분류: 결혼정장업체"를 기억시킨 뒤 `/admin/memory/list/{user_id}`로 확인한 결과다.

```json
{
  "id": "aa65ab63-99fd-4436-be7b-09aa674c5e5e",
  "doc": "결혼정장 업체명: 슈트패브릭, 전화번호: 0507-1370-9606, 분류: 결혼정장업체",
  "meta": {
    "ref_doc_id": "mem_1781587920315",
    "document_id": "mem_1781587920315",
    "_node_type": "TextNode",
    "_node_content": "{\"id_\": \"aa65ab63-...\", \"text\": \"\", ...}",
    "timestamp": "1781587920",
    "user_id": "U0B8D85JQUW",
    "doc_id": "mem_1781587920315"
  }
}
```

주목할 점은 `_node_type: "TextNode"`다. 기존에 ChromaDB를 직접 호출하면 이 필드가 없었다. LlamaIndex가 저장 시 노드 재구성에 필요한 메타데이터를 자동으로 추가하기 때문에, 이후 `VectorStoreIndex.as_retriever()`가 결과를 완전히 복원할 수 있다.

`_node_content`의 `text` 필드가 비어있는 것도 정상이다. LlamaIndex는 ChromaDB의 `documents` 필드에 텍스트를 저장하고 `_node_content`에는 중복 저장하지 않는다.

---

## 2. agent_service — 대화 히스토리 압축

### 문제

기존 `chat()` 호출은 매번 독립적이었다. 이전 대화 맥락이 없어서 "아까 말한 거 기억해?" 같은 흐름이 안 됐다.

### AS-IS

```python
# 이전 대화 컨텍스트 없음
async def chat(user_id, message, channel_id=""):
    augmented_message = await _prefetch_memory(user_id, message)  # ChromaDB 검색만
    result = await _invoke_graph(llm, tools, augmented_message, timeout=120)
```

### TO-BE

```python
# _chat_histories: dict[str, list[tuple[str, str]]] = {}
# 5턴 초과 시 오래된 대화를 LLM으로 요약, 최근 3턴은 원문 유지

async def _get_history_context(user_id: str) -> str:
    history = _chat_histories.get(user_id, [])
    if len(history) <= MAX_HISTORY_TURNS:
        return "[이전 대화]\n" + "\n\n".join(f"사용자: {u}\n비서: {a}" for u, a in history)

    # 오래된 대화 요약
    old = history[:-KEEP_RECENT_TURNS]
    old_text = "\n".join(f"사용자: {u}\n비서: {a}" for u, a in old)
    summary = await llm.ainvoke(f"다음 대화를 핵심 사실 위주로 2-3문장으로 요약해줘.\n\n{old_text}")

    # 히스토리 교체: 요약 1개 + 최근 3턴
    _chat_histories[user_id] = [("(이전 대화 요약)", summary.content)] + history[-KEEP_RECENT_TURNS:]
    return f"[이전 대화 요약]\n{summary.content}\n\n[최근 대화]\n..."

# chat()에서 주입
history_context = await _get_history_context(user_id)
augmented_message = history_context + "\n" + await _prefetch_memory(user_id, message)
```

```
대화 흐름 예시:

턴1: "또리 생일이 10월 27일이야"
턴2: "김철수 전화번호는 010-1234-5678"
턴3: "내일 회의 있어"
턴4: "저녁에 헬스장 갈 예정"
턴5: "퇴근 후 뭐 먹을까"

→ 5턴 초과 시:
   [요약] "또리 생일 10월 27일, 김철수 전화 010-1234-5678, 내일 회의 예정"
   [최근] 턴4, 턴5 원문 유지
```

LLM이 이전 대화를 참조하면서도 컨텍스트 윈도우를 낭비하지 않는다.

---

## 3. Knowledge Graph — LlamaIndex PropertyGraphIndex

### 추가 이유

ChromaDB 기억은 텍스트 유사도 기반이다. "또리 관련 정보 다 알려줘" 같은 엔티티 중심 쿼리에서 관련 없는 텍스트가 섞이거나 누락될 수 있다.

Knowledge Graph는 `(또리) -[생일]→ (10월 27일)` 같은 관계를 명시적으로 저장한다.

### 구현

```python
from llama_index.core.graph_stores import SimplePropertyGraphStore
from llama_index.core.graph_stores.types import EntityNode, Relation

# 저장 (기억 저장 시 자동 트리거)
async def save_to_graph(user_id: str, text: str) -> None:
    triplets = await _extract_triplets(text)  # Gemini로 (entity, attribute, value) 추출
    for t in triplets:
        entity_node = EntityNode(name=t["entity"], label="Entity",
                                 properties={t["attribute"]: t["value"]})
        value_node = EntityNode(name=t["value"], label="Value")
        relation = Relation(source_id=entity_node.id, target_id=value_node.id,
                            label=t["attribute"])
        store.upsert_nodes([entity_node, value_node])
        store.upsert_relations([relation])
    store.persist("./graph_store/user_graph.json")

# 조회
def query_graph(user_id: str, entity_name: str) -> list[str]:
    nodes = store.get(ids=[f"{entity_name}_Entity"])
    results = [f"{node.name}: {attr} = {val}" for attr, val in node.properties.items()]
    return results
```

```
"또리 생일 기억해줘. 10월 27일이야."
    ↓ save_to_graph 자동 호출
    ↓ _extract_triplets → [{"entity": "또리", "attribute": "생일", "value": "10월 27일"}]
    ↓ EntityNode("또리") -[생일]→ EntityNode("10월 27일")

"또리 관련 정보 알려줘"
    ↓ query_knowledge_graph(entity="또리")
    ↓ ["또리: 생일 = 10월 27일"]
```

그래프는 `./graph_store/` 디렉토리에 JSON으로 영속화된다.

---

## 추가된 패키지

```toml
# pyproject.toml
"llama-index-core>=0.12.0",
"llama-index-embeddings-ollama>=0.5.0",
"llama-index-retrievers-bm25>=0.5.0",
"llama-index-vector-stores-chroma>=0.4.0",
```

`rank-bm25`는 `llama-index-retrievers-bm25`로 대체 (직접 참조는 제거).

---

## 변경 요약

| 영역 | AS-IS | TO-BE |
|---|---|---|
| 임베딩 | httpx → Ollama API 직접 호출 | OllamaEmbedding (LlamaIndex) |
| 벡터 저장/조회 | ChromaDB 직접 호출 | VectorStoreIndex + ChromaVectorStore |
| 하이브리드 검색 | BM25Okapi + 수동 RRF | QueryFusionRetriever (mode="reciprocal_rerank") |
| 대화 히스토리 | 없음 (매 호출 독립) | `_chat_histories` + LLM 압축 요약 |
| 엔티티 기억 | 텍스트 평문 저장만 | Knowledge Graph (PropertyGraphIndex) |

---

## 시리즈 링크

- [(1) 도입 배경과 초기 세팅](/posts/ai-log-agent-new-1)
- [(2) FastAPI와 파이프라인 설계](/posts/ai-log-agent-new-2)
- [(3) Embedding 모델과 ChromaDB RAG 구축](/posts/ai-log-agent-new-3)
- [(4) RAG 검색에서 로컬 파일 직접 탐색으로 전환](/posts/ai-log-agent-new-4)
