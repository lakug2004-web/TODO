# 4. Domain Models

Files: `src/todoapp/enums.py`, `src/todoapp/models.py`, `src/todoapp/exceptions.py`

This is the heart of the domain — the data and the rules that protect it.

## Enums (`enums.py`)

### `Priority` (IntEnum)

```python
TRIVIAL=0  LOW=1  MEDIUM=2  HIGH=3  CRITICAL=4
```

It's an `IntEnum` on purpose: higher value = more urgent, so priorities sort
numerically (`-int(task.priority)` gives "highest first"). Helpers:

- `Priority.from_str("high")` → `Priority.HIGH` (case-insensitive; raises
  `ValueError` on unknown). Used so the UI/CLI can accept plain strings.
- `Priority.label` → `"High"` for display.

### `Status` (Enum)

```python
TODO  IN_PROGRESS  BLOCKED  DONE  ARCHIVED
```

Two derived properties drive filtering logic everywhere:

- `is_terminal` → `True` for `DONE`/`ARCHIVED`
- `is_active` → `True` for `TODO`/`IN_PROGRESS`/`BLOCKED`

### The state machine

`ALLOWED_TRANSITIONS` (`enums.py:57`) is a table of which status can move to
which. `can_transition(src, dst)` consults it (a status can always "transition"
to itself). Examples of what the table forbids:

- `DONE → IN_PROGRESS` is **not** allowed (a done task can only reopen to `TODO`
  or be `ARCHIVED`).
- `BLOCKED → DONE` is **not** allowed (you must unblock first).

This is the guard that `Task.transition_to` enforces — see below.

### `RecurrenceUnit`

`DAILY WEEKLY MONTHLY YEARLY` — consumed by `RecurrenceRule`.

## Models (`models.py`)

### `Tag` — immutable, normalized label

```python
@dataclass(frozen=True, slots=True, order=True)
class Tag:
    name: str
```

`__post_init__` normalizes: strips whitespace, lowercases, drops a leading `#`.
Empty or whitespace-containing names raise `ValidationError`. Because it's
`frozen` and normalized, `Tag("Work")`, `Tag("work")`, and `Tag("#work")` are
all equal and hashable — so a task's `tags` is a `set[Tag]` with no duplicates.
`str(tag)` renders as `#work`.

### `RecurrenceRule` — how a task repeats

```python
RecurrenceRule(unit=RecurrenceUnit.WEEKLY, interval=2)   # every 2 weeks
```

`interval` must be ≥ 1. `next_after(anchor_date)` computes the next due date,
with care for calendar edge cases:

- **Monthly** clamps the day (Jan 31 + 1 month → Feb 28/29, not an error).
- **Yearly** handles Feb 29 → Feb 28 on non-leap years.

### `Task` — the aggregate root

The central mutable object. Notable fields:

| Field | Notes |
|-------|-------|
| `title` | required; stripped; empty raises `ValidationError` |
| `priority` | defaults `MEDIUM` |
| `status` | defaults `TODO` |
| `due` | optional `date` |
| `tags` | `set[Tag]` |
| `dependencies` | `set[str]` of other task **ids** |
| `recurrence` | optional `RecurrenceRule` |
| `id` | auto: 12-char hex (`uuid4`) |
| `created_at`/`updated_at`/`completed_at` | UTC timestamps |
| `_seq` | monotonic counter, a stable tiebreaker for sorting |

**Why `_seq`?** Timestamps can collide at sub-second resolution. `_seq` is a
process-monotonic integer used as the final sort key so ordering is always
deterministic even for tasks created in the same instant.

#### Mutation helpers all "touch"

`rename`, `set_priority`, `add_tag`, `remove_tag`, `add_dependency`,
`remove_dependency`, and `transition_to` each call `self.touch()` to update
`updated_at`. Always mutate through these helpers, not raw attribute writes, so
the timestamp stays honest.

#### `transition_to(dst)` — the guarded mover

```python
def transition_to(self, dst):
    if not can_transition(self.status, dst):
        raise InvalidTransitionError(self.status, dst)
    self.status = dst
    self.completed_at = _utcnow() if dst is Status.DONE else None
    self.touch()
```

This is where the state machine is enforced and `completed_at` is set/cleared.

#### Derived properties

- `is_done` — status is `DONE`
- `is_overdue` — has a past due date **and** is not terminal
- `days_until_due` — signed int, or `None`
- `age_days` — days since creation

#### `spawn_next()` — recurrence engine

If the task has both a `recurrence` and a `due`, returns a fresh `Task` (new id,
status reset to `TODO`, due advanced via `recurrence.next_after`, dependencies
**not** carried over). Returns `None` otherwise. The service calls this on
completion (see [Service](10-service.md)).

#### Serialization

`to_dict()` is a display/JSON-friendly view. The *lossless* round-trip pair used
by persistence is `task_to_record`/`record_to_task` in `persistence.py` — see
[Repository & Persistence](05-repository-persistence.md).

## Exceptions (`exceptions.py`)

A small hierarchy rooted at `TodoError`, so callers can catch one base type:

```
TodoError
├── TaskNotFoundError
├── DuplicateTaskError
├── InvalidTransitionError      # illegal status move
├── ValidationError             # invariant broken (empty title, bad tag, blocked complete…)
├── CyclicDependencyError       # dependency would form a cycle
├── NothingToUndoError
└── NothingToRedoError
```

The Streamlit UI catches `TodoError` broadly to show friendly messages.

Next: [Repository & Persistence](05-repository-persistence.md).
