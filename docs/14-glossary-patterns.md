# 14. Glossary & Patterns

A quick reference: every design pattern used, where it lives, and the domain
vocabulary — so a new developer can map a term to a file fast.

## Design patterns

| Pattern | Where | One-line purpose |
|---------|-------|------------------|
| **Facade** | `TodoService` (`service.py`) | Single, simple entrypoint over many subsystems |
| **Repository** | `TaskRepository` (`repository.py`) | Abstract storage so logic doesn't know where data lives |
| **Port / Adapter** | port in `repository.py`; adapters `InMemory…` + `FileTaskRepository` | Swap implementations behind one interface |
| **Specification** | `specifications.py` | Composable, declarative query predicates (`&`, `\|`, `~`) |
| **Strategy** | `sorting.py` | Interchangeable, named sort algorithms |
| **Command** | `commands.py` | Encapsulate a reversible operation (`execute`/`undo`) |
| **Memento** | inside each command (saved old state) | Capture state to restore on undo |
| **Composite** | `MacroCommand`, `AndSpec`/`OrSpec` | Treat groups of objects like a single one |
| **Observer** | `EventBus` (`events.py`) | Broadcast changes to decoupled listeners |
| **State machine** | `ALLOWED_TRANSITIONS` + `Task.transition_to` | Enforce legal status changes |
| **Dependency injection** | `TodoService.__init__` | Pass in repo/bus/analyzer; default if omitted |
| **DAG + topological sort** | `service.py` (Kahn's algorithm) | Order tasks by dependency; detect cycles |

## Domain glossary

| Term | Meaning |
|------|---------|
| **Task** | The aggregate root — one todo item (`models.py`) |
| **Aggregate root** | The main object other data hangs off; mutated via its methods |
| **Tag** | Immutable, normalized (`#work`) label on a task |
| **Priority** | `TRIVIAL…CRITICAL`; an `IntEnum`, so it sorts numerically |
| **Status** | Lifecycle state: `TODO`, `IN_PROGRESS`, `BLOCKED`, `DONE`, `ARCHIVED` |
| **Terminal status** | `DONE` or `ARCHIVED` (`status.is_terminal`) |
| **Active status** | `TODO`/`IN_PROGRESS`/`BLOCKED` (`status.is_active`) |
| **Transition** | A status change, allowed only per `ALLOWED_TRANSITIONS` |
| **Overdue** | Past-due *and* not terminal |
| **Recurrence** | Rule that spawns the next occurrence when a task completes |
| **Dependency** | A task id this task must wait on before it can be `DONE` |
| **DAG** | Directed Acyclic Graph — the dependency network (no cycles allowed) |
| **Topological order** | Linear order where every dependency precedes its dependents |
| **Specification / Spec** | A composable predicate used to filter tasks |
| **Command** | A reversible unit of work on the repository |
| **Invoker** | `CommandInvoker` — runs commands, keeps undo/redo stacks |
| **Macro** | A command bundling several commands atomically |
| **Event** | An immutable record of something that happened |
| **Port** | An abstract interface (e.g. `TaskRepository`) |
| **Adapter** | A concrete implementation of a port |
| **`_seq`** | Monotonic per-task counter; deterministic sort tiebreaker |

## Where to look first, by question

| "How do I…" | Start in |
|-------------|----------|
| …understand the public API? | `service.py` + [Service doc](10-service.md) |
| …change what a task is? | `models.py` / `enums.py` |
| …add a new query filter? | `specifications.py` |
| …add a new sort order? | `sorting.py` |
| …add a reversible operation? | `commands.py` + `service.py` |
| …react to changes? | `events.py` |
| …add a metric/report? | `analytics.py` |
| …change storage? | `repository.py` / `persistence.py` |
| …change the web UI? | `streamlit_app.py` |

Back to the [index](README.md).
