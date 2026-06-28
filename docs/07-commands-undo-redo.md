# 7. Commands & Undo/Redo

File: `src/todoapp/commands.py`

Every mutation in the system is modelled as a **Command** — an object that knows
how to both *do* and *undo* one unit of work. That's what makes undo/redo free
and uniform across the whole app.

## The `Command` base

```python
class Command(ABC):
    name: str = "command"
    def execute(self) -> object: ...   # do it, return a result
    def undo(self) -> None: ...        # reverse it
```

Each concrete command captures whatever it needs to reverse itself (the
**Memento** idea — it remembers the prior state).

## Concrete commands

| Command | `execute` | `undo` (how it reverses) |
|---------|-----------|--------------------------|
| `AddTaskCommand` | `repo.add(task)` | `repo.delete(task.id)` |
| `DeleteTaskCommand` | deep-copies the task, then `repo.delete` | re-`add`s the saved copy |
| `UpdateFieldCommand` | records old value, sets new | restores old value |
| `TransitionCommand` | records old status, transitions | restores old status + `completed_at` |
| `MacroCommand` | runs N commands; rolls back on failure | undoes them in reverse |

### `DeleteTaskCommand` keeps a backup

It `copy.deepcopy`s the task **before** deleting (`commands.py:56`) so undo can
restore the full object — tags, dependencies, timestamps and all — not just the
id.

### `UpdateFieldCommand` is generic

One class handles any single editable field: it stashes
`getattr(task, field)` before `setattr`ting the new value, and restores it on
undo. The service whitelists which fields are editable this way
(`title`, `description`, `priority`, `due`, `recurrence`).

### `MacroCommand` — atomic multi-step

Runs several commands as one unit. If any step raises, it **rolls back the
already-applied steps in reverse order** before re-raising (`commands.py:123`),
so a macro is all-or-nothing. `TodoService.bulk_complete(*ids)` uses this to
complete many tasks atomically.

## `CommandInvoker` — the undo/redo engine

Holds two stacks:

```
run(cmd):   execute → push onto UNDO stack → clear REDO stack
undo():     pop UNDO → cmd.undo() → push onto REDO stack
redo():     pop REDO → cmd.execute() → push onto UNDO stack
```

- **`run`** executes and records. It clears the redo stack — once you make a new
  change, the old "future" is gone (standard editor semantics).
- **`undo`** on an empty stack raises `NothingToUndoError`; **`redo`** likewise
  raises `NothingToRedoError`.
- `history_limit` (default 100) caps the undo stack; the oldest entry is dropped
  when exceeded.
- Convenience: `can_undo`, `can_redo`, `history()` (names of pending-undo cmds).

## How the service uses it

`TodoService` owns one `CommandInvoker` and routes every mutation through it:

```python
self.invoker.run(AddTaskCommand(self.repo, task))      # add()
self.invoker.run(TransitionCommand(self.repo, id, st)) # set_status()
self.invoker.run(UpdateFieldCommand(...))              # update_field()
self.invoker.run(DeleteTaskCommand(self.repo, id))     # delete()
```

So `svc.undo()` / `svc.redo()` reverse/replay the last operation regardless of
what it was. They return the command's `name` (e.g. `"transition"`) for display.

## Worked example

```python
t = svc.add("draft")            # UNDO: [add]
svc.complete(t.id)              # UNDO: [add, transition]
svc.undo()                      # → "transition"; task back to TODO; REDO: [transition]
svc.redo()                      # → "transition"; task DONE again
svc.add("note")                # new change → REDO cleared
```

Next: [Events](08-events.md).
