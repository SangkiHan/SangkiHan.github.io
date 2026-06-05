---
title: "AI Log Analysis Agent 구축기 (4) — Ollama LLM 연동 & Slack 알림"
date: 2026-06-05 03:00:00 +0900
categories: [Project, DevSecOps]
tags: [ollama, llm, slack, python, fastapi]
---

## Ollama — 로컬 LLM 분석

M4 Mac mini (24GB)에서 `gemma4:12b` 모델을 Ollama로 실행한다.
외부 API 없이 **완전히 로컬에서 LLM 분석**이 가능하다.

```python
SYSTEM_PROMPT = """You are a senior DevSecOps engineer analyzing server error logs.
When given a log excerpt, respond ONLY in this JSON structure (no markdown, no code fences):
{
  "error_cause": "<brief root cause in Korean>",
  "bottleneck": "<suspected bottleneck or affected component>",
  "suggested_fix": "<corrected code block in markdown>",
  "commit_message": "<concise git commit message starting with fix:>"
}"""

async def analyze_log(raw_log: str, source_files: dict[str, str] | None = None) -> str:
    parts = [f"다음 에러 로그를 분석해줘:\n\n```\n{raw_log[:4000]}\n```"]

    if source_files:
        parts.append("\n\n관련 소스 파일:")
        for path, content in source_files.items():
            parts.append(f"\n### {path}\n```\n{content[:2000]}\n```")

    async with httpx.AsyncClient(timeout=120) as client:
        response = await client.post(
            f"{settings.ollama_host}/api/generate",
            json={
                "model": settings.ollama_model,
                "system": SYSTEM_PROMPT,
                "prompt": "".join(parts),
                "stream": False,
            },
        )
        raw = response.json().get("response", "")
        return _strip_code_fence(raw)
```

소스파일이 있으면 컨텍스트에 포함시킨다. 없어도 스택 트레이스만으로 분석이 가능하다.

---

## LLM 응답 정제

`gemma4:12b` 모델은 JSON을 마크다운 코드블록으로 감싸서 반환하는 경우가 있다.

```python
def _strip_code_fence(text: str) -> str:
    text = text.strip()
    text = re.sub(r"^```(?:json)?\s*", "", text)
    text = re.sub(r"\s*```$", "", text)
    return text.strip()
```

`format: "json"` 옵션을 써봤는데, 해당 모델에서는 thinking 과정이 노출되는 부작용이 있어서 제거했다.

```
# format: json 사용 시 발생하는 문제
{"thought": "The user wants a JSON response..."}<|tool_response>
```

시스템 프롬프트에 `no markdown, no code fences` 를 명시하고 `_strip_code_fence`로 후처리하는 방식이 가장 안정적이었다.

---

## Slack 알림 구성

Slack Incoming Webhook으로 분석 결과를 발송한다.
Block Kit을 사용해 보기 좋은 포맷으로 구성했다.

```python
def _build_blocks(server: Server, record: AnalysisRecord) -> list[dict]:
    suggestion = json.loads(record.llm_suggestion or "{}")

    approve_url = f"{settings.public_url}/api/v1/analysis/approve/{record.id}"
    reject_url = f"{settings.public_url}/api/v1/analysis/reject/{record.id}"

    return [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": f"🔴 [{server.name}] 에러 감지"},
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": f"*원인*\n{suggestion.get('error_cause', '')}"},
                {"type": "mrkdwn", "text": f"*병목 지점*\n{suggestion.get('bottleneck', 'N/A')}"},
            ],
        },
        {
            "type": "section",
            "text": {"type": "mrkdwn",
                     "text": f"*추천 수정 코드*\n{suggestion.get('suggested_fix', '')[:1200]}"},
        },
        {"type": "divider"},
        {
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "✅ 수락 & Push"},
                    "style": "primary",
                    "url": approve_url,
                },
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "❌ 거절"},
                    "style": "danger",
                    "url": reject_url,
                },
            ],
        },
    ]
```

버튼은 **링크 버튼**으로 구성했다. 클릭하면 브라우저에서 approve/reject API가 호출된다.

---

## 수락 / 거절 처리

```python
@router.get("/approve/{record_id}", response_class=HTMLResponse)
async def approve_fix(record_id: int, db: AsyncSession = Depends(get_db)):
    record = await db.get(AnalysisRecord, record_id)
    if not record or record.status != "pending":
        return _html_page("⚠️ 이미 처리됨", f"현재 상태: {record.status}")

    record.status = "approved"
    await db.commit()

    await slack_service.send_result(f"✅ [{server.name}] 분석 결과 수락됨")
    return _html_page("✅ 수락됨", "분석 결과가 수락되었습니다.")
```

수락/거절 시 HTML 페이지를 반환해 브라우저에서 결과를 확인할 수 있다.

---

## Slack Interactive Webhook (선택적)

버튼 클릭 시 Slack이 서버로 POST를 보내는 **Interactive** 방식도 지원한다.
Slack 앱 설정에서 Interactivity URL을 등록해야 한다.

```python
@router.post("/slack/actions")
async def handle_slack_action(request: Request):
    body = await request.body()
    parsed = parse_qs(body.decode())
    data = json.loads(parsed.get("payload", ["{}"])[0])

    action = data.get("actions", [{}])[0]
    action_id = action.get("action_id", "")
    record_id = int(action.get("value", "0"))

    async with AsyncSessionLocal() as db:
        record = await db.get(AnalysisRecord, record_id)
        if record and record.status == "pending":
            record.status = "approved" if action_id == "approve_fix" else "rejected"
            await db.commit()

    return Response(status_code=200)
```

`python-multipart` 없이 표준 라이브러리 `urllib.parse.parse_qs`로 form 데이터를 파싱한다.

---

## 실제 Slack 알림 결과

```
🔴 [puppynote] 에러 감지

원인: null 값이 포함된 String 객체에 .toUpperCase() 메서드를
      호출하여 발생한 NullPointerException
병목 지점: TestErrorController.java:21

추천 수정 코드:
// 방법 1: Optional 활용 (권장)
String result = Optional.ofNullable(value)
    .map(String::toUpperCase)
    .orElse("");

[✅ 수락 & Push]  [❌ 거절]
```

---

## 정리

| 포인트 | 내용 |
|---|---|
| 로컬 LLM | Ollama gemma4:12b — 외부 API 비용 없음 |
| 응답 정제 | `_strip_code_fence`로 마크다운 코드블록 제거 |
| Slack 버튼 | 링크 버튼으로 브라우저에서 수락/거절 |
| form 파싱 | 표준 라이브러리만 사용 (python-multipart 불필요) |
| 비동기 | httpx AsyncClient — FastAPI와 네이티브 비동기 |
