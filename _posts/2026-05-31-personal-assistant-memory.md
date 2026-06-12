---
layout: post
title: "개인 비서에 장기 기억 달기 — ChromaDB + Ollama 임베딩"
date: 2026-05-31 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [ChromaDB, VectorDB, Embedding, Ollama, Memory, RAG, Python]
---

대화형 AI의 근본적인 한계가 있다. 오늘 "내 생일은 10월 27일이야"라고 말해도, 내일 다시 "내 생일 언제야?"라고 물으면 모른다. LLM의 컨텍스트는 대화 세션이 끝나면 사라지기 때문이다.

개인 비서로 쓰려면 이전에 말한 것들을 기억해야 한다. "와이프 이름 뭐라고 했지?", "내가 선호하는 음식이 뭐야?" 같은 질문에 답할 수 있어야 한다.

이걸 구현하는 방법으로 ChromaDB를 선택했다. 텍스트를 벡터로 변환해서 저장하고, 나중에 유사한 텍스트를 빠르게 검색할 수 있는 벡터 데이터베이스다.

---

## 구조

```
[저장 흐름]
사용자 발화 → Ollama embed API → 768차원 벡터 → ChromaDB upsert

[검색 흐름]
검색 쿼리 → Ollama embed API → 쿼리 벡터 → ChromaDB cosine similarity → 유사 문서 반환
```

임베딩 모델은 Ollama의 `nomic-embed-text`를 사용했다. 이미 Ollama가 서버에 올라가 있어서 별도 외부 API 없이 사용할 수 있다.

---

## 임베딩 API 호출

LangChain의 임베딩 클래스를 쓰지 않고 Ollama HTTP API를 직접 호출했다. 의존성을 줄이기 위해서다.

```python
async def _embed(texts: list[str]) -> list[list[float]]:
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(
            f"{settings.ollama_host}/api/embed",
            json={"model": settings.ollama_embed_model, "input": texts},
        )
        resp.raise_for_status()
        return resp.json()["embeddings"]
```

`/api/embed`는 Ollama가 제공하는 임베딩 전용 엔드포인트다. `input`에 문자열 리스트를 주면 각각의 임베딩 벡터를 반환한다.

---

## ChromaDB 클라이언트

ChromaDB는 in-process 모드와 HTTP 서버 모드 두 가지를 지원한다. 프로덕션에서는 HTTP 서버 모드를 쓴다.

```python
_chroma_client = None

def _client():
    global _chroma_client
    if _chroma_client is None:
        _chroma_client = chromadb.HttpClient(
            host=settings.chroma_host,
            port=settings.chroma_port,
        )
    return _chroma_client
```

싱글턴으로 만드는 이유가 있다. 처음엔 매 요청마다 `chromadb.HttpClient()`를 새로 만들었다. 그랬더니 로그에 `ResourceWarning: unclosed socket` 경고가 쌓였다. 클라이언트 내부에서 TCP 커넥션을 열고, 객체가 GC될 때 명시적으로 닫지 않아서 생기는 문제였다. 싱글턴으로 바꾸자 경고가 사라졌다.

참고로 `chromadb.HttpClient`는 클래스가 아니라 팩토리 함수다. 처음에 `_chroma_client: chromadb.HttpClient | None = None`으로 타입 어노테이션을 달았다가 런타임 에러가 났다. `X | None` 문법은 클래스에만 사용할 수 있다.

---

## 저장

```python
async def store_memory(user_id: str, text: str) -> None:
    try:
        embeddings = await _embed([text[:2000]])
        col = _collection(user_id)
        doc_id = f"mem_{int(time.time() * 1000)}"
        col.upsert(
            ids=[doc_id],
            documents=[text[:2000]],
            embeddings=embeddings,
            metadatas=[{"timestamp": str(int(time.time()))}],
        )
    except Exception as e:
        logger.warning("[memory] store failed: %s", e)
```

