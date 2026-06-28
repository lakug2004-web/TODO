# 9. Analytics

File: `src/todoapp/analytics.py`

Computes reporting snapshots from a collection of tasks. Stateless and pure —
hand it any iterable of tasks, get a `Stats` back.

## `Stats` — the snapshot

```python
@dataclass(frozen=True, slots=True)
class Stats:
    total: int
    by_status: dict[str, int]      # every Status name → count (incl. zeros)
    by_priority: dict[str, int]    # every Priority label → count (incl. zeros)
    overdue: int
    completion_rate: float         # done / total, 0.0 if empty
    top_tags: list[tuple[str, int]]  # most common tags, name → count
```

`Stats.as_lines()` renders a tidy text report (used by the CLI demo and
printable anywhere).

## `Analyzer`

```python
Analyzer(top_n_tags=5).compute(tasks) -> Stats
```

`compute` walks the tasks once using `collections.Counter`:

- counts by status and by priority (then fills in every enum member so absent
  statuses/priorities show as `0` — stable shape for charts/tables),
- counts overdue via `task.is_overdue`,
- computes completion rate (guards divide-by-zero on an empty list),
- tallies tags and takes the top N.

### `workload_by_tag` (static)

```python
Analyzer.workload_by_tag(tasks) -> dict[str, int]
```

Counts tags **only across active tasks** (`status.is_active`), ordered most-
common first. Answers "where is my remaining work concentrated?" — distinct from
`top_tags`, which counts all tasks regardless of status.

## Usage

```python
s = svc.stats()                      # service delegates to Analyzer.compute
print("\n".join(s.as_lines()))
print(f"{s.completion_rate:.0%} done, {s.overdue} overdue")
```

The Streamlit **Stats tab** renders `Stats` as metric cards plus bar charts of
`by_status` and `by_priority`, and lists `top_tags` — see
[Streamlit UI](12-streamlit-ui.md).

## Why it's separate

Keeping aggregation out of `Task` and the repository means reporting can evolve
(new metrics, different groupings) without touching the domain model or storage,
and the same analyzer works over any task list — the whole repo, a filtered
query result, or a test fixture.

Next: [Service Facade](10-service.md).
