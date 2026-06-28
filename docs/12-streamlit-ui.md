# 12. Streamlit UI

File: `src/streamlit_app.py`

A thin web frontend over `TodoService`. It contains **no business logic** — every
action calls a service method and re-renders. This is the Presentation layer; if
something behaves "wrong", the cause is almost always in the engine, not here.

## Run it

```bash
uv run streamlit run src/streamlit_app.py     # → http://localhost:8501
```

## How it wires to the engine

### Import path (src layout)

`streamlit_app.py` lives in `src/`, beside the `todoapp` package. At the top it
prepends its own directory to `sys.path`:

```python
SRC = Path(__file__).parent            # .../src
if str(SRC) not in sys.path:
    sys.path.insert(0, str(SRC))
```

So `import todoapp` resolves whether or not the package is pip-installed —
running the file directly Just Works.

### One persistent service

```python
@st.cache_resource
def get_service() -> TodoService:
    return TodoService(repo=FileTaskRepository(DATA_FILE))
```

`@st.cache_resource` makes this a **singleton across reruns** (Streamlit re-runs
the whole script on every interaction). So there's one service, one repository,
one in-memory undo/redo stack for the whole session. `DATA_FILE` is
`tasks.json` at the **project root** (`Path(__file__).parent.parent`), and
`FileTaskRepository` autosaves there atomically — state survives restarts.

### The rerun pattern

Every mutating widget callback does `svc.<mutation>(...)` then `st.rerun()` to
refresh the view from the (now-changed) service. Errors are caught as
`TodoError` and shown via `st.error` / `st.sidebar.error`, so invalid input
(empty title, illegal transition, completing a blocked task, cyclic dependency)
surfaces as a friendly message instead of a crash.

## Layout

### Sidebar

- **➕ Add task** — a form (title, description, priority, optional due date,
  comma-separated tags, optional recurrence with unit + interval). Submits via
  `svc.add(...)`. Tags are split on commas and whitespace-trimmed.
- **↩️ History** — Undo / Redo buttons calling `svc.undo()` / `svc.redo()`;
  empty-stack errors are shown as warnings.

### Tab 1 — 📋 Tasks

- **Filters** build a [Specification](06-specifications-sorting.md) dynamically:
  status multiselect (`ByStatus`), min-priority (`ByPriority`), tag (`HasTag`),
  overdue-only (`IsOverdue`), free text (`TextMatches`) — `&`-chained — plus a
  sort selector. Results come from `svc.find(spec, sort=...)`.
- Each task renders in an expander showing description, id, status, tags, deps,
  with controls: a status selector + **Apply** (`set_status`), **✅ Complete**
  (`complete`), **🗑️ Delete** (`delete`).

### Tab 2 — 📊 Stats

Renders `svc.stats()`: metric cards (total, completion %, overdue, done) and bar
charts of `by_status` and `by_priority`, plus a top-tags caption. See
[Analytics](09-analytics.md).

### Tab 3 — 🔗 Order

- Lists `svc.topological_order()` (dependencies first); a cycle is reported as an
  error.
- **Add dependency** — pick a task and what it depends on; calls
  `svc.add_dependency(...)`, which rejects cycles. See [Service](10-service.md).

## Widget keys

Per-task widgets use keys like `f"st_{task.id}"`, `f"done_{task.id}"`. Streamlit
requires unique keys for repeated widgets; deriving them from the task id keeps
state correct across reruns even as the list changes.

## Extending the UI

Because the UI only calls the facade, adding a feature usually means: add the
method to `TodoService` (with a Command if it mutates), then add a widget here
that calls it and reruns. Keep logic in the engine; keep this file declarative.

Next: [Testing](13-testing.md).
