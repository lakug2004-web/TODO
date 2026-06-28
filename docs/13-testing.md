# 13. Testing

Files: `tests/test_todoapp.py`, `tests/test_persistence.py`

## Running

```bash
uv run pytest -q                       # everything
uv run pytest tests/test_persistence.py
uv run pytest -k undo                   # match test names
uv run pytest -q -x                     # stop on first failure
```

## What's covered

- **`test_todoapp.py`** — engine behaviour through `TodoService`: creating and
  finding tasks, the status state machine (legal and illegal transitions),
  undo/redo, specifications and sorting, the dependency DAG (cycle rejection,
  topological order, the "can't complete while blocked" guard), recurrence
  spawning, and analytics.
- **`test_persistence.py`** — the JSON adapter: that `task_to_record` /
  `record_to_task` round-trip losslessly, that `FileTaskRepository` autosaves and
  reloads in a fresh instance, atomic-write behaviour, and schema-version
  handling.

## How tests are structured

Most tests construct a `TodoService()` with the default in-memory repository,
exercise it through the public API, and assert on the result — mirroring how
real callers use the engine. Persistence tests use `tmp_path` (pytest's
temp-dir fixture) so they never touch the project's real `tasks.json`.

## Writing a new test

Pattern to copy:

```python
from todoapp import TodoService, Priority, Status, ByStatus

def test_complete_blocks_until_deps_done():
    svc = TodoService()
    a = svc.add("dep")
    b = svc.add("blocked")
    svc.add_dependency(b.id, a.id)

    import pytest
    from todoapp import ValidationError
    with pytest.raises(ValidationError):
        svc.complete(b.id)          # a not done yet

    svc.complete(a.id)
    svc.complete(b.id)              # now allowed
    assert svc.get(b.id).status is Status.DONE
```

Guidelines:

- Prefer driving through `TodoService` over poking internals — tests stay valid
  across refactors.
- For storage tests, always use `tmp_path`, never a hard-coded path.
- One behaviour per test; name it after the behaviour, not the method.
- When adding a feature, add a test for both the happy path and the guard it
  introduces (e.g. the error it should raise).

Next: [Glossary & Patterns](14-glossary-patterns.md).
