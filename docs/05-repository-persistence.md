# 5. Repository & Persistence

Files: `src/todoapp/repository.py`, `src/todoapp/persistence.py`

## The port: `TaskRepository`

An abstract base class defining the storage contract. Six abstract methods:

```python
add(task)      get(id)      update(task)
delete(id)     list()       exists(id)
```

Plus dunder conveniences with default implementations: `__len__`, `__iter__`,
`__contains__`. Any class implementing the six methods is a drop-in store.

> **Gotcha (documented in the service):** `__len__` makes an empty repository
> *falsy*. That's why `TodoService.__init__` checks `repo is not None` instead
> of `repo or InMemoryTaskRepository()` — `or` would replace a passed-in empty
> repo. See `service.py:39`.

## Adapter 1: `InMemoryTaskRepository`

A `dict[str, Task]`. Insertion order is preserved (Python dicts are ordered).
Fast, simple, ephemeral — the default when you call `TodoService()`.

Two extra methods beyond the port, used by undo/redo restore semantics:

- `snapshot()` → deep copy of the entire store
- `restore(snapshot)` → replace the store from a deep copy

`add` raises `DuplicateTaskError` on id collision; `get`/`update`/`delete` raise
`TaskNotFoundError` on a missing id.

## Adapter 2: `FileTaskRepository`

Same port, but every mutation is mirrored to a JSON file (write-through cache):
tasks live in an in-memory dict for fast reads and are flushed to disk on each
`add`/`update`/`delete`.

```python
svc = TodoService(repo=FileTaskRepository("tasks.json"))
svc.add("survives a restart")        # autosaved atomically

# fresh process, same file:
svc2 = TodoService(repo=FileTaskRepository("tasks.json"))
assert len(svc2) == 1                # loaded back from JSON
```

### Autosave

`autosave=True` (default) flushes on every write. Pass `autosave=False` to batch
changes and call `.save()` manually.

### Atomic writes — why you never get a truncated file

`_write_atomic` (`persistence.py:153`) writes to a temp file in the same
directory, then `os.replace`s it over the target. `os.replace` is atomic on
POSIX and Windows, so a crash mid-write leaves either the old complete file or
the new complete file — never a half-written one. On any error the temp file is
cleaned up.

### Schema versioning

The file is `{"version": 1, "tasks": [...]}`. `SCHEMA_VERSION = 1`. On load, a
mismatched version raises `ValidationError`, giving you a clean hook for future
migrations instead of silently misreading old data.

## Serialization: `task_to_record` / `record_to_task`

These are the **lossless** round-trip pair (distinct from `Task.to_dict`, which
is display-oriented):

- `task_to_record(task)` → dict with `priority` as int, `status`/recurrence unit
  as names, dates as ISO strings, and `_seq` preserved.
- `record_to_task(rec)` → rebuilds the exact `Task`, including timestamps and
  `_seq`. Corrupt records (missing keys, bad values) raise `ValidationError`
  wrapping the underlying cause.

Preserving `_seq` matters: it keeps sort order stable across save/load.

## Writing your own adapter

To back storage with SQLite, Redis, or an API: subclass `TaskRepository`,
implement the six methods, and pass an instance to `TodoService(repo=...)`.
Nothing else changes. That's the payoff of the port/adapter seam.

Next: [Specifications & Sorting](06-specifications-sorting.md).
