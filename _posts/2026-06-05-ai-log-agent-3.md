---
title: "AI Log Analysis Agent 구축기 (3) — FastAPI 분석 파이프라인 & Git 서비스"
date: 2026-06-05 02:00:00 +0900
categories: [Project, DevSecOps]
tags: [fastapi, python, git, dulwich, asyncio]
---

## 웹훅 수신 & 파이프라인 분리

웹훅을 받으면 **즉시 200을 반환**하고, 분석은 `BackgroundTasks`로 실행한다.
Ollama 분석에 1~2분이 걸리기 때문에, 이렇게 하지 않으면 Spring Boot 측에서 타임아웃이 발생한다.

```python
@router.post("/error")
async def receive_error(
    payload: ErrorEventPayload,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    # server_ip로 등록된 서버 조회
    result = await db.execute(
        select(Server)
        .join(ServerHost, ServerHost.server_id == Server.id)
        .where(ServerHost.host == payload.server_ip, Server.is_active == True)
    )
    server = result.scalar_one_or_none()

    if not server:
        return {"status": "ignored"}

    # 60초 내 동일 에러 중복 방지
    key = _dedup_key(payload.server_ip, payload.stack_trace)
    now = asyncio.get_event_loop().time()
    if key in _recent_errors and now - _recent_errors[key] < 60:
        return {"status": "deduplicated"}
    _recent_errors[key] = now

    background_tasks.add_task(_analysis_pipeline, server, payload)
    return {"status": "received"}   # 즉시 반환
```

중복 방지 로직도 넣었다. 동일 에러가 60초 내 반복 발생하면 무시한다.

---

## 분석 파이프라인

```python
async def _analysis_pipeline(server: Server, payload: ErrorEventPayload) -> None:
    async with AsyncSessionLocal() as db:
        raw_log = _build_raw_log(payload)
        source_files: dict[str, str] = {}

        if payload.stack_trace:
            try:
                # 1. git fetch (없으면 자동 clone)
                await asyncio.to_thread(
                    git_service.fetch,
                    server.id, server.git_repo_url, server.git_branch, server.github_token or ""
                )
                # 2. 원격 브랜치 HEAD 커밋 조회
                commit = await asyncio.to_thread(
                    git_service.get_remote_head, server.id, server.git_branch
                )
                # 3. 스택 트레이스에서 소스파일 추출
                source_files = await asyncio.to_thread(
                    git_service.read_files_at_commit, server.id, commit, payload.stack_trace
                )
            except Exception as e:
                logger.warning("[pipeline] git step failed: %s", e)

        # 4. Ollama 분석
        suggestion = await ollama_service.analyze_log(raw_log, source_files)

        # 5. DB 저장 & Slack 전송
        record = AnalysisRecord(server_id=server.id, ...)
        db.add(record)
        await db.commit()
        await slack_service.send_analysis(server, record)
```

git 단계가 실패해도 파이프라인은 계속 진행된다.
소스파일 없이도 스택 트레이스만으로 Ollama가 분석할 수 있기 때문이다.

---

## Git 서비스 — dulwich (순수 Python)

처음에는 `subprocess`로 `git` 바이너리를 호출했다.
하지만 Docker 빌드 환경에서 **HTTP 포트 80이 차단**되어 apt-get으로 git을 설치할 수 없었다.

해결책: **dulwich** — 시스템 git 없이 순수 Python으로 git 프로토콜을 구현한 라이브러리.

```python
from dulwich import porcelain
from dulwich.object_store import tree_lookup_path
from dulwich.repo import Repo

def clone(server_id: int, repo_url: str, branch: str, token: str = "") -> None:
    path = _repo_path(server_id)
    if path.exists():
        return
    path.parent.mkdir(parents=True, exist_ok=True)
    porcelain.clone(
        _auth_url(repo_url, token),
        str(path),
        branch=branch.encode(),
        errstream=io.BytesIO(),
    )

def fetch(server_id: int, repo_url: str = "", branch: str = "main", token: str = "") -> None:
    path = _repo_path(server_id)
    if not path.exists():
        # 최초 웹훅 수신 시 자동 clone
        clone(server_id, repo_url, branch, token)
        return
    with Repo(str(path)) as repo:
        porcelain.fetch(repo, remote_location=_auth_url(repo_url, token), errstream=io.BytesIO())
```

---

## 핵심 — 배포 시점 소스코드를 정확히 읽기

`git pull`로 최신 코드를 받으면 **배포된 버전과 다를 수 있다**.
원격 브랜치의 HEAD 커밋을 기준으로 파일을 읽어야 한다.

```python
def get_remote_head(server_id: int, branch: str) -> str:
    with Repo(str(_repo_path(server_id))) as repo:
        ref_key = f"refs/remotes/origin/{branch}".encode()
        sha = repo.refs[ref_key]
        return sha.decode("ascii")   # dulwich는 hex SHA를 ASCII bytes로 반환

def read_files_at_commit(server_id: int, commit_hash: str, stack_trace: str) -> dict[str, str]:
    paths = _extract_source_paths(stack_trace)
    result: dict[str, str] = {}

    with Repo(str(_repo_path(server_id))) as repo:
        commit = repo[bytes.fromhex(commit_hash)]
        for rel_path in paths:
            try:
                _mode, blob_sha = tree_lookup_path(
                    repo.object_store.__getitem__,
                    commit.tree,
                    rel_path.encode(),
                )
                content = repo[blob_sha].data.decode("utf-8", errors="replace")
                result[rel_path] = content[:2000]
            except Exception:
                pass
    return result
```

working tree를 전혀 건드리지 않고 object store에서 직접 파일을 읽는다.

---

## 스택 트레이스에서 소스 경로 추출

```python
def _extract_source_paths(stack_trace: str) -> list[str]:
    paths: list[str] = []

    # Java/Kotlin: at com.puppynote.service.UserService.getUser(UserService.java:42)
    java_pattern = re.compile(r"at ([\w$.]+)\.\w+\((\w+\.(?:java|kt)):\d+\)")
    for match in java_pattern.finditer(stack_trace):
        package = match.group(1)
        filename = match.group(2)
        package_path = package.split("$")[0].rsplit(".", 1)[0].replace(".", "/")
        paths.append(f"src/main/java/{package_path}/{filename}")
        paths.append(f"src/main/kotlin/{package_path}/{filename}")

    return list(dict.fromkeys(paths))[:10]
```

예: `at com.puppynoteserver.global.TestErrorController.triggerNpe(TestErrorController.java:21)`
→ `src/main/java/com/puppynoteserver/global/TestErrorController.java`

---

## 정리

| 단계 | 내용 |
|---|---|
| 웹훅 수신 | 즉시 200 반환, BackgroundTask로 파이프라인 분리 |
| 중복 방지 | 60초 내 동일 에러 스킵 (dedup hash) |
| git fetch | dulwich 순수 Python — 시스템 git 불필요 |
| 커밋 기준 | `refs/remotes/origin/{branch}` HEAD → 배포 시점 코드 정확히 조회 |
| 파일 읽기 | `tree_lookup_path` + object store — working tree 무수정 |
