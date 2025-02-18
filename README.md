# _#TaskLet_
> Let there be task!

### Interface prototype
```python
# tl/events.py
import typing

import tasklet as tl

class UserUpdate(tl.Event, type="user_updated"):
    user_id: int
    diff: dict[
        str, tuple[typing.Any, typing.Any]  # field: (before, after)
    ]
```

```python
# tl/history.py
import uuid

import tasklet as tl

from . import events
from src.models import BaseEvent, UserEvent


def get_database_event(event: tl.Event):
    match event.type:
        case events.UserUpdate.type:
            return UserEvent(
                id=event.id,
                user_id=event.user_id,
                handled=event.handled,
                subtype=events.UserUpdate.type,
                data={
                    "diff": event.diff
                },
                created_at=event.created_at,
            )
        case _:
            raise RuntimeError(f"Unknown event: {event}")


def restore_user_event(event: UserEvent) -> tl.Event:
    match event.subtype:
        case events.UserUpdate.type:
            return events.UserUpdate(
                id=event.id,
                user_id=event.user_id,
                handled=event.handled,
                diff=event.data["diff"],
                created_at=event.created_at
            )


class History(tl.HistoryBackend):
    def __init__(self, session_holder):
        self.session_holder = session_holder

    async def save(self, event: tl.Event) -> None:
        async with self.session_holder.session() as session:
            session.add(get_database_event(event))
            await session.commit()

    async def restore(self, id: uuid.UUID) -> tl.Event:
        async with self.session_holder.session() as session:
            event = await session.get(BaseEvent, id)

            match event.type:
                case "user_event":
                    return restore_user_event(event)

```

```python
# tl/handlers.py
import tasklet as tl
from sqlalchemy.asyncio import AsyncSession

import src
from . import events

handle = tl.Handle()

@handle.subscribe(tl.Startup)
async def tl_startup(event: tl.Startup, scope: tl.Scope):
    await src.database.session_holder.init(
        "postgresql://dev:password@localhost:5432/database"
    )
    scope.update(
        session=src.database.session_holder.session
    )
    print(
        f"TaskLet stated on {event.host} using {event.backend.type} as backend"
    )
    
@handle.subscribe(tl.Shutdown)
async def tl_shutdown(event: tl.Shutdown):
    await src.database.session_holder.dispose()
    print(
        f"TaskLet stopped"
    )

@handle.subscribe(events.UserUpdate)
async def user_updated(event: events.UserUpdate, session: AsyncSession):
    """Do something on user update"""

    return {
        "value": 123
    }
```

```python
# tasklet.py
import tasklet
import asyncio

import tl
import src

server = tasklet.TaskLetServer(
    backend=tasklet.load_backend("redis://localhost:6379"),
    history_backend=tl.history.History(src.database.session_holder),
    handle=tl.handlers.handle
)

if __name__ == "__main__":
    asyncio.run(server.start())
```

```python
# main.py
import asyncio

import tasklet as tl

import src
from tl import events

tasklet = tl.TaskLetApp(
    backend=tl.load_backend("redis://localhost:6379"),
)

async def main():
    src.database.session_holder.init(
        "postgresql://dev:password@localhost:5432/database"
    )
    event = events.UserUpdate(
        user_id=0,
        diff={
            "name": ("kuyugama", "Kuyugama")
        }
    )
    
    await tasklet.fire(event)
    
    info = await tasklet.query_event_info(event.id)
    
    assert info.event.id == event.id
    assert info.handled is False
    assert len(info.handle_info) == 0
    
    handle_info = await tasklet.query_handle_info(event.id, handler="user_updated")
    
    assert handle_info.success
    assert handle_info.handler == "user_updated"
    assert handle_info.data == {"value": 123}


if __name__ == "__main__":
    asyncio.run(main())
```