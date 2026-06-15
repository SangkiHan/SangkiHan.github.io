---
layout: post
title: "개인 비서 기억 시스템 디버깅 — 고유명사 검색 실패, 데이터 오염, 확인 UX"
date: 2026-06-15 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [ChromaDB, BM25, HybridSearch, RRF, LangGraph, Slack, BlockKit, Python, Debugging]
---

기억 기능을 만들고 실제로 써보니 예상 못한 곳에서 계속 터졌다. 구현보다 디버깅에 더 많은 시간을 쓴 날이었다. 문제들을 순서대로 정리했다.

---

## 1. "또리" 를 못 찾는 문제 — 순수 벡터 검색의 한계

```
사용자: 또리 생일 기억해줘
봇: ✅ 기억했습니다!

사용자: 또리 생일이 언제야
봇: 죄송합니다. 또리의 생일 정보는 찾을 수 없었습니다.
```

분명히 저장했는데 못 찾는다. 로그를 보면 검색 자체는 실행됐다. 결과가 없는 게 아니라 **관련 없는 문서 3개가 반환**됐다.

원인은 임베딩 모델이다. "또리"는 고유명사다. 임베딩 모델은 의미 기반으로 벡터를 만드는데, 사전에 없는 고유명사는 의미를 학습하지 못해 벡터 유사도가 낮게 나온다. "또리 생일"로 검색하면 "반려견 또리의 생일: 2024년 3월 30일" 문서보다 우연히 다른 문서들이 더 높은 유사도를 가질 수 있다.

해결책은 **BM25 키워드 검색과 벡터 검색을 합치는 것**이다.

BM25는 단어가 문서에 몇 번 등장하는지를 기반으로 점수를 매긴다. 벡터가 "또리"의 의미를 못 잡아도 BM25는 "또리"라는 문자열이 정확히 있는 문서에 높은 점수를 준다.

두 점수를 RRF(Reciprocal Rank Fusion)로 합산한다. 벡터 순위 3위, BM25 순위 1위면 두 점수를 합쳐 최종 순위를 결정한다. 어느 한쪽이 놓쳐도 다른 쪽이 잡아준다.

```python
from rank_bm25 import BM25Okapi
import re

def _tokenize(text: str) -> list[str]:
    return re.findall(r"[A-Za-z가-힣0-9]+", text.lower())


async def search_memory(user_id: str, query: str, n: int = 3) -> list[str]:
    col = _collection(user_id)
    count = col.count()
    candidates = min(max(n * 2, 5), count)

    # 벡터 검색
    embeddings = await _embed([query])
    vec_results = col.query(query_embeddings=embeddings, n_results=candidates)
    vec_ids = vec_results["ids"][0]
    vec_docs = vec_results["documents"][0]

    # BM25 검색
    all_data = col.get(limit=2000, include=["documents"])
    all_ids = all_data["ids"]
    all_docs = all_data["documents"]

    bm25 = BM25Okapi([_tokenize(doc) for doc in all_docs])
    bm25_scores = bm25.get_scores(_tokenize(query))
    bm25_top_idx = sorted(range(len(bm25_scores)), key=lambda i: bm25_scores[i], reverse=True)[:candidates]

    # RRF 합산 (K=60)
    K = 60
    rrf: dict[str, float] = {}
    id_to_doc: dict[str, str] = {}

    for rank, (doc_id, doc) in enumerate(zip(vec_ids, vec_docs)):
        rrf[doc_id] = rrf.get(doc_id, 0.0) + 1 / (K + rank + 1)
        id_to_doc[doc_id] = doc

    for rank, idx in enumerate(bm25_top_idx):
        doc_id = all_ids[idx]
        rrf[doc_id] = rrf.get(doc_id, 0.0) + 1 / (K + rank + 1)
        id_to_doc[doc_id] = all_docs[idx]

    sorted_ids = sorted(rrf, key=lambda d: rrf[d], reverse=True)
    return [id_to_doc[doc_id] for doc_id in sorted_ids[:n]]
```

`_tokenize`는 한국어(`가-힣`)와 영어·숫자를 모두 추출한다. 형태소 분석기 없이도 "또리", "생일", "2024년" 같은 단어들을 토큰으로 정확히 매칭한다.

저장된 기억이 수백 개가 아닌 이상 ChromaDB에서 전체 문서를 가져와 인메모리 BM25를 구성해도 충분히 빠르다.

---

## 2. 저장할 때도 확인받기 — Block Kit 티키타카

저장과 검색은 해결했는데, 다른 문제가 있었다. 같은 정보를 조금 다르게 말하면 중복 저장되거나 업데이트 여부를 알 수 없었다.

```
사용자: 또리 생일은 10월 27일이야 기억해줘
봇: 기억했습니다.

사용자: 아 또리 생일 다시 1월 1일로 기억해줘
봇: 기억했습니다.  ← 업데이트됐는지 새로 저장됐는지 모름
```

사용자 입장에서는 뭐가 저장됐는지 확인할 방법이 없다.

