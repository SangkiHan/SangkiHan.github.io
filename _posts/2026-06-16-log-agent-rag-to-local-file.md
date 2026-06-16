---
title: Log Agent - RAG 기반 코드 검색에서 로컬 파일 직접 조회로 전환
author: SangkiHan
date: 2026-06-16 11:00:00 +0900
categories: [AI, LLM]
tags: [LangGraph, RAG, ChromaDB, Ollama, FastAPI, Agent]
---

## 개요

서버 에러 로그를 자동으로 분석하는 Log Agent에서 프로젝트 소스코드 검색 방식을 변경했습니다.

기존에는 **RAG(Retrieval-Augmented Generation)** 방식으로 ChromaDB에 코드를 청킹·임베딩해 벡터 검색으로 관련 파일을 찾았습니다. 하지만 실제 운영 중에 여러 한계를 발견했고, **로컬 파일을 직접 읽는 Tool 방식**으로 전환했습니다.

---

## 기존 방식 — RAG 기반 코드 검색

### 전체 흐름

```
에러 발생
  → Git push
  → Webhook 수신
  → 전체 파일 청킹 + ChromaDB 임베딩 인덱싱
  → LLM 분석 시 벡터 검색으로 관련 청크 조회
  → 조회된 청크를 프롬프트에 삽입해 분석
```

### AS-IS 코드 (webhook.py)

```python
@router.post("/error")
async def receive_error(payload, background_tasks, db):
    server = ...

    # RAG 인덱싱 — 매 에러마다 커밋 단위로 전체 파일 인덱싱
    if not rag_service.is_indexed(server.id, commit):
        chunks = await asyncio.to_thread(
            git_service.list_all_files_at_commit, server.id, commit
        )
        await rag_service.index_repo(server.id, commit, chunks)

    background_tasks.add_task(_analysis_pipeline, server, payload)
```

### AS-IS 코드 (ollama_service.py — 검색 도구)

```python
def _make_search_tool(server_id: int, commit: str):
    @langchain_tool
    async def search_code(query: str) -> str:
        """관련 소스코드를 벡터 검색으로 가져옵니다."""
        results = await rag_service.search(server_id, commit, query, n=5)
        if not results:
            return "관련 코드를 찾을 수 없습니다."
        return "\n\n---\n\n".join(
            f"[{r['file_path']}]\n{r['content']}" for r in results
        )
    return [search_code]
```

### AS-IS 프롬프트

```
You are a senior DevSecOps engineer.

Available tool:
- search_code(query): 코드베이스에서 관련 코드 청크를 벡터 검색합니다.

에러 로그를 분석하고 search_code로 관련 코드를 찾아 수정 제안을 JSON으로 반환하세요.
```

---

## 문제점

### 1. 청크가 잘려서 컨텍스트 유실

RAG는 파일을 일정 크기로 잘라(청킹) 저장합니다. 문제는 관련 코드가 청크 경계에서 잘려나갔을 때, LLM이 불완전한 정보로 분석한다는 점입니다.

```
[실제 파일]                    [ChromaDB에 저장된 청크]
┌──────────────────────┐      ┌──────────────────────┐
│ public Result query( │      │ public Result query( │
│   String param) {    │  →   │   String param) {    │
│   // 핵심 로직       │      │   // 핵심 로직       │
│   ...                │      └──────────────────────┘
│   return mapper      │      ┌──────────────────────┐  ← 다른 청크
│     .selectList();   │      │   return mapper      │
│ }                    │      │     .selectList();   │
└──────────────────────┘      │ }                    │
                              └──────────────────────┘
```

LLM이 첫 번째 청크만 받으면 `selectList()` 호출부를 못 보는 상황이 발생합니다.

### 2. 매 에러마다 인덱싱 오버헤드

에러가 발생할 때마다 해당 커밋 기준으로 전체 파일을 인덱싱합니다. 파일 수가 많은 프로젝트에서는 인덱싱 자체가 수십 초 이상 걸립니다.

### 3. 벡터 검색의 의미론적 한계

에러 스택 트레이스에 등장하는 클래스명·메서드명은 **정확한 키워드 매칭**이 필요합니다. 벡터 유사도 검색은 의미적으로 유사한 청크를 찾는 데는 강하지만, `AsResultMapper.selectList` 같은 정확한 식별자 검색에는 오히려 취약합니다.

