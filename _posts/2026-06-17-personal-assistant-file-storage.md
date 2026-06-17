---
layout: post
title: "개인 비서에 Slack 파일 저장소 추가 — 업로드·카테고리·번들·시맨틱 검색"
date: 2026-06-17 10:00:00 +0900
categories: [AI, 개인비서 구축]
tags: [Slack, FastAPI, ChromaDB, Python, FileStorage, LlamaIndex]
---

Slack에서 파일을 보내면 저장해두고, 나중에 "회의록 파일 줘"라고 말하면 돌려주는 기능이 필요했다. 단순히 파일명으로만 찾으면 한 글자라도 틀리면 못 찾는다. 그래서 파일명과 날짜·종류를 ChromaDB에 임베딩해서 유사도 검색으로 찾는 방식을 선택했다.

---

## 전체 흐름

```
[파일 저장]

사용자가 Slack DM에 파일 전송
    ↓
Slack Events API → FastAPI /events
    ↓ event.files[] 감지
파일 메타데이터 수신 (url_private, name, mimetype)
    ↓
httpx로 Slack CDN에서 파일 다운로드 (bot token 인증)
    ↓
로컬 디스크 저장 (/app/user_files/{user_id}/{timestamp}_{filename})
    ↓
DB 저장 (user_files 테이블 — 메타데이터)
    ↓
ChromaDB 임베딩 저장 (파일명 + 카테고리 + 종류 + 날짜)
    ↓
Slack 확인 메시지 전송

[파일 검색·전송]

사용자: "회의록 파일 줘"
    ↓
LangGraph 에이전트 → find_file 도구 호출
    ↓
ChromaDB 벡터 유사도 검색 (쿼리 임베딩 후 코사인 유사도)
    ↓
매칭 파일 1개 → 바로 전송
매칭 파일 여러 개 → 목록 안내 후 선택
    ↓
로컬 디스크에서 파일 읽기
    ↓
Slack files_upload_v2로 DM 전송
```

---

## Slack 파일 이벤트 구조

Slack에서 파일을 전송하면 `message` 이벤트 안에 `files` 배열이 포함된다.

```json
{
  "type": "message",
  "text": "업무",
  "files": [
    {
      "name": "회의록.docx",
      "mimetype": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
      "url_private": "https://files.slack.com/files-pri/...",
      "url_private_download": "https://files.slack.com/files-pri/..."
    }
  ]
}
```

이미지 파일(`image/*`)은 기존 영수증 인식 흐름으로, 나머지는 파일 저장 흐름으로 분기한다.

```python
# slack.py
files = event.get("files", [])
if files:
    image_files = [f for f in files if f.get("mimetype", "").startswith("image/")]
    other_files = [f for f in files if not f.get("mimetype", "").startswith("image/")]

    for f in image_files:
        await agent_service.handle_receipt_image(user_id, url, channel_id)

    if other_files:
        async with AsyncSessionLocal() as db:
            if len(other_files) == 1:
                result = await file_service.handle_slack_file(db, user_id, other_files[0], channel_id, text)
            else:
                result = await file_service.handle_slack_files(db, user_id, other_files, channel_id, text)
        await slack_service.send_message(channel_id, result)
    return
```

`text`는 파일과 함께 입력한 메시지다. 카테고리 추출에 사용한다.

---

## 파일 다운로드 — bot token 인증 필수

`url_private`는 인증 없이 접근하면 403이 반환된다. Slack Bot Token을 Authorization 헤더에 포함해야 한다.

```python
async def _download(url: str) -> bytes:
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.get(
            url,
            headers={"Authorization": f"Bearer {settings.slack_bot_token}"},
        )
        resp.raise_for_status()
        return resp.content
```

이를 위해 Slack App → OAuth & Permissions에서 **`files:read`** 스코프를 추가해야 한다.

---

## 파일 저장 — 로컬 + DB + ChromaDB

### 로컬 저장

파일명 충돌을 막기 위해 `{timestamp}_{원본파일명}` 패턴으로 저장한다.

```python
def _storage_dir(user_id: str) -> Path:
    path = Path(settings.file_storage_path) / user_id
    path.mkdir(parents=True, exist_ok=True)
    return path

safe_name = f"{int(time.time())}_{filename}"
stored_path = str(_storage_dir(user_id) / safe_name)
with open(stored_path, "wb") as f:
    f.write(content_bytes)
```

같은 파일명이 이미 있으면 덮어쓴다. 새 버전을 받았을 때 자동으로 업데이트된다.

### DB 저장 (메타데이터)

