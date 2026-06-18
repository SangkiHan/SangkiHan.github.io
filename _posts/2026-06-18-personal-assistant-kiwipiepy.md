---
layout: post
title: "개인 비서 한국어 처리 개선 — kiwipiepy 형태소 분석기 도입"
date: 2026-06-18 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Python, kiwipiepy, NLP, Korean, BM25, PersonalAssistant]
---

정규식으로 한국어를 파싱하는 로직이 프로젝트 여러 곳에 흩어져 있었다. 카테고리 추출, BM25 토크나이저, 긍정/부정 판별 — 모두 정규식 패턴과 하드코딩된 단어 목록에 의존했다. 실제로 쓰다 보면 "코오롱 업무**에** 저장해줘"처럼 조사 하나가 바뀌면 버그가 생겼고, "좋아요"나 "맞아요" 같은 활용형은 YES로 인식하지 못했다.

kiwipiepy를 도입해서 이런 한국어 파싱 로직을 형태소 분석 기반으로 교체했다.

---

## kiwipiepy란?

카카오에서 만든 한국어 형태소 분석기다. C++ 백엔드로 빠르고, 한국어 명사·동사·조사를 형태소 단위로 분리해준다.

```python
from kiwipiepy import Kiwi

kiwi = Kiwi()
tokens = kiwi.tokenize("코오롱 업무에 저장해줘")
# Token(form='코오롱', tag='NNP', ...)
# Token(form='업무',  tag='NNG', ...)
# Token(form='에',   tag='JKB', ...)  ← 부사격 조사
# Token(form='저장', tag='NNG', ...)
# Token(form='하',   tag='XSV', ...)
# Token(form='아',   tag='EC',  ...)
# Token(form='줘',   tag='EF',  ...)
```

정규식은 "으로|로|에" 같은 특정 패턴만 잡을 수 있지만, 형태소 분석기는 **어떤 조사·어미가 오더라도 품사(tag)로 구분**한다.

---

## 싱글톤 공유

Kiwi 인스턴스 초기화는 언어 모델 로딩 때문에 ~1초가 걸린다. 세 서비스가 각자 인스턴스를 만들지 않도록 `app/core/kiwi.py`에 공유 싱글톤을 만들었다.

```python
# app/core/kiwi.py
from kiwipiepy import Kiwi

_instance: Kiwi | None = None

def get() -> Kiwi:
    global _instance
    if _instance is None:
        _instance = Kiwi()
    return _instance
```

첫 호출에 한 번만 로드되고, 이후 호출은 캐시된 인스턴스를 반환한다.

---

## 1. 카테고리 추출 — file_service.py

파일과 함께 보낸 텍스트에서 카테고리를 추출하는 로직이다.

### Before

```python
_CATEGORY_PATTERN = re.compile(r'^[가-힣A-Za-z0-9 _\-]{1,20}$')
_IGNORE_WORDS = {"저장", "해줘", "이거", "파일", "문서", "보내", "올려", "넣어", "주세요", "좀"}
_CATEGORY_SENTENCE_PATTERN = re.compile(
    r'([가-힣A-Za-z0-9 _\-]{1,20})(?:으로|로|에)\s*(?:저장|분류|파일|묶어)'
)

def _extract_category(text: str) -> str | None:
    text = text.strip()
    if not text:
        return None

    m = _CATEGORY_SENTENCE_PATTERN.search(text)
    if m:
        cat = m.group(1).strip()
        if cat:
            return cat

    if len(text) <= 15:
        words = set(re.findall(r'[가-힣]+', text))
        if not (words & _IGNORE_WORDS) and _CATEGORY_PATTERN.match(text):
            return text

    return None
```

문제:
- 조사 변형(에/로/으로)을 일일이 나열해야 함
- `_IGNORE_WORDS`로 걸러도 "저장해줘"가 짧으면 통과하는 버그 발생
- 새로운 표현("에게 저장해줘", "폴더에 넣어줘")이 나오면 패턴 추가 필요

### After

