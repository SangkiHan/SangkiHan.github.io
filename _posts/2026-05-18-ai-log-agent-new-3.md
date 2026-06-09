---
layout: post
title: "AI 오류 자동 분석 에이전트 구축기 (3) — Embedding 모델과 ChromaDB RAG 구축"
date: 2026-05-18 12:00:00 +0900
categories: [Project, AI]
tags: [RAG, ChromaDB, Embedding, Python, LLM]
---

에러 로그만 보고 정확한 수정 코드를 제안하려면 LLM이 소스 파일을 알고 있어야 한다. 소스 파일 전체를 매번 프롬프트에 넣을 수는 없다. RAG(Retrieval-Augmented Generation)가 필요한 이유다.

## 왜 RAG를 도입했나

puppynote-server는 수백 개의 Java 파일로 구성되어 있다. 이걸 전부 LLM 프롬프트에 넣으면:

1. **토큰 한계** — gemma4:12b는 컨텍스트 윈도우가 제한적이다. 소스 전체를 넣을 수 없다.
2. **속도 저하** — 토큰이 많을수록 처리 시간이 길어진다.
3. **노이즈** — 관련 없는 파일이 가득하면 정작 중요한 파일의 내용이 묻힌다.

**해결**: 에러와 관련된 파일 3~5개만 골라서 LLM에게 보여준다. 이게 RAG다.

---

## 구조

```
소스 파일
    │ 청크 분할 (git_service.py)
    ▼
nomic-embed-text (임베딩 모델)
    │ 텍스트 → 벡터 변환
    ▼
ChromaDB (벡터 DB)
    │ 코사인 유사도로 저장
    ▼
에러 쿼리 → 유사 파일 검색
    │
    ▼
LLM 프롬프트에 관련 파일만 포함
```

---

## nomic-embed-text

텍스트를 벡터(숫자 배열)로 변환하는 임베딩 전용 모델이다. 에러 분석을 담당하는 gemma4:12b와는 완전히 별개의 모델이다.

```python
# rag_service.py
async def _embed(texts: list[str]) -> list[list[float]]:
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(
            f"{settings.ollama_host}/api/embed",
            json={
                "model": settings.ollama_embed_model,  # "nomic-embed-text"
                "input": texts,
            },
        )
        resp.raise_for_status()
        return resp.json()["embeddings"]
```

Ollama가 nomic-embed-text를 내부적으로 실행하므로 별도 서버 없이 `/api/embed` 엔드포인트 하나로 임베딩을 처리한다.

---

## ChromaDB 컬렉션 구조

서버별로 독립적인 컬렉션을 만든다.

```python
def _collection(server_id: int):
    client = chromadb.HttpClient(
        host=settings.chroma_host, port=settings.chroma_port
    )
    return client.get_or_create_collection(
        name=f"server_{server_id}",
        metadata={"hnsw:space": "cosine"},  # 코사인 유사도
    )
```

`hnsw:space: cosine`으로 코사인 유사도를 거리 함수로 사용한다. 텍스트 검색에서 코사인 유사도가 유클리드 거리보다 더 적합하다 (벡터 길이가 아닌 방향으로 유사도 측정).

---

## 초기 구현의 문제: 줄 단위 청크

처음에는 소스 파일을 단순히 일정 길이로 잘랐다.

```python
# 초기 구현 (문제 있음)
def _simple_chunk(content, chunk_size=1500):
    return [content[i:i+chunk_size] for i in range(0, len(content), chunk_size)]
```

**문제**: Java/Kotlin 파일을 1500자로 자르면 메서드가 중간에 잘린다.

```java
// 청크 1 (끝 부분)
    private String buildMessage(User user) {
        return "Hello, " + user.getName()

// 청크 2 (시작 부분)
        + " welcome!";
    }
```

잘린 청크는 의미가 없다. LLM이 받아봐야 불완전한 코드라 정확한 판단을 못 한다.

---

## 개선: 의미 단위 청크

Java/Kotlin 파일은 메서드/클래스 경계로 자른다.

```python
# git_service.py
_JAVA_BOUNDARY = re.compile(
    r"^[ \t]{0,4}"
    r"(?:(?:public|private|protected|static|final|abstract|synchronized|default|override)\s+)*"
    r"(?:class\s|interface\s|enum\s|record\s|fun\s+\w|\w[\w<>\[\]]*\s+\w+\s*\()"
)

def _split_java_semantic(content: str) -> list[str]:
    """메서드/클래스 선언 줄을 경계로 분할."""
    lines = content.split("\n")
    boundaries = [0]
    for i, line in enumerate(lines[1:], 1):
        indent = len(line) - len(line.lstrip())
        if indent <= 4 and line.strip() and _JAVA_BOUNDARY.match(line):
            boundaries.append(i)

    blocks = []
    for idx, start in enumerate(boundaries):
        end = boundaries[idx + 1] if idx + 1 < len(boundaries) else len(lines)
        block = "\n".join(lines[start:end]).strip()
        if block:
            blocks.append(block)
    return blocks or [content]
```

