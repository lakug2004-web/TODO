# 11. CLI Demo

Files: `src/todoapp/cli.py`, `src/todoapp/__init__.py` (`main`),
`src/todoapp/__main__.py`

The CLI is **not** an interactive command parser — it's a scripted, end-to-end
demonstration that exercises every part of the engine and prints what happens.
It's the single best document for "what can this thing do?".

## Running it

```bash
uv run todoapp            # via the console script (project.scripts)
uv run python -m todoapp  # via the module entrypoint
```

Both call `todoapp.main()` → `cli.main()` → `run_demo()`.

### How the entrypoints wire up

- `pyproject.toml` declares `todoapp = "todoapp:main"`, so `uv run todoapp`
  calls `__init__.main()`, which lazily imports and runs `cli.main`.
- `__main__.py` re-exports `cli.main` so `python -m todoapp` works too.

## What the demo does, step by step

`run_demo()` (`cli.py:30`) walks through, printing a banner per section:

1. **Subscribe a logger** to the event bus (`subscribe_all`) so every event
   prints as it happens — you see the Observer pattern live.
2. **Create tasks** — "Write design doc", "Review design doc", "Deploy service",
   an overdue "Buy milk", and a daily-recurring "Daily standup".
3. **Wire dependencies** — review depends on write; deploy depends on review.
4. **Topological order** — prints tasks with dependencies first.
5. **Guard demo** — tries to complete "deploy" before its deps are done; catches
   and prints the `ValidationError` ("blocked as expected").
6. **Progress work** — moves write to IN_PROGRESS, then completes write →
   review → deploy in dependency order (now allowed).
7. **Recurrence** — completes the standup; the engine spawns tomorrow's
   occurrence (you see a second `TASK_CREATED`).
8. **Queries** — high-priority active tasks; overdue OR `#home`; text search for
   "design" — all via [Specifications](06-specifications-sorting.md).
9. **Undo/redo** — undoes the last two operations, redoes one.
10. **Statistics** — prints `Stats.as_lines()`.
11. **Workload by tag** — active-task tag distribution.

Reading the printed output alongside the source is the fastest onboarding path.

## Note on persistence

The demo uses the **default in-memory** repository, so it leaves no files
behind. For a persistent experience, use the Streamlit UI (which uses
`FileTaskRepository`) or construct `TodoService(repo=FileTaskRepository(...))`
yourself — see [Repository & Persistence](05-repository-persistence.md).

Next: [Streamlit UI](12-streamlit-ui.md).
