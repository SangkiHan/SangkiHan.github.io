---
layout: post
title: "개인 비서에 Google Calendar·Sheets 연동하기 — 서비스 계정과 삽질"
date: 2026-06-03 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Google, Calendar, Sheets, ServiceAccount, Python, OAuth]
---

일정 추가와 지출 기록을 자연어로 처리하려면 Google Calendar와 Google Sheets가 필요하다. "다음주 월요일 오후 2시 치과 예약해줘"라고 하면 실제 Calendar에 이벤트가 생기고, "스타벅스 6500원"이라고 하면 Sheets에 행이 추가되는 방식이다.

---

## 서비스 계정 방식

Google API를 쓰는 방법은 크게 두 가지다.

**OAuth2 사용자 인증**: 사용자가 직접 구글 로그인을 해서 토큰을 발급받는다. 개인 계정의 리소스에 접근할 수 있지만, 리프레시 토큰 관리가 필요하고 만료 처리도 해야 한다.

**서비스 계정**: GCP에서 발급하는 봇 계정이다. JSON 키 파일만 있으면 토큰 갱신 없이 사용 가능하다. 다만 이 계정 자체의 캘린더/시트에 접근하는 것이 기본이라, 내 개인 캘린더에 이벤트를 추가하려면 해당 서비스 계정을 캘린더에 공유해줘야 한다.

서버에서 자동으로 동작해야 하므로 서비스 계정을 선택했다.

---

## Google Calendar 연동

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build

SCOPES = [
    "https://www.googleapis.com/auth/calendar",
    "https://www.googleapis.com/auth/spreadsheets",
]

def _get_service(api_name: str, api_version: str):
    credentials_data = _load_credentials_data()
    creds = service_account.Credentials.from_service_account_info(
        credentials_data, scopes=SCOPES
    )
    return build(api_name, api_version, credentials=creds)

def add_event(title: str, date: str, time: str, description: str = "") -> str:
    service = _get_service("calendar", "v3")
    start_dt = f"{date}T{time}:00"
    end_dt = f"{date}T{str(int(time[:2]) + 1).zfill(2)}{time[2:]}:00"

    event = {
        "summary": title,
        "description": description,
        "start": {"dateTime": start_dt, "timeZone": "Asia/Seoul"},
        "end": {"dateTime": end_dt, "timeZone": "Asia/Seoul"},
    }
    created = service.events().insert(
        calendarId=settings.google_calendar_id,
        body=event,
    ).execute()
    return f"일정 등록 완료: {created.get('htmlLink')}"
