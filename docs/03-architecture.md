# 3. Architecture

## Layers

The code is organized into four layers. Dependencies point **downward** only —
upper layers know about lower ones, never the reverse.

```
┌─────────────────────────────────────────────────────────┐
│ PRESENTATION   streamlit_app.py · cli.py                 │
│                (turn user intent into service calls)     │
├─────────────────────────────────────────────────────────┤
│ APPLICATION    service.py (TodoService facade)           │
│                commands.py · events.py · analytics.py    │
│                (orchestration, undo/redo, notifications) │
├─────────────────────────────────────────────────────────┤
│ DOMAIN         models.py · enums.py · specifications.py  │
│                sorting.py · exceptions.py                │
│                (the rules: what a Task is, how it moves) │
├─────────────────────────────────────────────────────────┤
│ INFRASTRUCTURE repository.py (port) · persistence.py     │
│                (where tasks are stored)                  │
└─────────────────────────────────────────────────────────┘
```

## Key idea: the port/adapter seam

`repository.py` defines `TaskRepository`, an **abstract port** — six methods
(`add`, `get`, `update`, `delete`, `list`, `exists`). `TodoService` only ever
talks to this interface. Two adapters implement it:

- `InMemoryTaskRepository` — a dict, fast, ephemeral
- `FileTaskRepository` — a dict mirrored to JSON on every write

Because the service depends on the *port*, swapping storage is a one-line change
at construction time and nothing in the business logic notices. This is the
single most important structural decision in the codebase.

```python
TodoService()                                    # in-memory (default)
TodoService(repo=FileTaskRepository("tasks.json"))  # persistent
```

## Data flow: life of `svc.complete(task_id)`

Tracing one operation shows how the layers cooperate:

1. **`TodoService.complete(id)`** (`service.py:122`) delegates to `set_status`.
2. **`set_status`** (`service.py:112`) first calls `_assert_dependencies_done` —
   the DAG guard. If any dependency isn't `DONE`, it raises `ValidationError`
   and nothing changes.
3. It wraps the change in a **`TransitionCommand`** and hands it to the
   **`CommandInvoker`** (`commands.py:141`), which calls `execute()` and pushes
   the command onto the undo stack.
4. The command calls **`Task.transition_to(DONE)`** (`models.py:136`), which
   checks the **state machine** (`can_transition`) and stamps `completed_at`.
5. The command persists via `repo.update(task)`. If this is a
   `FileTaskRepository`, the JSON file is atomically rewritten.
6. Back in the service, an **event** (`TASK_COMPLETED`) is published on the
   **`EventBus`**, notifying any subscribers.
7. If the task is recurring, `_maybe_spawn_recurrence` creates the next
   occurrence and emits `TASK_CREATED`.

Every numbered step lives in a different module, each replaceable in isolation.

## Why a facade?

Without `TodoService`, a caller would have to assemble a repository, build a
command, push it through an invoker, remember to emit events, and re-implement
the dependency guard every time. The facade collapses all of that into intent-
level methods (`add`, `complete`, `find`, `undo`) while still letting advanced
callers reach the underlying `repo`, `bus`, and `invoker` directly.

## Design patterns map

| Pattern | Where | Doc |
|---------|-------|-----|
| Facade | `TodoService` | [10](10-service.md) |
| Repository / Port-Adapter | `repository.py`, `persistence.py` | [5](05-repository-persistence.md) |
| Specification | `specifications.py` | [6](06-specifications-sorting.md) |
| Strategy | `sorting.py` | [6](06-specifications-sorting.md) |
| Command + Memento | `commands.py` | [7](07-commands-undo-redo.md) |
| Observer | `events.py` | [8](08-events.md) |
| State machine | `enums.py` + `Task.transition_to` | [4](04-domain-models.md) |

Next: [Domain Models](04-domain-models.md).