```python
class UserFile(Base):
    __tablename__ = "user_files"

    id, user_id, original_name, stored_path, mimetype, size_bytes
    chroma_id   # ChromaDB 문서 ID — 삭제·업데이트 시 참조
    category    # 카테고리 (예: 업무, 개인)
    bundle_id   # 번들 FK (여러 파일 묶음)
    created_at, updated_at
```

### ChromaDB 임베딩 — 파일 내용이 아닌 메타데이터

파일 내용을 임베딩하면 이진 파일(docx, xlsx, pdf)은 처리가 복잡해진다. 대신 **파일명 + 카테고리 + 종류 + 날짜**를 텍스트로 만들어 임베딩한다.

```python
def _make_embed_text(filename, mimetype, dt, category):
    parts = [
        f"파일명: {filename}",
        f"종류: {mimetype}",
        f"날짜: {dt.strftime('%Y-%m-%d')}",
    ]
    if category:
        parts.insert(1, f"카테고리: {category}")
    return " | ".join(parts)

# 결과 예시
# "파일명: 6월 회의록.docx | 카테고리: 업무 | 종류: application/vnd.openxmlformats... | 날짜: 2026-06-17"
```

이렇게 하면 "6월 회의록"이나 "업무 문서"처럼 대략적으로 검색해도 벡터 유사도로 찾을 수 있다. 파일명을 정확히 기억하지 않아도 된다.

---

## 카테고리 자동 인식

파일과 함께 보낸 텍스트가 짧고 의미 있는 단어면 카테고리로 처리한다.

```python
_CATEGORY_PATTERN = re.compile(r'^[가-힣A-Za-z0-9 _\-]{1,15}$')
_IGNORE_WORDS = {"저장", "해줘", "이거", "파일", "문서", "보내", "올려", "넣어", "주세요", "좀"}

def _extract_category(text: str) -> str | None:
    text = text.strip()
    if not text or len(text) > 15:
        return None
    words = set(re.findall(r'[가-힣]+', text))
    if words & _IGNORE_WORDS:   # 저장 관련 단어가 섞이면 카테고리가 아님
        return None
    if _CATEGORY_PATTERN.match(text):
        return text
    return None
```

| 전송한 텍스트 | 카테고리 인식 결과 |
|---|---|
| `"업무"` | ✅ 업무 |
| `"개인 문서"` | ✅ 개인 문서 |
| `"저장해줘"` | ❌ None (무시어 포함) |
| `"이 파일 좀 저장해주세요"` | ❌ None (길이 초과 + 무시어) |

나중에 에이전트로 카테고리를 변경할 수 있다.

```
"회의록.docx 업무로 분류해줘"
→ ✅ 회의록.docx 카테고리를 업무로 설정했습니다.
```

---

## 번들 — 여러 파일 묶어서 저장

여러 파일을 한 번에 전송하면 **FileBundle**로 묶어 저장한다.

```python
class FileBundle(Base):
    __tablename__ = "file_bundles"

    id, user_id, name, category, created_at
```

각 파일의 `bundle_id`가 같은 번들을 가리킨다. 번들 단위로 검색하거나 한꺼번에 다운로드할 수 있다.

```python
async def handle_slack_files(db, user_id, slack_files, channel_id, text=""):
    category = _extract_category(text)
    bundle = FileBundle(user_id=user_id, name=bundle_name, category=category)
    db.add(bundle)
    await db.flush()  # bundle.id 확보

    for sf in slack_files:
        content = await _download(sf["url_private"])
        await save_file(db, user_id, sf["name"], content, sf["mimetype"],
                        category=category, bundle_id=bundle.id)

    await db.commit()
```

---

## 파일 검색

### ChromaDB 유사도 검색

```python
async def search_files(user_id: str, query: str, n: int = 5) -> list[str]:
    index = _get_index(user_id)
    retriever = index.as_retriever(similarity_top_k=n)
    nodes = await retriever.aretrieve(query)
    return [node.node.metadata.get("filename", "") for node in nodes]
```

쿼리도 임베딩 → 저장된 메타데이터 임베딩과 코사인 유사도 비교 → 상위 N개 파일명 반환.

### 에이전트 도구

```python
@langchain_tool
async def find_file(query: str) -> str:
    """저장된 파일을 이름, 날짜, 종류로 검색해서 Slack으로 전송합니다."""
    filenames = await file_service.search_files(user_id, query, n=5)
    matched = [await file_service.get_file_by_name(db, user_id, fn) for fn in filenames]

    if len(matched) == 1:
        content = await asyncio.to_thread(file_service.read_file_bytes, matched[0].stored_path)
        await slack_service.upload_file(channel_id, matched[0].original_name, content)
        return f"*{matched[0].original_name}* 파일을 보냈습니다."

    # 여러 개 → 목록 안내
    lines = [f"• {f.original_name} ({f.updated_at.strftime('%Y-%m-%d')})" for f in matched[:5]]
    return "찾은 파일:\n" + "\n".join(lines) + "\n더 구체적으로 알려주세요."
```

