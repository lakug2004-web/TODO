# 6. Specifications & Sorting

Files: `src/todoapp/specifications.py`, `src/todoapp/sorting.py`

## Specifications — composable filters

A **Specification** is a predicate over a `Task` that can be combined with
boolean operators. Instead of scattering `if`-conditions across the codebase,
queries become declarative, reusable objects.

### The base class

`Specification` (`specifications.py:19`) is abstract with one method,
`is_satisfied_by(task) -> bool`, and operator overloads:

| Operator | Builds | Meaning |
|----------|--------|---------|
| `a & b` | `AndSpec` | both must match |
| `a \| b` | `OrSpec` | either matches |
| `~a` | `NotSpec` | negation |

`__call__` aliases `is_satisfied_by`, so a spec is also a plain callable —
handy for `filter()` / list comprehensions.

### Leaf specifications

| Spec | Matches when |
|------|--------------|
| `ByStatus(*statuses)` | task status is one of the given |
| `ByPriority(minimum)` | `task.priority >= minimum` |
| `HasTag(tag)` | task has that tag |
| `HasAllTags(*tags)` | task has every given tag (subset test) |
| `IsOverdue()` | `task.is_overdue` |
| `DueBefore(when)` | has a due date on/before `when` |
| `TextMatches(needle)` | needle in title or description (case-insensitive) |
| `IsActive()` | `task.status.is_active` |
| `Always()` | everything (useful as a neutral default) |
| `Predicate(fn)` | wrap any `Callable[[Task], bool]` as a spec |

### Composing

```python
from todoapp import ByPriority, ByStatus, HasTag, IsOverdue, Priority, Status

# high+ priority AND still open
hot = ByPriority(Priority.HIGH) & ByStatus(Status.TODO, Status.IN_PROGRESS)

# overdue OR tagged #home
attention = IsOverdue() | HasTag("home")

# active but NOT tagged #someday
now = ByStatus(Status.TODO) & ~HasTag("someday")

results = svc.find(hot, sort="due")
```

`TodoService.find(spec, sort=..., reverse=...)` applies the spec then sorts. The
Streamlit UI builds a spec dynamically from the filter widgets by `&`-chaining
whichever filters the user set — see
[Streamlit UI](12-streamlit-ui.md).

### Why this pattern pays off

- **Reusable:** name a complex query once, use it everywhere.
- **Testable:** each leaf is trivially unit-tested in isolation.
- **Readable:** `ByPriority(HIGH) & ~HasTag("someday")` reads like the intent.
- **Extensible:** add a new filter by writing one small class — no edits to
  existing specs.

## Sorting — named strategies

`sorting.py` is the **Strategy pattern**: a registry of sort-key functions
selected by name.

| Strategy | Orders by |
|----------|-----------|
| `"priority"` | highest priority, then soonest due, then oldest (`_seq`) |
| `"due"` | soonest due first (no due = last), then priority |
| `"created"` | creation order (`_seq`) |
| `"title"` | alphabetical, case-insensitive |
| `"status"` | by status enum value, then priority |

`sort_tasks(tasks, strategy="priority", reverse=False)` looks up the key in
`STRATEGIES` and returns a new sorted list; an unknown strategy raises
`ValueError`.

Tasks with no due date sort *last* under due-based ordering via a `date.max`
sentinel — see `by_priority_desc`/`by_due_date` (`sorting.py:13`).

### Adding a strategy

```python
# in sorting.py
def by_age(task): return -task._seq
STRATEGIES["age"] = by_age
```

Then `svc.find(spec, sort="age")` just works.

Next: [Commands & Undo/Redo](07-commands-undo-redo.md).