### 4. 파일 경로 정보 부정확

청크에는 파일 경로가 메타데이터로 붙지만, LLM이 before/after 코드를 제안할 때 어떤 파일의 몇 번째 청크에서 가져온 건지 추적이 어려워 경로 오류가 빈번했습니다.

---

## 변경 방식 — 로컬 파일 직접 조회

### 전체 흐름

```
에러 발생
  → Webhook 수신
  → Git fetch (로컬에 최신 코드 동기화)
  → LLM이 직접 grep_files / read_file / list_directory 도구로 파일 탐색
  → 실제 파일 내용을 읽고 분석
```

RAG 인덱싱 단계가 완전히 사라집니다. LLM이 마치 개발자처럼 직접 레포지토리를 탐색합니다.

### TO-BE 코드 (webhook.py)

```python
@router.post("/error")
async def receive_error(payload, background_tasks, db):
    server = ...

    # RAG 인덱싱 없음 — git fetch만 수행
    background_tasks.add_task(_analysis_pipeline, server, payload)


async def _analysis_pipeline(server, payload):
    await asyncio.to_thread(
        git_service.fetch,
        server.id, server.git_repo_url,
        server.git_branch, server.github_token or ""
    )
    suggestion = await ollama_service.analyze_log(server.id, raw_log, ...)
```

### TO-BE 코드 (ollama_service.py — 파일 도구)

```python
_SOURCE_EXTENSIONS = {".java", ".kt", ".py", ".ts", ".tsx", ".js", ".go",
                      ".xml", ".yml", ".yaml", ".properties"}
_MAX_FILE_CHARS = 4000
_MAX_GREP_FILES = 5
_MAX_GREP_LINES = 10


def _safe_path(repo_path: Path, rel: str) -> Path | None:
    """Path traversal 방지: repo 외부 경로는 None 반환."""
    target = (repo_path / rel).resolve()
    return target if str(target).startswith(str(repo_path.resolve())) else None


def _make_file_tools(server_id: int, repo_path: Path):
    @langchain_tool
    async def grep_files(pattern: str, path: str = "", file_extension: str = "") -> str:
        """레포지토리에서 클래스명·메서드명·패턴을 검색해 파일 경로와 매칭 라인을 반환합니다."""
        base = _safe_path(repo_path, path) if path else repo_path.resolve()
        exts = ({f".{file_extension.lstrip('.')}"} if file_extension else _SOURCE_EXTENSIONS)
        regex = re.compile(pattern, re.IGNORECASE)

        results = []
        for fp in sorted(base.rglob("*")):
            if not fp.is_file() or fp.suffix.lower() not in exts or ".git" in fp.parts:
                continue
            lines = fp.read_text(encoding="utf-8", errors="replace").split("\n")
            matched = [f"  {i}: {l}" for i, l in enumerate(lines, 1) if regex.search(l)]
            if matched:
                rel = fp.relative_to(repo_path)
                results.append(f"=== {rel} ===\n" + "\n".join(matched[:_MAX_GREP_LINES]))
                if len(results) >= _MAX_GREP_FILES:
                    break

        return "\n\n".join(results) if results else f"패턴 '{pattern}'을 찾을 수 없습니다."

    @langchain_tool
    async def read_file(path: str) -> str:
        """레포지토리 내 특정 파일의 내용을 읽습니다."""
        target = _safe_path(repo_path, path)
        if target is None:
            return "접근 불가: 레포지토리 외부 경로"
        content = target.read_text(encoding="utf-8", errors="replace")
        if len(content) > _MAX_FILE_CHARS:
            return content[:_MAX_FILE_CHARS] + f"\n... (총 {len(content)}자, 처음 {_MAX_FILE_CHARS}자만 표시)"
        return content

    @langchain_tool
    async def list_directory(path: str = "") -> str:
        """레포지토리 내 디렉토리 목록을 반환합니다."""
        target = _safe_path(repo_path, path) if path else repo_path.resolve()
        entries = []
        for item in sorted(target.iterdir()):
            if item.name.startswith("."):
                continue
            rel = item.relative_to(repo_path)
            entries.append(f"{rel}/" if item.is_dir() else str(rel))
        return "\n".join(entries[:100]) or "(빈 디렉토리)"

    return [grep_files, read_file, list_directory]
```