---

## 파일 전송 — files_upload_v2

Slack SDK의 `files.upload`는 deprecated됐다. `files_upload_v2`를 사용한다. 내부적으로 `files.getUploadURLExternal` → 업로드 → `files.completeUploadExternal` 흐름을 자동 처리한다.

```python
async def upload_file(channel_id: str, filename: str, file_bytes: bytes, comment: str = "") -> None:
    client = _client()
    try:
        await client.files_upload_v2(
            channel=channel_id,
            file=file_bytes,
            filename=filename,
            title=filename,
            initial_comment=comment,
        )
    except AttributeError:
        # 구버전 slack_sdk fallback
        await client.files_upload(
            channels=channel_id,
            file=file_bytes,
            filename=filename,
            initial_comment=comment,
        )
```

파일 전송에는 **`files:write`** 스코프가 필요하다. Slack App 설정에서 추가하고 워크스페이스 재설치가 필요하다.

---

## 카테고리별 일괄 다운로드 — zip 묶음

카테고리 파일이 여러 개일 때 하나씩 보내면 번거롭다. zip으로 묶어 한 번에 보낸다.

```python
def create_zip(files: list[tuple[str, bytes]]) -> bytes:
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as zf:
        for filename, content in files:
            zf.writestr(filename, content)
    return buf.getvalue()

# 에이전트 도구
@langchain_tool
async def find_files_by_category(category: str) -> str:
    files = await file_service.find_by_category(db, user_id, category)

    file_pairs = [(f.original_name, read_file_bytes(f.stored_path)) for f in files]
    zip_bytes = await asyncio.to_thread(file_service.create_zip, file_pairs)
    zip_name = f"{category}_파일묶음.zip"

    await slack_service.upload_file(channel_id, zip_name, zip_bytes,
                                    f"{category} 카테고리 파일 {len(file_pairs)}개입니다.")
```

---

## 파일 영속성 — Docker 볼륨 마운트

컨테이너 내부에만 저장하면 재배포 시 파일이 전부 사라진다. `docker-compose.yml`에 볼륨을 추가해야 한다.

```yaml
volumes:
  - /var/lib/jenkins/personal-assistant/credentials.json:/app/credentials.json:ro
  - /var/lib/jenkins/personal-assistant/user_files:/app/user_files
```

배포 전 호스트에 디렉토리를 미리 만들어둔다.

```bash
mkdir -p /var/lib/jenkins/personal-assistant/user_files
```

---

## 정리

| 항목 | 방식 | 이유 |
|---|---|---|
| 파일 다운로드 | httpx + bot token | Slack `url_private`는 인증 필요 |
| 파일 검색 | ChromaDB 벡터 유사도 | 정확한 파일명 기억 불필요 |
| 임베딩 대상 | 파일명 + 카테고리 + 날짜 | 이진 파일 내용 파싱 불필요 |
| 파일 전송 | files_upload_v2 개별 전송 | Slack에서 자연스럽게 확인 가능 |
| 카테고리 추출 | 문장 패턴 우선 + 짧은 텍스트 | LLM 호출 오버헤드 없음 |
| 다중 파일 조회 | 개별 파일 순차 전송 | 파일마다 미리보기·다운로드 가능 |
| 영속성 | Docker 볼륨 마운트 | 컨테이너 재시작 대응 |

---

## 트러블슈팅

실제로 붙여 쓰다 보니 바로 문제들이 나왔다. 기록해둔다.

---

### 1. 이미지를 전부 영수증으로 처리하던 문제

처음에는 이미지(`image/*`)와 그 외 파일을 분기해서 이미지는 영수증 OCR로 보냈다. 실제로 써보니 대시보드 캡처, 도면 사진 등 영수증과 무관한 이미지를 전부 영수증으로 처리하려고 해서 계속 오류가 났다.

영수증 자동 인식 기능 자체를 제거하고, 이미지도 다른 파일과 동일하게 파일 저장소로 처리하도록 변경했다.

```python
# 변경 전: 이미지/비이미지 분기
image_files = [f for f in files if f.get("mimetype", "").startswith("image/")]
other_files = [f for f in files if not ...]
for f in image_files:
    await agent_service.handle_receipt_image(...)  # 제거

# 변경 후: 모든 파일을 file_service로 통일
async with AsyncSessionLocal() as db:
    if len(files) == 1:
        result = await file_service.handle_slack_file(...)
    else:
        result = await file_service.handle_slack_files(...)
```

---