Slack Block Kit을 이용해서 저장 전에 확인 버튼을 보여주는 방식으로 바꿨다.

**신규 저장 시:**

```
봇: 💾 이렇게 기억할게요: _또리 생일: 1월 1일_ 맞나요?
    [예] [아니오]
```

**기존 기억이 있을 때 (수정):**

```
봇: 🔄 기존 기억을 수정할게요.
    • 기존: _또리 생일: 10월 27일_
    • 수정: _또리 생일: 1월 1일_
    맞나요?
    [예] [아니오]
```

구현 방식은 두 단계다.

첫째, `save_memory` 도구에서 실제로 저장하지 않고 pending 상태에 등록한다.

```python
_pending_memory: dict[str, dict] = {}  # user_id → {action, text, existing_id}

async def save_memory(text: str) -> str:
    existing_id, existing_doc = await memory_service.find_similar(user_id, text)

    if existing_id:
        if existing_doc.strip() == text.strip():
            return "이미 동일한 내용을 기억하고 있습니다."
        _pending_memory[user_id] = {"action": "update", "text": text, ...}
        confirm_text = f"🔄 기존 기억을 수정할게요.\n• 기존: _{existing_doc}_\n• 수정: _{text}_\n맞나요?"
    else:
        _pending_memory[user_id] = {"action": "new", "text": text, ...}
        confirm_text = f"💾 이렇게 기억할게요: _{text}_ 맞나요?"

    blocks = [
        {"type": "section", "text": {"type": "mrkdwn", "text": confirm_text}},
        {"type": "actions", "elements": [
            {"type": "button", "action_id": "memory_confirm", "text": {"type": "plain_text", "text": "예"}, "value": user_id, "style": "primary"},
            {"type": "button", "action_id": "memory_cancel", "text": {"type": "plain_text", "text": "아니오"}, "value": user_id},
        ]},
    ]
    await slack_service.send_message(channel_id, confirm_text, blocks=blocks)
    return "사용자에게 확인을 요청했습니다."
```

둘째, Slack Interactivity에서 버튼 클릭을 받는 `/actions` 엔드포인트를 추가한다.

```python
@router.post("/actions")
async def slack_actions(request: Request):
    body = await request.body()
    form = parse_qs(body.decode("utf-8"))
    payload = json.loads(form.get("payload", ["{}"])[0])

    action_id = payload["actions"][0]["action_id"]
    user_id = payload["actions"][0]["value"]
    channel_id = payload["container"]["channel_id"]

    if action_id == "memory_confirm":
        pending = _pending_memory.pop(user_id, None)
        if pending:
            await memory_service.store_memory(user_id, pending["text"])
            await slack_service.send_message(channel_id, "✅ 기억했습니다!")

    elif action_id == "memory_cancel":
        _pending_memory.pop(user_id, None)
        await slack_service.send_message(channel_id, "취소했습니다.")
```

Slack 앱 설정의 Interactivity & Shortcuts에 `/api/v1/slack/actions` URL을 등록해야 버튼 클릭이 서버에 전달된다.

---

## 3. 대화 자동 저장이 데이터를 오염시킨 문제

기억해달라고 말하지 않아도 대화에서 중요 정보를 자동으로 추출해 저장하는 기능을 추가했다.

```python
async def _auto_save_memory(user_id: str, user_message: str, agent_reply: str) -> None:
    prompt = f"...핵심 정보 (없으면 빈 문자열):"
    resp = await llm.ainvoke(prompt)
    extracted = resp.content.strip()
    if extracted and len(extracted) > 5:
        await memory_service.store_memory(user_id, extracted)
```

실제로 실행하고 나니 ChromaDB에 이런 것들이 저장되기 시작했다.

```
핵심 정보 (없으면 빈 문자열): 반려견 또리의 생일은 2024년 3월 30일이다.
(내용 없음)
```

**문제 ①: 프롬프트 prefix 오염**

프롬프트 끝에 `핵심 정보 (없으면 빈 문자열):`를 넣었더니 LLM이 그 문자열을 답변에 그대로 복사해 붙였다. "또리 생일 알려줘"로 검색하면 이 오염된 문서가 나오고, 에이전트는 형식이 이상한 이 텍스트를 보고 실제 정보인지 판단하지 못했다.

**문제 ②: 무의미한 응답 저장**

LLM이 기억할 정보가 없을 때 "(내용 없음)"이라고 답했는데, `len("(내용 없음)") = 8 > 5`라서 그대로 ChromaDB에 저장됐다. 쓰레기 데이터가 top-3 슬롯을 차지하면 실제 데이터가 검색에서 밀린다.

두 가지를 수정했다.