```

`calendarId`에 `"primary"`를 쓰면 서비스 계정 자체의 캘린더에 이벤트가 생긴다. 내 구글 계정 캘린더에 넣으려면 `calendarId`를 내 Gmail 주소로 지정해야 한다. 처음에 이걸 몰라서 이벤트가 등록됐다고 응답은 오는데 내 캘린더에는 아무것도 안 보였다.

설정으로 분리했다:

```python
# config.py
google_calendar_id: str = "primary"  # 환경변수로 본인 Gmail 주소 지정
```

캘린더 앱에서 서비스 계정 이메일(`xxx@xxx.iam.gserviceaccount.com`)을 공유 대상으로 추가하고, `GOOGLE_CALENDAR_ID`에 본인 Gmail 주소를 넣으면 된다.

---

## credentials.json 파싱 에러

서비스 계정 키는 JSON 파일이다. Jenkins를 쓰다 보니 이 JSON을 환경변수로 주입하는 방식을 택했다. Jenkinsfile에 JSON 문자열을 변수로 저장하고, 배포 시 파일로 쓰는 방식이다.

```groovy
// Jenkinsfile (Groovy)
env.GOOGLE_CREDENTIALS_JSON = '{ "type": "service_account", ..., "private_key": "-----BEGIN PRIVATE KEY-----\nMIIE..." }'
```

Groovy 단일 따옴표 문자열에서 `\n`은 이스케이프 시퀀스가 아니라 리터럴 백슬래시+n이다. 그래서 파일에 저장되는 JSON에는 실제 줄바꿈 대신 `\n` 두 글자가 들어간다.

```python
# json.loads는 strict 모드에서 제어 문자를 허용하지 않음
json.loads(raw)  # → JSONDecodeError: Invalid control character at line 1
```

실제 `\n` 줄바꿈이 들어간 JSON은 제어 문자가 포함된 것으로 처리된다. `strict=False` 옵션으로 우회할 수 있다.

```python
def _load_credentials_data() -> dict:
    with open(path) as f:
        raw = f.read()
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return json.loads(raw, strict=False)
```

정상적인 JSON이면 첫 번째 시도에서 성공한다. Groovy에서 넘어온 `\n` 리터럴이 포함된 경우만 `strict=False`로 처리된다.

---

## Google Sheets 연동

```python
def add_expense(amount: int, category: str, memo: str = "") -> str:
    service = _get_service("sheets", "v4")
    today = date.today().isoformat()
    values = [[today, amount, category, memo]]

    service.spreadsheets().values().append(
        spreadsheetId=settings.expense_sheet_id,
        range="Sheet1!A:D",
        valueInputOption="USER_ENTERED",
        body={"values": values},
    ).execute()
    return f"지출 기록: {today} | {amount:,}원 | {category} | {memo}"
```

`valueInputOption="USER_ENTERED"`는 숫자를 숫자로, 날짜를 날짜로 인식하게 한다. `RAW`로 하면 모두 문자열로 저장된다.

월별 요약 조회는 행 전체를 읽어서 파이썬에서 집계한다.

```python
def get_monthly_summary(year_month: str = "") -> str:
    if not year_month:
        year_month = date.today().strftime("%Y-%m")

    service = _get_service("sheets", "v4")
    result = service.spreadsheets().values().get(
        spreadsheetId=settings.expense_sheet_id,
        range="Sheet1!A:D",
    ).execute()
    rows = result.get("values", [])

    summary: dict[str, int] = {}
    for row in rows:
        if len(row) < 3:
            continue
        row_date, amount_str, category = row[0], row[1], row[2]
        if not row_date.startswith(year_month):
            continue
        try:
            summary[category] = summary.get(category, 0) + int(amount_str.replace(",", ""))
        except ValueError:
            pass

    if not summary:
        return f"{year_month} 지출 내역이 없습니다."

    lines = [f"*{year_month} 지출 요약*"]
    total = 0
    for cat, amt in sorted(summary.items(), key=lambda x: -x[1]):
        lines.append(f"• {cat}: {amt:,}원")
        total += amt
    lines.append(f"• *합계: {total:,}원*")
    return "\n".join(lines)
```

Sheets API는 BigQuery 같은 집계 쿼리 기능이 없어서 전체 행을 가져온 뒤 필터링한다. 지출 데이터가 수천 건이 넘어가면 비효율적이지만, 개인 가계부 수준에서는 충분하다.

---

## 동기 함수를 비동기 에이전트에서 쓰기

Google API 클라이언트(`google-api-python-client`)는 동기 라이브러리다. 비동기 에이전트에서 그냥 호출하면 이벤트 루프가 블록된다.

```python
@langchain_tool
async def add_calendar_event(title: str, date: str, time: str, description: str = "") -> str:
    """Google Calendar에 일정을 추가합니다."""
    loop = asyncio.get_event_loop()
    result = await asyncio.wait_for(
        loop.run_in_executor(None, lambda: calendar_service.add_event(title, date, time, description)),
        timeout=45,
    )
    return result
```

`run_in_executor`로 스레드풀에서 실행한다. `None`을 넘기면 기본 ThreadPoolExecutor를 사용한다. 이렇게 하면 동기 함수를 비동기적으로 실행할 수 있다.
