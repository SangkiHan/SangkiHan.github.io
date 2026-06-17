---
title: "AI 오류 자동 분석 에이전트 구축기 (4) — RAG 검색에서 로컬 파일 직접 탐색으로 전환"
author: SangkiHan
date: 2026-06-16 14:00:00 +0900
categories: [AI, 프로젝트]
tags: [LLM, Agent, RAG, ChromaDB, LangGraph, Ollama, 리팩토링]
---

## 배경

에이전트 구축기 (3)에서 ChromaDB 기반 RAG로 소스코드를 검색하는 방식을 구현했다. 에러가 발생하면 관련 파일을 청킹·임베딩해 벡터로 저장하고, 분석 시 유사도 검색으로 관련 청크를 가져오는 방식이었다.

실제 운영하면서 이 방식이 생각만큼 잘 동작하지 않는다는 걸 깨달았다. 이 포스트는 왜 RAG를 버렸는지, 어떻게 바꿨는지를 정리한다.

---

## 무엇이 문제였나

### 1. 청크 경계에서 코드가 잘린다

RAG는 파일을 일정 크기로 잘라 저장한다. 문제는 에러 원인 코드가 청크 경계에 걸렸을 때다.

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

LLM이 첫 번째 청크만 받으면 `selectList()` 호출부를 못 보는 상황이 발생했다. 코드 분석은 파일 전체 맥락이 필요한데, RAG는 구조적으로 이를 보장할 수 없었다.

### 2. 에러마다 인덱싱 오버헤드가 붙는다

기존 구조는 에러가 발생할 때마다 해당 커밋 기준으로 전체 파일을 인덱싱했다.

```python
# 기존 코드 (webhook.py)
if not rag_service.is_indexed(server.id, commit):
    chunks = await asyncio.to_thread(
        git_service.list_all_files_at_commit, server.id, commit
    )
    await rag_service.index_repo(server.id, commit, chunks)
```

파일이 많은 프로젝트에서는 인덱싱 자체가 수십 초씩 걸렸다. 에러 분석이 느려지는 원인이 됐다.

### 3. 정확한 식별자 검색에 벡터가 약하다

스택 트레이스에는 이미 `AsResultMapper`, `TestErrorController` 같은 **정확한 클래스명**이 있다. 벡터 유사도 검색은 의미적으로 비슷한 문서를 찾는 데 강하지만, 정확한 식별자 매칭은 오히려 `grep`이 훨씬 정확하다.

```
# 스택 트레이스 예시
at com.sems.global.TestErrorController.triggerNpe(TestErrorController.java:21)
```

이미 파일명이 나와있는데 벡터 검색으로 "비슷한 파일"을 찾을 필요가 없었다.

---

## 어떻게 바꿨나

Claude Code가 코드베이스를 탐색하는 방식에서 힌트를 얻었다. Claude Code는 RAG 없이 `grep`, `read_file`, `list_directory` 도구로 직접 파일을 탐색한다. 같은 방식을 에이전트에 적용했다.

### 제거한 것

- ChromaDB 소스코드 인덱싱 파이프라인 전체
- `search_code` 단일 도구 (벡터 검색)
- 에러 수신 시 RAG 인덱싱 블록

### 추가한 것

```python
def _make_file_tools(repo_path: Path):
    @langchain_tool
    async def grep_files(pattern: str, path: str = "", file_extension: str = "") -> str:
        """레포지토리에서 클래스명·메서드명·패턴을 검색합니다."""
        regex = re.compile(pattern, re.IGNORECASE)
        results = []
        for fp in sorted(base.rglob("*")):
            if not fp.is_file() or fp.suffix not in _SOURCE_EXTENSIONS:
                continue
            lines = fp.read_text(encoding="utf-8", errors="replace").split("\n")
            matched = [f"  {i}: {l}" for i, l in enumerate(lines, 1) if regex.search(l)]
            if matched:
                results.append(f"=== {fp.relative_to(repo_path)} ===\n" + "\n".join(matched[:10]))
        return "\n\n".join(results) if results else f"패턴 '{pattern}'을 찾을 수 없습니다."

    @langchain_tool
    async def read_file(path: str) -> str:
        """파일 전체 내용을 읽습니다."""
        content = target.read_text(encoding="utf-8", errors="replace")
        return content[:4000]  # 길면 앞 4000자

    @langchain_tool
    async def list_directory(path: str = "") -> str:
        """디렉토리 구조를 탐색합니다."""
        ...

    return [grep_files, read_file, list_directory]
```