```python
async def _auto_save_memory(user_id: str, user_message: str, agent_reply: str) -> None:
    import re as _re
    prompt = (
        f"다음 대화에서 나중에 참고할 사실 정보가 있으면 '주어: 정보' 형태로 한 줄만 써줘. "
        f"사실 정보가 없으면 아무것도 쓰지 마. 설명이나 prefix 없이 정보만 써줘.\n\n"
        f"사용자: {user_message}\n비서: {agent_reply}\n\n정보:"
    )
    resp = await llm.ainvoke(prompt)
    extracted = resp.content.strip()

    # prefix 제거
    extracted = _re.sub(r'^(정보:?|사실:?|핵심\s*정보[^:]*:?)\s*', '', extracted).strip()

    # 무의미한 응답 필터
    _EMPTY = _re.compile(
        r'^[\(\[]*(없음|없어|없다|null|n/a|내용\s*없음|빈\s*문자열|없습니다)[\)\]\.]*$',
        _re.IGNORECASE,
    )
    if not extracted or len(extracted) <= 5 or _EMPTY.match(extracted):
        return  # 저장하지 않음

    await memory_service.store_memory(user_id, extracted)
```

프롬프트를 `정보:`로 단순화하고, 저장 전에 regex로 prefix를 제거한다. 그리고 "없음", "(내용 없음)", "null" 같은 무의미한 응답은 저장하지 않는다.

이미 쌓인 오염 데이터는 별도로 정리해야 한다.

```python
async def purge_junk_memories(user_id: str) -> int:
    col = _collection(user_id)
    all_data = col.get(limit=2000, include=["documents"])
    junk_ids = [
        doc_id for doc_id, doc in zip(all_data["ids"], all_data["documents"])
        if _JUNK_PATTERNS.search(doc) or len(doc.strip()) <= 5
    ]
    if junk_ids:
        col.delete(ids=junk_ids)
    return len(junk_ids)
```

관리용 엔드포인트로 노출해서 오염 데이터를 한 번에 정리할 수 있도록 했다.

```
POST /admin/memory/purge/{user_id}
→ {"deleted": 5, "user_id": "U0B8D85JQUW"}
```

---

## 4. 찾아놓고 "없다"고 하는 에이전트

BM25로 하이브리드 검색을 붙이고, 오염 데이터도 정리했다. 그런데 여전히 "또리 생일이 언제야"에 "없다"고 답하는 일이 있었다.

로그를 보면 prefetch가 3개를 찾아서 에이전트에게 전달했는데도 에이전트가 무시했다.

```
[memory] hybrid search found=3 (vec+bm25 rrf)
[agent] prefetch_memory found=3
[agent] msg[1] type=AIMessage tool_calls=0  ← 도구도 안 씀
reply: "[기억된 정보 (자동 조회)]에 또리의 생일에 대한 정보가 포함되어 있지 않습니다."
```

에이전트가 `[기억된 정보 (자동 조회)]` 블록의 내용을 읽고 직접 "없다"고 판단한 것이다. 오염 데이터 시절에 "핵심 정보 (없으면 빈 문자열):" 같은 이상한 형식에 익숙해져서 생긴 현상으로 보인다.

시스템 프롬프트에 명시적으로 지시를 추가했다.

```python
"""
- 메시지 앞에 `[기억된 정보 (자동 조회)]` 블록이 있으면 그 내용은 ChromaDB에서 이미
  검색된 신뢰할 수 있는 기억입니다. search_memory 도구를 따로 호출하지 말고
  해당 정보를 바로 사용하세요
- search_memory 도구 결과가 반환되면 그 내용을 반드시 신뢰하고 "없다"고 하지 마세요.
  형식이 어색해도 내용 안에 답이 있으면 답변하세요
"""
```

---

## 전체 흐름 정리

이번에 터진 문제들은 각각 독립적인 것 같았지만 연결되어 있었다.

```
auto_save_memory 프롬프트가 잘못됨
    → "핵심 정보 (없으면 빈 문자열):", "(내용 없음)" 등이 ChromaDB에 저장됨
    → 쓰레기 데이터 3개가 top-3 점령
    → 실제 또리 생일 데이터가 검색에서 밀려남
    → 에이전트가 "[기억된 정보]에 또리 정보 없음"이라고 판단
    → "없다"고 답변
```

루트 원인은 `auto_save_memory`의 프롬프트 설계 실수였다. 하나의 버그가 검색 실패처럼 보이는 증상을 만들었다.

| 문제 | 원인 | 해결 |
|---|---|---|
| 고유명사 검색 실패 | 임베딩이 "또리"를 의미로 파악 못함 | BM25 + 벡터 RRF 하이브리드 |
| prefix 오염 저장 | 프롬프트 끝에 질문 구문이 답변에 복사됨 | 프롬프트 단순화 + regex 제거 |
| "(내용 없음)" 저장 | `len > 5` 체크만으로 필터 부족 | 무의미 응답 패턴 regex 필터 |
| 찾아놓고 "없다"고 함 | 에이전트가 오염 형식 보고 불신 | 시스템 프롬프트에 신뢰 지시 강화 |
| 중복 수정 Block Kit | 동일 내용도 "수정할게요" 표시 | exact match 시 early return |

만들 때보다 운영하면서 디버깅할 때 더 많이 배운다.
