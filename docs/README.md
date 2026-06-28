# todoapp — Developer Documentation

In-depth docs for the `todoapp` engine and its Streamlit UI. Start here, then
read in order — each page builds on the previous one.

## Reading order

| # | Doc | What you'll learn |
|---|-----|-------------------|
| 1 | [Overview](01-overview.md) | What the project is, the demo, the mental model |
| 2 | [Getting Started](02-getting-started.md) | Install, run, test, project layout |
| 3 | [Architecture](03-architecture.md) | How the layers fit; data flow of one operation |
| 4 | [Domain Models](04-domain-models.md) | `enums.py`, `models.py` — `Task`, `Tag`, `RecurrenceRule` |
| 5 | [Repository & Persistence](05-repository-persistence.md) | The storage port, in-memory + JSON adapters |
| 6 | [Specifications & Sorting](06-specifications-sorting.md) | Composable filters and named sort strategies |
| 7 | [Commands & Undo/Redo](07-commands-undo-redo.md) | Command pattern, invoker, macro rollback |
| 8 | [Events](08-events.md) | The synchronous observer bus |
| 9 | [Analytics](09-analytics.md) | `Analyzer` → `Stats` aggregations |
| 10 | [Service Facade](10-service.md) | `TodoService`, the dependency DAG, topological sort |
| 11 | [CLI Demo](11-cli-demo.md) | The scripted end-to-end walkthrough |
| 12 | [Streamlit UI](12-streamlit-ui.md) | The web frontend, wiring, and each tab |
| 13 | [Testing](13-testing.md) | What the suite covers and how to extend it |
| 14 | [Glossary & Patterns](14-glossary-patterns.md) | Every design pattern, named and located |

## One-paragraph summary

`todoapp` is a pure-stdlib task engine layered cleanly behind a single facade
(`TodoService`). Tasks live in a repository (in-memory or JSON-file). Every
mutation goes through a reversible **Command**, giving free undo/redo. Queries
are built from composable **Specifications**. Tasks form a dependency **DAG**
with cycle detection and topological ordering. A synchronous **EventBus**
broadcasts changes. `streamlit_app.py` is a thin UI over that same facade.

## Conventions in these docs

- `path:line` references point at the exact source location.
- Code blocks are runnable unless marked otherwise.
- "Port" = an abstract interface; "adapter" = a concrete implementation of it.