컬렉션은 사용자마다 분리한다(`memory_{user_id}`). 여러 사용자가 같은 앱을 쓰더라도 메모리가 섞이지 않는다.

`text[:2000]`으로 자르는 건 임베딩 모델의 컨텍스트 한계 때문이기도 하고, ChromaDB 문서 크기 제한 때문이기도 하다.

---

## 검색

```python
async def search_memory(user_id: str, query: str, n: int = 3) -> list[str]:
    try:
        col = _collection(user_id)
        count = col.count()
        if count == 0:
            return []
        embeddings = await _embed([query[:2000]])
        results = col.query(
            query_embeddings=embeddings,
            n_results=min(n, count),
        )
        return results["documents"][0]
    except Exception as e:
        logger.warning("[memory] search failed: %s", e)
        return []
```

`n_results=min(n, count)`가 필요하다. 컬렉션에 저장된 문서 수보다 많은 수를 요청하면 ChromaDB가 에러를 낸다. 예를 들어 저장된 게 1개인데 `n_results=3`을 요청하면 실패한다.

cosine similarity 기준으로 가장 유사한 문서 3개를 가져온다. 컬렉션 생성 시 `metadata={"hnsw:space": "cosine"}`으로 지정했다.

---

## 에이전트에 메모리 주입하는 방법

처음에는 `search_memory`를 에이전트 도구로만 등록했다. "내 생일 언제야?" 라고 물으면 에이전트가 알아서 `search_memory`를 호출해서 찾아줄 거라고 생각했다.

실제로는 호출하지 않는 경우가 많았다. LLM이 "기억에서 찾아야 할 것 같다"는 판단을 매번 정확히 내리지 않는다. "내 생일 언제야?" → "제가 기억하는 생일 정보가 없습니다"라고 그냥 답해버렸다.

해결책으로 **메모리 자동 주입**을 적용했다.

```python
async def _prefetch_memory(user_id: str, message: str) -> str:
    try:
        memories = await asyncio.wait_for(
            memory_service.search_memory(user_id, message),
            timeout=10,
        )
        if not memories:
            return message
        context = "\n".join(f"- {m}" for m in memories)
        return f"[기억된 정보 (자동 조회)]\n{context}\n\n[사용자 메시지]\n{message}"
    except Exception as e:
        return message
```

`chat()` 함수 진입 시 항상 ChromaDB를 검색해서, 관련 기억이 있으면 사용자 메시지 앞에 붙여준다. 에이전트가 도구를 호출할지 말지 결정하는 게 아니라, 이미 컨텍스트에 답이 들어있는 셈이다.

"내 생일 언제야?" → 메시지 앞에 `[기억된 정보] - 사용자 생일: 10월 27일`이 붙어서 전달됨 → LLM이 바로 답변한다.

타임아웃을 10초로 설정한 이유는, 검색 실패 시 원래 메시지를 그대로 써서 에이전트가 정상 동작해야 하기 때문이다. 메모리 검색이 실패해도 응답은 나와야 한다.

---

## Docker 네트워크 문제

ChromaDB는 별도 Docker 컨테이너로 띄웠다. 처음에 `CHROMA_HOST=host.docker.internal`로 설정했는데 연결이 안 됐다. `host.docker.internal`은 컨테이너에서 호스트 머신에 접근하는 경로인데, bridge 네트워크에서는 동작하지 않을 수 있다.

결국 두 컨테이너를 같은 Docker 네트워크에 붙이는 방법으로 해결했다.

```yaml
# docker-compose.yml
networks:
  log-agent-net:
    name: log-agent_default
    external: true
```

ChromaDB가 포함된 `log-agent-backend`의 네트워크(`log-agent_default`)를 external로 참조해서 같은 네트워크에 합류하는 방식이다. 이후엔 `CHROMA_HOST=chromadb`(컨테이너 이름)으로 바로 접근된다.

`docker inspect chromadb`로 컨테이너가 어떤 네트워크에 붙어있는지 확인해서 네트워크 이름을 찾았다.
