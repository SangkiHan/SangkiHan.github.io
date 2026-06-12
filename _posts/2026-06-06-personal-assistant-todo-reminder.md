---
layout: post
title: "개인 비서 — 할 일 목록과 리마인더 구현"
date: 2026-06-06 10:00:00 +0900
categories: [AI, Side-Project]
tags: [APScheduler, SQLAlchemy, MySQL, Todo, Reminder, FastAPI, Python]
---

일정(Calendar)은 특정 날짜의 이벤트고, 할 일(Todo)은 완료해야 하는 작업이고, 리마인더(Reminder)는 특정 시간에 알림을 보내는 것이다. 세 가지가 비슷해 보이지만 실제 사용 패턴은 다르다.

- "다음주 화요일 오전 10시 치과" → Calendar (날짜와 시간이 있는 이벤트)
- "오늘 PR 리뷰해야 해" → Todo (완료 처리가 필요한 작업)
- "30분 후에 약 먹으라고 알려줘" → Reminder (시간 기반 알림)

---

## 데이터 모델

```python
# todo.py
class Todo(Base):
    __tablename__ = "todos"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    user_id: Mapped[str] = mapped_column(String(50))
    content: Mapped[str] = mapped_column(Text)
    done: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)


# reminder.py
class Reminder(Base):
    __tablename__ = "reminders"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    slack_user_id: Mapped[str] = mapped_column(String(50))
    message: Mapped[str] = mapped_column(Text)
    run_at: Mapped[datetime] = mapped_column(DateTime)
    fired: Mapped[bool] = mapped_column(Boolean, default=False)
```

`fired` 필드가 중요하다. 리마인더를 취소할 때 행을 삭제하지 않고 `fired=True`로 표시한다. 삭제하면 "아까 설정했던 리마인더 취소해줘"라는 요청에서 에이전트가 어떤 리마인더를 취소해야 할지 목록을 보여줄 수가 없다.

---

## 할 일 서비스

```python
async def list_todos(db: AsyncSession, user_id: str) -> str:
    today_start = datetime.now().replace(hour=0, minute=0, second=0, microsecond=0)
    result = await db.execute(
        select(Todo)
        .where(Todo.user_id == user_id, Todo.created_at >= today_start)
        .order_by(Todo.created_at)
    )
    todos = result.scalars().all()
    if not todos:
        return "오늘 등록된 할 일이 없습니다."

    display = ["*오늘 할 일*"]
    refs = []
    for t in todos:
        check = "✅" if t.done else "⬜"
        display.append(f"{check} {t.content}")
        if not t.done:
            refs.append(f"id={t.id}: {t.content}")

    if refs:
        display.append("\n[AGENT_ONLY - 사용자에게 표시 금지]\n" + "\n".join(refs))
    return "\n".join(display)
```

처음엔 `done=False`인 항목만 반환했다. "오늘 할 일이 뭐야?"라고 물으면 아직 못 한 것들만 보여줬는데, 이미 끝낸 것들을 포함해서 오늘 전체 현황을 보는 게 더 유용하다. `created_at >= today_start`로 오늘 생성된 전체 항목을 가져오고 ✅/⬜로 구분한다.

`[AGENT_ONLY]` 섹션을 별도로 붙이는 이유는 에이전트가 `complete_todo(todo_id=N)` 호출 시 DB 기본키가 필요하기 때문이다. 그런데 사용자 화면에는 숫자 ID가 보일 필요가 없다. 에이전트는 도구 응답에서 ID를 읽어 사용하고, 시스템 프롬프트에서 `[AGENT_ONLY]` 섹션을 사용자에게 노출하지 말라고 지시한다.

---

## APScheduler로 리마인더 실행

```python
async def _fire_reminders() -> None:
    from app.models.reminder import Reminder
    from app.services import slack_service

    async with AsyncSessionLocal() as db:
        now = datetime.now()
        result = await db.execute(
            select(Reminder).where(
                Reminder.run_at <= now,
                Reminder.fired == False,
            )
        )
        reminders = result.scalars().all()
        for r in reminders:
            await slack_service.send_dm(r.slack_user_id, f"⏰ 리마인더: {r.message}")
            r.fired = True
        if reminders:
            await db.commit()


def start_scheduler() -> None:
    global _scheduler
    _scheduler = AsyncIOScheduler(timezone="Asia/Seoul")
    _scheduler.add_job(_fire_reminders, "interval", minutes=1)
    _scheduler.start()
```

1분마다 DB를 polling해서 `run_at <= 지금`이고 아직 `fired=False`인 리마인더를 찾아 Slack DM을 보낸다.