정규식이 들여쓰기 4 이하의 `public void`, `private String`, `class Foo`, Kotlin의 `fun` 등을 잡는다. 이렇게 잘린 블록은 완전한 메서드 또는 클래스다.

---

## 슬라이딩 윈도우

의미 단위로 나눠도 하나의 블록이 1500자를 넘을 수 있다. 이 경우 슬라이딩 윈도우로 추가 분할한다.

```python
def _make_chunks(path: str, content: str, chunk_size: int = 1500, overlap: int = 200):
    ext = Path(path).suffix.lower()
    # .java/.kt는 의미 단위 분할, 나머지는 전체를 하나의 블록으로
    blocks = _split_java_semantic(content) if ext in {".java", ".kt"} else [content]

    result = []
    for block in blocks:
        if len(block) <= chunk_size:
            result.append(block)
        else:
            # 큰 블록은 overlap=200으로 슬라이딩
            start = 0
            while start < len(block):
                result.append(block[start:start + chunk_size])
                if start + chunk_size >= len(block):
                    break
                start += chunk_size - overlap  # 200자 겹침
    return result
```

`overlap=200`은 청크 경계에서 문맥이 끊기지 않도록 앞 청크의 마지막 200자를 다음 청크 시작에 포함한다.

---

## 인덱싱

commit hash가 바뀔 때만 전체 재인덱싱한다. 매번 하면 느리다.

```python
async def index_repo(server_id: int, commit_hash: str, chunks: list[tuple[str, str]]) -> None:
    col = _collection(server_id)

    all_embeddings = []
    for i in range(0, len(chunks), _BATCH_SIZE):  # 50개씩 배치
        batch = [content for _, content in chunks[i:i + _BATCH_SIZE]]
        embeddings = await _embed(batch)
        all_embeddings.extend(embeddings)

    col.upsert(
        ids=[f"{path}__chunk{i}" for i, (path, _) in enumerate(chunks)],
        documents=[content for _, content in chunks],
        embeddings=all_embeddings,
        metadatas=[{"path": path, "commit": commit_hash} for path, _ in chunks],
    )

    # commit hash를 파일에 저장 — 다음 호출 시 재인덱싱 여부 판단
    state = _load_state()
    state[str(server_id)] = commit_hash
    _save_state(state)
```

---

## 검색

```python
async def search_relevant_files(server_id: int, query: str, n_results: int = 5) -> list[str]:
    query_embeddings = await _embed([query[:2000]])
    results = col.query(
        query_embeddings=query_embeddings,
        n_results=candidates,
    )
    return paths  # 파일 경로 목록
```

에러 쿼리를 임베딩해서 ChromaDB에서 코사인 유사도로 가장 가까운 청크를 찾는다.

---

## 통신 흐름 상세

```
[Before — LLM 분석]

에러 로그
    │
    ▼
LLM 프롬프트
    │ 소스 파일 없음 또는 전체 파일 포함
    │ → 토큰 초과 or 관련 없는 코드로 노이즈
    ▼
부정확한 수정 제안

[After — RAG 도입]

puppynote-server     FastAPI(노트북)       ChromaDB     Ollama(M4)
     │ 에러 발생         │                    │             │
     │──POST /error──▶  │                    │             │
     │                  │──git clone          │             │
     │                  │──청크 분할           │             │
     │                  │   (의미 단위)        │             │
     │                  │──embed────────────▶ nomic-embed  │
     │                  │──upsert───────────▶ │             │
     │                  │   (commit 변경시)   │             │
     │                  │                    │             │
     │                  │──analyze──────────────────────▶  │
     │                  │   에러 쿼리로        │             │ search_files
     │                  │──embed query──────▶ nomic-embed  │ (tool call)
     │                  │──cosine search────▶ ChromaDB     │
     │                  │◀──관련 파일 3~5개── │             │
     │                  │────────────────────────────────▶ │
     │                  │                관련 소스 파일     │
     │                  │◀──────────────────────────────── │
     │                  │              JSON 분석 결과       │
     │                  │──Slack 전송                       │

M4 Mac Mini에서 nomic-embed-text와 gemma4:12b가 동시에 돌아간다.
ChromaDB는 개인 노트북 서버(FastAPI와 동일 호스트)에서 Docker로 실행.
```
