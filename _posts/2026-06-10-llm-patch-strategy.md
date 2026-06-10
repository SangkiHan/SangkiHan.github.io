---
layout: post
title: "LLM이 코드를 수정할 때 생기는 문제들 — Self-reflection과 전체 파일 교체 전략"
date: 2026-06-10 14:00:00 +0900
categories: [AI, RAG]
tags: [LLM, 코드패치, Self-reflection, Python, FastAPI]
---

에러 분석 에이전트의 가장 핵심적인 기능은 수정 코드를 GitHub PR로 자동 생성하는 것이다. LLM이 `before`/`after` 스니펫을 제안하면, 실제 파일에서 `before`를 찾아 `after`로 교체한다.

이 패칭 과정에서 겪은 문제들과 최종 해결 방법을 정리한다.

---

## 처음 겪은 문제: string 매칭 실패

LLM이 제안하는 `before` 코드가 실제 파일의 코드와 미묘하게 달라서 substring 매칭이 실패했다.

```
실제 파일:
    public String triggerNpe() {

LLM이 제안한 before:
    public String triggerNpe(@RequestParam(required = false) String value)
    {
```

줄 끝 공백, CRLF vs LF, 들여쓰기 차이, 심지어 없는 코드를 hallucinate하는 경우까지. 4단계 fallback을 만들었다.

```python
def _find_and_replace(original, before, after):
    # 1차: 정확한 매칭
    if before in original:
        return original.replace(before, after, 1)

    # 2차: CRLF → LF 정규화
    if before_lf in orig_lf:
        return orig_lf.replace(before_lf, after_lf, 1)

    # 3차: 들여쓰기 보정 (4/8/2칸 시도)
    for indent in ("    ", "        ", "  ", ...):
        ...

    # 4차: 라인 단위 fuzzy 매칭 (strip 후 순서 비교)
    return _fuzzy_replace(orig_lf, before_lf, after_lf)
```

---

## Self-reflection 도입

4단계 fallback으로도 해결 안 되는 경우가 있었다. LLM이 완전히 없는 코드를 hallucinate하면 4단계 모두 실패한다.

그래서 **Self-reflection**을 추가했다. LLM이 자신이 제안한 `before`가 실제 파일에 있는지 스스로 검증하고, 없으면 파일을 다시 보고 올바른 `before`를 찾는 단계다.

```python
async def _self_reflect(suggestion: str, server_id: int) -> str:
    # before가 파일에 있으면 통과
    if norm(before) in norm(file_content):
        logger.info("[reflect] before verified ✓")
        return suggestion

    # 없으면 LLM에게 올바른 before 찾기 요청
    prompt = (
        f"파일 내용:\n{file_content[:3000]}\n\n"
        f"수정 의도 before (틀릴 수 있음):\n{before}\n\n"
        "파일에서 실제 before에 해당하는 코드를 찾아줘.\n"
        "응답은 JSON만: {\"before\": \"...\"}"
    )
    # LLM 응답으로 before 교정
    ...
```

---

## Self-reflection의 한계

실제로 동작시켜보니 Self-reflection이 before를 "교정"해도 결과가 망가지는 경우가 생겼다. PR에 올라온 코드가 이랬다:

```java
// 원본
@GetMapping("/npe")
public String triggerNpe() {
    String value = null;
    return value.toUpperCase();
}

// PR에 올라온 코드 (망가진 결과)
@GetMapping("/npe")
public String triggerNpe() {
    String value = null;
    return value.toUpperCase();
}     @GetMapping("/npe")
    public String triggerNpe(@RequestParam(required = false) String value)
    {
        return (value != null) ? value.toUpperCase() : "";
    }
    String value = null;
    return value.toUpperCase();
}
```

원인을 분석하면:

1. LLM이 `before`로 메서드 시그니처만 제안 (닫는 중괄호 없음)
2. Self-reflection이 파일에서 그 시그니처를 찾아 "교정 성공"이라고 판단
3. 하지만 교정된 before가 메서드 전체가 아닌 시그니처 일부만이어서
4. `after`로 교체 시 원본 코드가 제거되지 않고 새 코드만 추가됨
5. 결과적으로 코드 중복

