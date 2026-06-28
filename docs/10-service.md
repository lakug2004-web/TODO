# 10. Service Facade (`TodoService`)

File: `src/todoapp/service.py`

`TodoService` is the **one class you normally use**. It wires together the
repository, the command invoker (undo/redo), the event bus, and the analyzer,
and exposes intent-level methods. It also owns the dependency-DAG logic.

## Construction

```python
TodoService(repo=None, bus=None, analyzer=None)
```

All dependencies are injected with sensible defaults:

- `repo` → `InMemoryTaskRepository()` if `None`
- `bus` → fresh `EventBus()`
- `analyzer` → `Analyzer()`
- always creates its own `CommandInvoker`

> The `repo is not None` check (not `repo or ...`) is deliberate: an empty
> repository is falsy because `TaskRepository.__len__` returns 0. See
> [Repository & Persistence](05-repository-persistence.md).

## API surface

### Creation & retrieval

| Method | Does |
|--------|------|
| `add(title, *, description, priority, due, tags, recurrence)` | create a task (priority accepts a `Priority` or string); emits `TASK_CREATED` |
| `get(id)` | one task (raises `TaskNotFoundError`) |
| `all()` | every task |
| `pending()` | active tasks, priority-sorted |
| `done()` | completed tasks, most recently completed first |
| `find(spec, *, sort, reverse)` | filter by a [Specification](06-specifications-sorting.md) + sort |

### Mutation (all reversible via undo/redo)

| Method | Does |
|--------|------|
| `update_field(id, field, value)` | edit one whitelisted field (`title`, `description`, `priority`, `due`, `recurrence`) |
| `set_status(id, status)` | guarded transition; DAG check when → DONE |
| `complete(id)` | shorthand for `set_status(id, DONE)` |
| `delete(id)` | remove a task |
| `bulk_complete(*ids)` | complete many atomically (`MacroCommand`) |
| `tag(id, *tags)` | add tags |

### Dependencies (DAG)

| Method | Does |
|--------|------|
| `add_dependency(id, depends_on)` | add an edge; rejects unknown targets and cycles |
| `topological_order()` | tasks ordered so dependencies come first |

### History & reporting

| Method | Does |
|--------|------|
| `undo()` / `redo()` | reverse/replay last op; return its name |
| `stats()` | a `Stats` snapshot |
| `len(svc)` | task count |

## The dependency DAG

Each task carries `dependencies: set[str]` (ids it depends on). The service
treats the whole set of tasks as a directed graph and enforces two invariants.

### 1. No cycles (`add_dependency`)

Before adding edge `task → depends_on`, `_creates_cycle` (`service.py:156`)
walks outward from `depends_on` following dependency edges; if it can reach
`task`, the edge would close a loop and `CyclicDependencyError` is raised. A
missing target raises `TaskNotFoundError`.

### 2. Can't finish before your blockers (`set_status` → DONE)

`_assert_dependencies_done` (`service.py:170`) checks every dependency is `DONE`;
if any is still open it raises `ValidationError` naming the blockers. So you
literally cannot mark a task done while it's blocked.

### Topological order — Kahn's algorithm

`topological_order()` (`service.py:181`) returns tasks so every dependency
appears before the tasks that depend on it:

1. compute in-degree (number of unmet dependencies) per task,
2. start with all zero-in-degree tasks (sorted, for deterministic output),
3. repeatedly emit one and decrement its dependents' in-degrees,
4. if not all tasks are emitted, a cycle exists → `CyclicDependencyError`.

This drives the Streamlit **Order tab** and the CLI demo's dependency section.

## Recurrence integration

When `set_status(id, DONE)` succeeds on a recurring task,
`_maybe_spawn_recurrence` (`service.py:220`) calls `Task.spawn_next()`; if it
returns a new occurrence, the service adds it and emits `TASK_CREATED` with a
`recurred_from` payload. Completing "Daily standup" today automatically creates
tomorrow's.

## Escape hatches

The facade isn't a wall: `svc.repo`, `svc.bus`, `svc.invoker`, and
`svc.analyzer` are public, so advanced code can subscribe to events, inspect
undo history, or talk to storage directly.

Next: [CLI Demo](11-cli-demo.md).