```python
from app.core import kiwi as _kiwi_mod

_CAT_ACTION_NOUNS = {"저장", "분류", "묶"}
_CAT_PARTICLES = {"에", "로", "으로"}
_CAT_NOUN_TAGS = {"NNG", "NNP", "SL"}

def _extract_category(text: str) -> str | None:
    text = text.strip()
    if not text:
        return None

    try:
        tokens = _kiwi_mod.get().tokenize(text)

        # "X에/로/으로 저장/분류/묶" 패턴 — 조사 앞 명사구를 카테고리로 추출
        for i, tok in enumerate(tokens):
            if tok.form in _CAT_PARTICLES:
                following = {t.form for t in tokens[i + 1: i + 4]}
                if following & _CAT_ACTION_NOUNS:
                    noun_parts = [t.form for t in tokens[:i] if t.tag in _CAT_NOUN_TAGS]
                    if noun_parts:
                        return " ".join(noun_parts).strip()

        # 짧은 텍스트에서 용언·어미 없이 명사만 있을 때 직접 카테고리로
        has_predicate = any(t.tag.startswith(("V", "E", "X")) for t in tokens)
        if not has_predicate and len(text) <= 15:
            noun_parts = [t.form for t in tokens if t.tag in _CAT_NOUN_TAGS]
            if noun_parts:
                return " ".join(noun_parts).strip()
    except Exception:
        pass

    return None
```

| 입력 | Before | After |
|---|---|---|
| `"코오롱 업무에 저장해줘"` | 코오롱 업무에 저장해줘 ❌ | 코오롱 업무 ✅ |
| `"KB 태양광 업무로 저장해줘"` | KB 태양광 업무 ✅ | KB 태양광 업무 ✅ |
| `"업무"` | 업무 ✅ | 업무 ✅ |
| `"저장해줘"` | 저장해줘 ❌ (버그) | None ✅ |

포인트: 조사(`에`, `로`, `으로`)와 행위 명사(`저장`, `분류`) 사이의 관계를 형태소 tag로 판단하므로, 새로운 표현이 오더라도 명사구를 올바르게 추출한다.

---

## 2. BM25 토크나이저 — memory_service.py

하이브리드 검색(벡터 + BM25)에서 BM25 인덱싱과 쿼리에 쓰는 토크나이저다.

### Before

```python
def _tokenize(text: str) -> list[str]:
    tokens = re.findall(r"[A-Za-z가-힣0-9]+", text.lower())
    korean_words = re.findall(r"[가-힣]+", text)
    extra: list[str] = []
    for word in korean_words:
        for i in range(len(word) - 1):
            extra.append(word[i : i + 2])  # 2-gram
        extra.extend(list(word))            # 음절 단위
    return tokens + extra
```

"내차가" → `["내차가", "내차", "차가", "내", "차", "가"]`

"차"로 검색 시 "차가"·"차" 토큰으로 매칭은 되지만, "이"·"가"·"의" 같은 공통 음절 노이즈가 많아 BM25 IDF 계산이 오염됐다.

### After

```python
_BM25_KEEP_TAGS = {"NNG", "NNP", "VV", "VA", "SL", "XR"}

def _tokenize(text: str) -> list[str]:
    try:
        morphs = _kiwi_mod.get().tokenize(text)
        result = [t.form.lower() for t in morphs if t.tag in _BM25_KEEP_TAGS and len(t.form) > 1]
    except Exception:
        result = []
    result += re.findall(r"[a-z0-9]+", text.lower())
    return list(dict.fromkeys(result))
```

"내차가" → 형태소: `["내(MM)", "차(NNG)", "가(JKS)"]` → keep: `["차"]`

조사("가", "은", "이")와 어미("어", "는", "고")가 자동으로 제거되고 의미 있는 명사·동사만 토큰으로 남는다. BM25 IDF 계산 품질이 올라가고, "또리"·"차" 같은 고유명사·단어 검색 정확도가 향상된다.

---

## 3. 정크 메모리 감지 — memory_service.py

LLM 응답이 ChromaDB에 오염 데이터로 저장되는 것을 방지하는 필터다.

### Before