실시간 정확도가 1분 단위인 셈이다. "30분 후에 알려줘"를 29분 59초에 설정하면 30분 1초~31분 사이에 알림이 온다. 개인 비서 용도로는 충분하다.

`AsyncIOScheduler`의 `timezone="Asia/Seoul"` 설정이 중요하다. 컨테이너의 TZ 설정과 스케줄러 타임존이 다르면 예상치 못한 시간에 알림이 간다. `docker-compose.yml`에도 `TZ: Asia/Seoul`을 환경변수로 명시했다.

---

## 시간 파싱

"30분 후", "1시간 후"를 datetime으로 변환해야 한다.

```python
@langchain_tool
async def set_reminder(message: str, datetime_str: str) -> str:
    """특정 시간에 Slack 알림을 설정합니다. datetime_str: 'YYYY-MM-DD HH:MM' 또는 '30분 후', '1시간 후'"""
    import re
    from datetime import datetime, timedelta

    now = datetime.now()
    m = re.search(r"(\d+)\s*(분|시간)", datetime_str)
    if m:
        val, unit = int(m.group(1)), m.group(2)
        run_at = now + (timedelta(minutes=val) if unit == "분" else timedelta(hours=val))
    else:
        try:
            run_at = datetime.strptime(datetime_str, "%Y-%m-%d %H:%M")
        except ValueError:
            return "시간 형식을 인식하지 못했습니다. 예: '30분 후', '2026-06-12 15:00'"
    ...
```

LLM에게 시간 계산을 맡기는 방식과, 도구에서 직접 파싱하는 방식 중 후자를 선택했다. LLM이 "오늘 오후 7시 30분"을 `2026-06-12 19:30`으로 바꿔서 넘겨주면, 도구는 그냥 `datetime.strptime`으로 파싱한다. 시스템 프롬프트에 날짜 변환 규칙을 명시해두면 LLM이 날짜를 계산해서 포맷에 맞게 전달한다.

---

## 리마인더 조회·취소

처음엔 `set_reminder`만 있었다. 설정은 할 수 있지만 뭐가 있는지 조회하거나 취소하는 기능이 없었다. "오늘 등록된 알림 뭐가 있어?" → "없습니다"라고 답했다. DB를 보지 않고 그냥 답한 것이다.

도구 세트가 부족하면 에이전트는 아무것도 없다고 답해버린다. 필요한 기능을 도구로 명시적으로 등록해야 에이전트가 쓸 수 있다.

```python
async def list_reminders(db, slack_user_id: str) -> str:
    result = await db.execute(
        select(Reminder).where(
            Reminder.slack_user_id == slack_user_id,
            Reminder.fired == False,
        ).order_by(Reminder.run_at)
    )
    reminders = result.scalars().all()
    if not reminders:
        return "등록된 리마인더가 없습니다."
    lines = [f"• #{r.id} {r.run_at.strftime('%m/%d %H:%M')} — {r.message}" for r in reminders]
    return "\n".join(lines)


async def cancel_reminder(db, slack_user_id: str, reminder_id: int) -> str:
    result = await db.execute(
        select(Reminder).where(
            Reminder.id == reminder_id,
            Reminder.slack_user_id == slack_user_id,
            Reminder.fired == False,
        )
    )
    r = result.scalar_one_or_none()
    if not r:
        return f"리마인더 #{reminder_id}를 찾을 수 없습니다."
    r.fired = True
    await db.commit()
    return f"리마인더 #{reminder_id} '{r.message}' 취소됨."
```

`list_reminders` 출력에 `#N` ID를 포함한다. "19시 30분 리마인더를 18시로 수정해줘"라는 요청이 오면, 에이전트는 먼저 `list_reminders`로 목록을 확인하고 → `cancel_reminder(id=N)`으로 취소하고 → `set_reminder`로 새로 등록한다. 세 번의 도구 호출이 연쇄적으로 일어나는 시퀀스다.

---

## recursion_limit 조정

이 시퀀스 때문에 한 번 오류가 났다.

```
list_reminders 호출 → 2 스텝
cancel_reminder 호출 → 2 스텝
set_reminder 호출 → 2 스텝
최종 응답 → 1 스텝  ← recursion_limit=6 초과
```

"Sorry, need more steps to process this request."라는 응답이 왔다. LangGraph가 스텝 한계에 도달하면 반환하는 메시지다. `recursion_limit`을 6에서 20으로 올렸다. 도구 9회 호출까지 처리 가능한 수치다.

리마인더 수정 요청은 단순해 보이지만 내부에서 3개의 도구가 연속으로 호출된다. 에이전트를 운영하면서 "이 요청에 몇 개의 도구가 필요한가"를 로그로 확인하고 `recursion_limit`을 조정하는 게 중요하다.