### TO-BE 프롬프트

```
You are a senior DevSecOps engineer analyzing server error logs.

MANDATORY: You MUST read the actual source files before writing your final answer.
Never guess or invent code.

Available tools:
- grep_files(pattern, path="", file_extension=""): Search for class/method names across the repo. Start here.
- read_file(path): Read a specific file. Use after finding the path via grep_files.
- list_directory(path=""): List files/directories. Use to explore project structure when needed.

Workflow:
1. Extract class or file names from the stack trace
2. Call grep_files with the class name to find the file path
3. Call read_file to read the actual source code
4. If you need config or dependencies, grep_files again
5. Once you have seen the real code, respond ONLY in JSON
```

---

## 실제 에이전트 동작 비교

### AS-IS — 벡터 검색 1회 후 분석

```
[LLM]  → search_code("AsResultMapper selectList")
[Tool] ← 청크 A (AsResultMapper 인터페이스 일부)
         청크 B (XML mapper 일부 — 잘린 상태)
         청크 C (관련 없는 유사 코드)
[LLM]  → 불완전한 청크를 보고 추측으로 분석
```

### TO-BE — 직접 파일 탐색 후 분석

```
[LLM]  → grep_files("AsResultMapper")
[Tool] ← === src/main/java/.../AsResultMapper.java ===
           23: public interface AsResultMapper {
           24:   List<AsResult> selectList(AsResultParam param);

[LLM]  → read_file("src/main/java/.../AsResultMapper.java")
[Tool] ← (파일 전체 내용)

[LLM]  → grep_files("selectList", file_extension="xml")
[Tool] ← === src/main/resources/mapper/.../asResult.xml ===
           142: <select id="selectList" resultType="AsResult">

[LLM]  → read_file("src/main/resources/mapper/.../asResult.xml")
[Tool] ← (XML 전체 내용)

[LLM]  → 실제 코드를 보고 정확한 원인 분석 + 수정 제안
```

---

## Before / After 비교

| 항목 | AS-IS (RAG) | TO-BE (로컬 파일) |
|------|-------------|------------------|
| **인덱싱** | 에러마다 전체 파일 청킹·임베딩 | 없음 (git fetch만) |
| **검색 방식** | 벡터 유사도 검색 | grep + 직접 파일 읽기 |
| **파일 내용** | 청크 단위(잘린 조각) | 파일 전체 |
| **식별자 매칭** | 의미론적 유사도 기반 | 정규식 정확 매칭 |
| **파일 경로** | 메타데이터로 추론 | 실제 경로 직접 반환 |
| **추가 인프라** | ChromaDB (에러 메모리용은 유지) | 없음 |
| **분석 정확도** | 청크 품질에 의존 | 실제 코드 기반 |

---

## 핵심 교훈

### RAG가 강한 경우

- 대규모 문서/지식베이스에서 주제 기반 검색
- 자연어 질문에 의미적으로 유사한 문서 검색
- 전체 코드를 LLM 컨텍스트에 넣기 불가능한 규모

### 로컬 파일 직접 조회가 강한 경우

- **스택 트레이스처럼 정확한 클래스명·파일명이 이미 있는 경우**
- LLM이 탐색 흐름을 스스로 제어해야 하는 경우
- 파일 전체 맥락이 필요한 코드 분석

에러 로그 분석은 후자에 해당합니다. 스택 트레이스에 이미 `AsResultMapper`, `asResult.xml` 같은 키워드가 있고, LLM이 그 파일을 온전히 읽어야 정확한 원인을 파악할 수 있습니다. RAG의 "의미 검색"보다 grep의 "정확 검색"이 훨씬 적합한 도메인이었습니다.

---

## 마치며

RAG는 범용적으로 좋은 기술이지만, 모든 상황에 최선은 아닙니다. 에러 로그 분석처럼 정확한 식별자 기반 탐색이 필요한 도메인에서는, 오히려 LLM에게 직접 파일을 탐색할 수 있는 도구를 주는 것이 더 단순하고 정확했습니다.

복잡한 인프라(ChromaDB 인덱싱 파이프라인)를 제거하고, LLM이 개발자처럼 직접 코드를 탐색하게 하는 방향이 결과적으로 더 나은 분석 품질을 만들어냈습니다.
