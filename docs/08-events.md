# 8. Events

File: `src/todoapp/events.py`

A tiny **synchronous** event bus implementing the Observer pattern. It lets code
react to changes (log them, trigger side effects, update a UI) without the
service knowing who's listening.

## Event types

```python
class EventType(Enum):
    TASK_CREATED
    TASK_UPDATED
    TASK_DELETED
    STATUS_CHANGED
    TASK_COMPLETED
```

## The `Event` record

```python
@dataclass(frozen=True, slots=True)
class Event:
    type: EventType
    task_id: str
    payload: dict        # extra context, e.g. {"status": "DONE"}
    at: datetime         # UTC timestamp, auto-set
```

Immutable, so listeners can't accidentally mutate a shared event.

## `EventBus` API

| Method | Purpose |
|--------|---------|
| `subscribe(event_type, listener)` | listen to one type |
| `subscribe_all(listener)` | listen to every event |
| `publish(event)` | log + dispatch to matching listeners |
| `history` | tuple of every event ever published |

`subscribe`/`subscribe_all` **return an unsubscribe callable** — call it to
detach the listener:

```python
off = svc.bus.subscribe(EventType.TASK_COMPLETED, on_done)
# ...later...
off()   # stop listening
```

Dispatch is synchronous and ordered: type-specific listeners first, then global
ones. Every published event is appended to `history`, so the bus doubles as an
audit log.

## How the service emits

`TodoService._emit` (`service.py:217`) wraps `bus.publish`. The service fires:

| Service call | Event(s) emitted |
|--------------|------------------|
| `add` | `TASK_CREATED` |
| `update_field` / `tag` | `TASK_UPDATED` |
| `set_status` | `STATUS_CHANGED` (+ `TASK_COMPLETED` if → DONE) |
| recurrence spawn | `TASK_CREATED` (payload notes `recurred_from`) |
| `delete` | `TASK_DELETED` |

## Example: log every change

This is exactly what the CLI demo does (`cli.py:23`):

```python
def logger(ev):
    print(f"event {ev.type.name:<14} {ev.task_id}")

svc.bus.subscribe_all(logger)
svc.add("hello")     # prints: event TASK_CREATED  <id>
```

## Notes & caveats

- **Synchronous:** `publish` runs listeners inline. A slow or throwing listener
  blocks/propagates into the caller — keep them fast and defensive.
- **Not persisted:** `history` lives in memory only.
- The bus is a great extension point: wire notifications, metrics, or
  cache-invalidation here without touching core logic.

Next: [Analytics](09-analytics.md).