```python
_JUNK_PATTERNS = re.compile(
    r'(핵심\s*정보[^:]*:|없으면\s*빈\s*문자열|내용\s*없음|없음$|없습니다$|\(내용\s*없음\))',
    re.IGNORECASE,
)

junk_ids = [
    doc_id for doc_id, doc in zip(ids, documents)
    if _JUNK_PATTERNS.search(doc) or len(doc.strip()) <= 5
]
```

"없습니다", "없음"만 잡음. "해당 정보가 없어요", "잘 모르겠어요" 같은 활용형은 통과.

### After

```python
_JUNK_NEG_STEMS = {"없", "모르", "불명"}

def _is_junk_doc(doc: str) -> bool:
    if _JUNK_PATTERNS.search(doc) or _REALTIME_PATTERNS.search(doc) or len(doc.strip()) <= 5:
        return True
    # 짧은 부정 응답 감지 — "없-" "모르-" 어간 형태소 분석
    if len(doc) <= 30:
        try:
            tokens = _kiwi_mod.get().tokenize(doc)
            if any(t.form in _JUNK_NEG_STEMS for t in tokens):
                return True
        except Exception:
            pass
    return False
```

| 입력 | Before | After |
|---|---|---|
| `"없습니다"` | 정크 감지 ✅ | 정크 감지 ✅ |
| `"해당 정보가 없어요"` | 통과 ❌ | 정크 감지 ✅ |
| `"잘 모르겠어요"` | 통과 ❌ | 정크 감지 ✅ |

---

## 4. 긍정/부정 판별 — slack.py

파일 삭제 확인, 지출 기록 확인 등에서 사용자 응답을 긍정/부정으로 판별하는 로직이다.

### Before

```python
_YES = {"예", "네", "ㅇㅇ", "응", "좋아", "맞아", "ㅇ", "ok", "OK", "그래"}
_NO = {"아니오", "아니", "아냐", "ㄴ", "취소", "no", "No", "NO", "싫어"}

if text in _YES:
    ...
if text in _NO:
    ...
```

정확히 일치해야만 인식. "좋아요", "맞아요", "아니에요"처럼 활용형이 붙으면 인식 불가.

### After

```python
_YES_STEMS = {"응", "네", "맞", "좋"}
_NO_STEMS = {"아니", "싫"}

def _is_affirmative(text: str) -> bool:
    if text in _YES or text.lower() in {"ok", "yes"}:
        return True
    try:
        tokens = _kiwi_mod.get().tokenize(text)
        return any(t.form in _YES_STEMS for t in tokens)
    except Exception:
        return False

def _is_negative(text: str) -> bool:
    if text in _NO:
        return True
    try:
        tokens = _kiwi_mod.get().tokenize(text)
        return any(t.form in _NO_STEMS for t in tokens)
    except Exception:
        return False
```

| 입력 | Before | After |
|---|---|---|
| `"좋아요"` | 인식 불가 ❌ | 긍정 ✅ |
| `"맞아요"` | 인식 불가 ❌ | 긍정 ✅ |
| `"아니에요"` | 인식 불가 ❌ | 부정 ✅ |
| `"응"` | 긍정 ✅ | 긍정 ✅ |

---

## 정리

| 위치 | 교체 전 | 교체 후 |
|---|---|---|
| `file_service._extract_category` | 조사 변형 정규식 + 무시어 목록 | 형태소 분석으로 명사구·조사·행위동사 구분 |
| `memory_service._tokenize` | 수동 2-gram + 음절 분해 | 형태소 단위 명사·동사 추출 |
| `memory_service._is_junk_doc` | 특정 문자열 정규식 | 어간 감지로 활용형 부정 응답 포함 |
| `slack._is_affirmative/negative` | 키워드 정확 일치 | 어간 매칭으로 활용형 처리 |

kiwipiepy는 한 번 인스턴스를 만들어두면 여러 곳에서 공유할 수 있고, 첫 호출 이후 각 tokenize 호출은 ~1ms 수준이다. 정규식 패턴을 계속 추가·유지보수하는 것보다 "형태소 분석기에 위임"하는 방식이 훨씬 견고하다.

---

➡️ 시리즈 인덱스: [Slack으로 부리는 개인 비서](/posts/personal-assistant-intro)
