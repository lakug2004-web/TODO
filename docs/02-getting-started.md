# 2. Getting Started

## Prerequisites

- **Python ≥ 3.14** (see `.python-version` and `pyproject.toml`)
- **[uv](https://docs.astral.sh/uv/)** for dependency + environment management

uv reads `pyproject.toml` and `uv.lock`, creates a virtualenv, and runs commands
inside it. You never activate the venv manually; prefix commands with `uv run`.

## Install

```bash
uv sync          # create the venv and install deps (incl. dev + streamlit)
```

## Run

```bash
uv run todoapp                      # the scripted demo (console script)
uv run python -m todoapp           # same demo, via module entrypoint
uv run streamlit run src/streamlit_app.py   # the web UI (http://localhost:8501)
```

## Test

```bash
uv run pytest -q                   # full suite
uv run pytest tests/test_todoapp.py -q
uv run pytest -k persistence       # match by name
```

See [Testing](13-testing.md) for what's covered.

## Project layout

```
TODO/
├── pyproject.toml        # project metadata, deps, console script
├── uv.lock               # pinned dependency graph
├── .python-version       # interpreter pin (3.14)
├── README.md             # short top-level readme
├── tasks.json            # JSON store created by the Streamlit UI (gitignored-ish)
├── docs/                 # ← you are here
├── src/
│   ├── streamlit_app.py  # web UI (imports the package next to it)
│   └── todoapp/          # the engine package
│       ├── __init__.py        # public API re-exports + main()
│       ├── __main__.py        # enables `python -m todoapp`
│       ├── enums.py           # Priority, Status, RecurrenceUnit, transitions
│       ├── exceptions.py      # TodoError hierarchy
│       ├── models.py          # Tag, RecurrenceRule, Task
│       ├── repository.py      # TaskRepository port + InMemory adapter
│       ├── persistence.py     # FileTaskRepository (JSON) + serde
│       ├── specifications.py  # composable query predicates
│       ├── sorting.py         # named sort strategies
│       ├── commands.py        # Command pattern + CommandInvoker
│       ├── events.py          # EventBus (observer)
│       ├── analytics.py       # Analyzer → Stats
│       ├── service.py         # TodoService facade + DAG
│       └── cli.py             # scripted demo
└── tests/
    ├── test_todoapp.py        # engine behaviour
    └── test_persistence.py    # JSON round-trip
```

## src-layout note

This project uses the **src layout**: the importable package lives in
`src/todoapp/`, not at the repo root. `pyproject.toml` is configured (via
`uv_build`) so `import todoapp` resolves after `uv sync`. The Streamlit file
sits in `src/` alongside the package and prepends its own directory to
`sys.path` so it can `import todoapp` whether or not the package is installed —
see [Streamlit UI](12-streamlit-ui.md).

Next: [Architecture](03-architecture.md).