string 교체는 **before가 파일 내용과 완벽하게 일치해야만** 올바르게 동작한다. LLM이 "비슷한" before를 찾아줘도 완벽하지 않으면 코드가 망가진다.

---

## 근본적인 해결: 전체 파일 교체

string 매칭 자체가 너무 취약하다. 방향을 바꿨다.

**분석 시점에 LLM이 수정된 전체 파일을 생성해서 저장하고, PR 시에는 그 파일 전체를 push한다.**

```
이전 방식:
  LLM → before/after 스니펫
  → 파일에서 before 찾기 → after로 교체
  → 실패 시 재시도... 반복

새 방식:
  LLM → before/after 스니펫
  → string 매칭 성공: 바로 저장
  → 실패: LLM에게 원본 파일 전체 + before/after 전달
          "이 파일에 이 변경을 적용한 전체 파일을 반환해줘"
  → LLM이 완성된 파일 반환 → 저장
  → 승인 클릭 시 저장된 파일 그대로 push
```

```python
async def _enrich_with_patched_content(suggestion: str, server_id: int) -> str:
    original = full_path.read_text(encoding="utf-8", errors="replace")
    data["file_patch"]["original_content"] = original

    if before and after:
        # 1차: 빠른 string 매칭
        patched = _find_and_replace(original, before, after)
        if patched is None:
            # 2차: LLM이 원본 파일 전체를 보고 직접 수정
            patched = await apply_patch_with_llm(original, before, after)

        if patched:
            data["file_patch"]["patched_content"] = patched
```

```python
# ollama_service.py
async def apply_patch_with_llm(original: str, before: str, after: str) -> str:
    prompt = (
        "아래 파일에 코드 변경을 적용해줘. "
        "설명 없이 변경이 적용된 파일 전체 내용만 반환해.\n\n"
        f"=== 원본 파일 ===\n{original}\n\n"
        f"=== 변경 전 (Before) ===\n{before}\n\n"
        f"=== 변경 후 (After) ===\n{after}"
    )
    # LLM이 수정된 전체 파일 반환
    ...
```

LLM은 원본 파일 전체를 보고 수정 의도를 파악하기 때문에 before가 조금 틀려도 올바른 위치를 찾아 적용한다. string 매칭으로는 절대 해결 못하는 hallucination 문제를 LLM으로 우회한다.

---

## 최종 파이프라인

```
에러 발생
  ↓
LLM 분석 → before/after JSON 생성
  ↓
_enrich_with_patched_content (분석 시점)
  ├── string 매칭 성공 → patched_content 저장
  └── 실패 → LLM에게 전체 파일 수정 요청 → patched_content 저장
  ↓
Slack 전송 (수락/거절 버튼)
  ↓
수락 클릭
  └── patched_content 그대로 GitHub push → PR 생성
```

승인 시점에는 더 이상 매칭 로직이 없다. 분석 시점에 이미 올바른 파일이 준비되어 있다.

---

## Self-reflection이 남긴 것

Self-reflection은 제거했지만, 개념적으로 중요한 패턴이다.

LLM이 자신의 출력을 스스로 검증하는 루프 — 이것이 Agentic AI의 핵심 패턴 중 하나다. 이번 구현에서는 "before 코드를 검증"하는 데 썼지만, 실제로는 더 넓은 맥락에서 사용된다.

- 코드 생성 후 컴파일 가능한지 확인
- 요약 후 원문의 정보가 누락되지 않았는지 확인
- 번역 후 의미가 보존됐는지 확인

"LLM이 틀릴 수 있다"는 전제에서 시작해 **틀린 걸 스스로 고치는 루프를 만드는 것**이 Self-reflection의 본질이다. 이번에는 루프 안에서도 string 매칭이라는 취약한 단계가 있어서 실패했지만, 패치 적용을 LLM에게 위임하는 방식으로 그 취약점을 제거했다.