### 에이전트 동작 비교

**AS-IS — 벡터 검색 후 청크 전달**

```
[LLM]  → search_code("AsResultMapper selectList")
[Tool] ← 청크 A (인터페이스 일부)
         청크 B (XML mapper 일부 — 잘린 상태)
[LLM]  → 불완전한 정보로 추측 분석
```

**TO-BE — 직접 탐색 후 파일 전체 전달**

```
[LLM]  → grep_files("AsResultMapper")
[Tool] ← === src/main/java/.../AsResultMapper.java ===
           23: public interface AsResultMapper {

[LLM]  → read_file("src/main/java/.../AsResultMapper.java")
[Tool] ← (파일 전체 내용)

[LLM]  → grep_files("selectList", file_extension="xml")
[Tool] ← === src/main/resources/mapper/.../asResult.xml ===
           142: <select id="selectList" resultType="AsResult">

[LLM]  → read_file("src/main/resources/mapper/.../asResult.xml")
[Tool] ← (XML 전체 내용)

[LLM]  → 실제 코드를 보고 정확한 수정안 작성
```

---

## 결과

RAG 방식과 비교해 실제 개선된 점들이다.

| 항목 | AS-IS (RAG) | TO-BE (로컬 파일) |
|------|-------------|------------------|
| 인덱싱 | 에러마다 전체 파일 청킹·임베딩 | 없음 (git fetch만) |
| 파일 내용 | 청크 단위 (잘린 조각) | 파일 전체 |
| 식별자 매칭 | 벡터 유사도 기반 | 정규식 정확 매칭 |
| 파일 경로 | 메타데이터로 추론 | 실제 경로 직접 반환 |
| 추가 인프라 | ChromaDB 인덱싱 파이프라인 | 없음 |

---

## 남긴 것: Error Memory

소스코드 인덱싱을 위한 ChromaDB는 제거했지만, **Error Memory**(과거 에러 분석 사례 저장)는 유지했다. 이건 RAG의 약점인 정확한 식별자 검색과 무관하고, "과거에 비슷한 에러를 어떻게 해결했나"라는 의미 검색에는 벡터가 적합하기 때문이다.

```
[Error Memory 흐름]
에러 분석 완료
  → 분석 결과를 ChromaDB에 저장 (error_memory 컬렉션)

다음 에러 발생
  → "비슷한 에러 사례 있나?" 벡터 검색
  → 과거 분석 결과를 프롬프트에 주입
```

RAG가 강한 용도(의미 기반 사례 검색)에는 계속 쓰고, 약한 용도(정확한 코드 파일 탐색)는 직접 파일 접근으로 대체한 셈이다.

---

## 핵심 교훈

RAG는 범용적으로 좋은 기술이지만, 모든 상황에 최선은 아니다. 에러 로그 분석처럼 **스택 트레이스에 이미 정확한 파일명과 클래스명이 있는 경우**, 벡터 검색보다 직접 파일 탐색이 훨씬 단순하고 정확했다.

인프라를 줄이고 LLM에게 직접 파일을 탐색할 수 있는 도구를 주는 것이 결과적으로 더 나은 분석 품질을 만들어냈다.

---

## 시리즈 링크

- [(1) 도입 배경과 초기 세팅](/posts/ai-log-agent-new-1)
- [(2) FastAPI와 파이프라인 설계](/posts/ai-log-agent-new-2)
- [(3) Embedding 모델과 ChromaDB RAG 구축](/posts/ai-log-agent-new-3)
- **(4) RAG 검색에서 로컬 파일 직접 탐색으로 전환 ← 현재 포스트**
