---
layout: post
title: "Contextual RAG — 청크에 LLM 컨텍스트를 붙여 검색 품질 높이기"
date: 2026-06-11 10:00:00 +0900
categories: [AI, RAG]
tags: [RAG, ChromaDB, Python, LLM, 임베딩, 검색]
---

RAG를 구축하다 보면 청크 분할 자체가 검색 품질을 떨어뜨리는 경우를 만난다. 파일을 1500자씩 자르면 클래스 이름이나 메서드 목적 같은 맥락 정보가 청크에서 잘려나가기 때문이다.

Anthropic이 제안한 **Contextual RAG**는 이 문제를 LLM으로 해결한다.

---

## 문제: 청크는 맥락을 잃는다

`UserService.java`를 청크로 자르면 이런 상황이 생긴다.

```
청크 3:
    if (user == null) {
        throw new UserNotFoundException(id);
    }
    return user;
}
```

이 청크만 보면 어떤 클래스의 어떤 메서드인지 전혀 알 수 없다. 임베딩 모델도 마찬가지다. "UserService에서 NullPointerException 발생"이라는 쿼리로 검색할 때 이 청크가 상위에 오를 이유가 없다.

---

## 해결: 청크마다 LLM 요약을 앞에 붙인다

인덱싱 전에 LLM에게 각 청크가 무엇인지 한 문장으로 설명하게 하고, 원본 앞에 붙인다.

```python
async def _contextualize_chunk(path: str, content: str) -> str:
    prompt = (
        f"파일: {path}\n코드:\n{content[:800]}\n\n"
        "이 코드의 역할을 한 문장으로 설명해줘. 클래스명/메서드명/주요 기능을 포함해. "
        "설명만 출력하고 다른 말은 하지 마."
    )
    # Ollama 호출 → 요약 생성
    summary = await _call_llm(prompt)
    return f"{summary}\n\n{content}"
```

결과는 이렇게 된다.

```
UserService 클래스의 findById 메서드로, ID로 사용자를 조회하고 없으면 UserNotFoundException을 던진다.

    if (user == null) {
        throw new UserNotFoundException(id);
    }
    return user;
}
```

이제 "UserService NullPointerException"으로 검색하면 이 청크가 상위에 온다.

---

## 구현: 병렬 처리로 속도 확보

청크가 수백 개면 LLM 호출이 그만큼 필요하다. `asyncio.Semaphore`로 동시 호출을 제한하면서 `asyncio.gather`로 병렬 처리한다.

```python
_CONTEXT_SEMAPHORE = asyncio.Semaphore(5)  # Ollama 동시 호출 5개로 제한

async def _contextualize_all(chunks: list[tuple[str, str]]) -> list[tuple[str, str]]:
    tasks = [_contextualize_chunk(path, content) for path, content in chunks]
    contextualized = await asyncio.gather(*tasks)
    return [(path, ctx) for (path, _), ctx in zip(chunks, contextualized)]
```

LLM 호출이 실패해도 원본 청크로 폴백하므로 인덱싱이 중단되지 않는다.

---

## 효과

| | 기존 RAG | Contextual RAG |
|---|---|---|
| 청크 내용 | 코드만 | LLM 요약 + 코드 |
| 임베딩 대상 | 컨텍스트 없는 코드 | 목적이 명시된 코드 |
| BM25 대상 | 동일 | 요약에 클래스명·메서드명 포함 |
| 인덱싱 속도 | 빠름 | LLM 호출만큼 추가 |

벡터 검색과 BM25 **둘 다** 개선된다는 점이 핵심이다. 요약에 클래스명·메서드명이 포함되기 때문에 키워드 검색도 더 잘 걸린다.

---

## 트레이드오프

인덱싱 시간이 늘어난다. 청크 100개면 LLM 호출이 100번 추가된다. 이 프로젝트에서는 커밋이 바뀔 때만 재인덱싱하므로 허용 가능한 비용이었다. 실시간 인덱싱이 필요한 환경이라면 경량 모델을 따로 두거나, 파일 첫 청크에만 적용하는 방식으로 절충할 수 있다.