### 2. 카테고리 추출 — 문장 전체가 카테고리로 저장되는 버그

"KB 태양광 업무로 저장해줘"를 보냈더니 카테고리가 `KB 태양광 업무`가 아니라 `KB 태양광 업무로 저장해줘` 전체로 저장됐다.

원인: `_extract_category`에서 짧은 텍스트 분기가 먼저 실행됐는데, 15자 이하면 무조건 카테고리로 판단하는 로직이었다. "KB 태양광 업무로 저장해줘"가 딱 15자라 통과해버렸다.

수정: **문장 패턴 매칭을 먼저 실행**하고, 그다음 짧은 단어 분기를 시도하도록 순서를 바꿨다.

```python
_CATEGORY_SENTENCE_PATTERN = re.compile(
    r'([가-힣A-Za-z0-9 _\-]{1,20})(?:으로|로)\s*(?:저장|분류|파일|묶어)'
)

def _extract_category(text: str) -> str | None:
    text = text.strip()
    if not text:
        return None

    # 문장 패턴 우선: "X로 저장/분류" → X를 카테고리로
    m = _CATEGORY_SENTENCE_PATTERN.search(text)
    if m:
        cat = m.group(1).strip()
        if cat:
            return cat

    # 짧고 단순한 단어 직접 입력 (예: "업무", "개인")
    if len(text) <= 15:
        words = set(re.findall(r'[가-힣]+', text))
        if not (words & _IGNORE_WORDS) and _CATEGORY_PATTERN.match(text):
            return text

    return None
```

| 입력 | 수정 전 | 수정 후 |
|---|---|---|
| `"KB 태양광 업무로 저장해줘"` | KB 태양광 업무로 저장해줘 | KB 태양광 업무 ✅ |
| `"업무로 저장해줘"` | 업무로 저장해줘 | 업무 ✅ |
| `"업무"` | 업무 | 업무 ✅ |

---

### 3. ChromaDB 중복 항목 누적

파일 삭제 후 목록에 같은 파일이 여러 번 나왔다. 삭제 후보 목록에 동일 파일이 3개씩 나오기도 했다.

원인: `_index_file`이 ChromaDB에 문서를 추가할 때 기존 항목을 지우지 않고 `insert`를 호출해서, 같은 `chroma_id`로 중복 추가될 수 있었다.

수정: `_index_file`에서도 먼저 삭제 후 추가하도록 변경했다. (`_reindex_file`과 동일한 방식)

```python
def _index_file(user_id, chroma_id, filename, mimetype, dt, category):
    try:
        col = _client().get_or_create_collection(...)
        try:
            col.delete(ids=[chroma_id])  # 기존 항목 먼저 제거
        except Exception:
            pass
        _indexes.pop(user_id, None)
        doc = Document(text=_make_embed_text(...), id_=chroma_id, ...)
        _get_index(user_id).insert(doc)
    except Exception as e:
        logger.warning("[file] chroma index failed: %s", e)
```

추가로 삭제 후보 목록을 표시하기 전에 `original_name` 기준으로 중복을 제거하는 로직도 넣었다.

---

### 4. 파일 삭제 — 정확한 파일명 요구가 너무 엄격

"★2026년 신재생에너지(태양광) 발전량 모니터링 접속방법(59개소)_2026.02.06.xlsx"처럼 긴 파일명을 그대로 다시 입력하기는 현실적으로 어렵다. 정확히 일치해야만 삭제되다 보니 실용성이 없었다.

`difflib.get_close_matches`로 유사도 0.6 이상이면 매칭되도록 변경했다.

```python
import difflib

# 정확히 일치하면 우선 사용
if text in candidate_names:
    matched_name = text
else:
    # 퍼지 매칭
    close = difflib.get_close_matches(text, candidate_names, n=1, cutoff=0.6)
    if close:
        matched_name = close[0]
```

"태양광 xlsx 삭제" → 긴 파일명과 유사도 계산 → 자동 매칭 후 삭제.

---

### 5. 다중 파일 전송 — zip보다 개별 전송이 낫다

처음에 여러 파일을 검색하면 zip으로 묶어 전송했다. Slack에서 zip을 받으면 압축을 직접 풀어야 해서 불편했다. 파일 하나씩 메시지로 올라오는 게 미리보기도 보이고 다운로드도 편하다.

```python
# 변경 전
zip_bytes = create_zip(file_pairs)
await slack_service.upload_file(channel_id, "결과.zip", zip_bytes)

# 변경 후
for f in matched:
    content = read_file_bytes(f.stored_path)
    await slack_service.upload_file(channel_id, f.original_name, content)
```

---

➡️ 시리즈 인덱스: [Slack으로 부리는 개인 비서](/posts/personal-assistant-intro)
