# 1. Overview

## What this is

`todoapp` is an in-memory todo engine written in pure Python standard library
(no runtime dependencies). It is deliberately over-engineered for a todo list:
it exists to demonstrate a clean, layered architecture and a handful of classic
design patterns working together. A Streamlit web UI sits on top.

## Why it's structured this way

The whole codebase is organized so that **policy** (what a task is, how status
changes, how dependencies resolve) is separated from **mechanism** (where tasks
are stored, how they're displayed). That separation is what lets the same core
power both the CLI demo and the Streamlit UI, and lets storage switch from
memory to a JSON file without changing a line of business logic.

## The mental model

```
                 ┌──────────────────────────────┐
   UI / CLI ───► │        TodoService           │  ◄── the only entrypoint
                 │  (facade: wires everything)  │      you normally touch
                 └──────────────────────────────┘
                    │        │         │       │
            ┌───────┘    ┌───┘     ┌───┘    ┌──┘
            ▼            ▼         ▼        ▼
        Repository   Commands   EventBus  Analyzer
        (storage)    (undo/redo)(observer)(stats)
            │
            ▼
         Task  ◄── the aggregate root (models.py)
        / Tag / RecurrenceRule
```

You call methods on `TodoService`. It coordinates the repository, the undo/redo
invoker, the event bus, and the analytics engine. Everything else is an
implementation detail behind that facade.

## See it run

```bash
uv run todoapp          # scripted end-to-end demo (prints events + queries)
```

The demo (`cli.py`) creates tasks, wires dependencies, blocks an illegal
completion, completes work in order, spawns a recurring task, runs queries,
undoes/redoes, and prints statistics. Reading its output top-to-bottom is the
fastest way to understand the engine's capabilities. See
[CLI Demo](11-cli-demo.md) for a line-by-line walkthrough.

## Capabilities at a glance

- Create / update / delete tasks with priority, due date, tags, description
- Guarded status state machine (TODO → IN_PROGRESS → DONE, etc.)
- Full undo/redo of every mutation, plus atomic multi-step macros
- Composable queries (`&`, `|`, `~`) and named sort strategies
- Task dependencies as a DAG: cycle detection + topological order
- Recurring tasks that spawn their next occurrence on completion
- Synchronous event bus for reacting to changes
- Aggregated statistics (completion rate, overdue count, tag workload)
- Pluggable persistence (in-memory or atomic JSON file)
- Streamlit web UI over the whole thing

Next: [Getting Started](02-getting-started.md).
