# SupplyTrace

A tool that helps find **software supply chain attacks** in a Git repository.

## The problem

When someone attacks a software project, they usually don't touch the actual
code. They quietly change **how the code gets built** — the CI workflow, or the
dependency lockfile. In `git log`, those commits look exactly like every other
commit.

## The idea

Git treats every file as plain text. It has no idea that
`.github/workflows/build.yml` controls how the software is built, while
`README.md` controls nothing.

So SupplyTrace labels every changed file by its **role**:

| File | Role |
|---|---|
| `src/app.py` | SOURCE |
| `.github/workflows/build.yml` | WORKFLOW — *how it's built* |
| `package.json` | DEPENDENCY_MANIFEST — *what it says it needs* |
| `package-lock.json` | LOCKFILE — *what it actually installs* |

Then it looks for **odd combinations** of those roles, for example:

- one commit changed a workflow **and** a dependency file
- a lockfile changed **without** the manifest changing
  (the installed packages changed, but nobody asked for a new package)

Commits are scored by how many rules fire, and printed highest first.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

## Try it

We ship a fake project with an attack hidden in 15 commits:

```bash
python examples/build_demo.py --target ~/paylite
```

Look at it with plain Git first — you won't spot anything:

```bash
git -C ~/paylite log --oneline
```

Now ask the tool where to look:

```bash
supplytrace investigate ~/paylite
```

It puts one commit on top with a score of **11**; the next scores **3**.
See everything about it, then check the actual change:

```bash
supplytrace commit ~/paylite d95c2685eaf3
git -C ~/paylite show d95c2685eaf3
```

The commit swapped a build action from a version tag to a **branch** (so its
code can change anytime), added a `curl ... | sh` step, and gave that step the
publishing secret.

## Commands

| Command | What it does |
|---|---|
| `supplytrace analyze <repo>` | Overview: commits, authors, branches, changed files |
| `supplytrace investigate <repo>` | Ranks commits by what's worth reading first |
| `supplytrace commit <repo> <sha>` | Everything about one commit |

All three take `--json` if you want machine-readable output.

## How it works

1. Runs `git log` through a locked-down wrapper (a repo's own config can make
   Git execute programs, so those settings are switched off, and it never runs
   anything from the repo)
2. Parses the output into Python objects
3. Labels every changed file by its role
4. Applies rules to those labels and scores each commit

## Status

Working: reading Git history, labelling files, scoring commits, the CLI.

Not built yet: reading *inside* files. It knows `release.yml` changed, not
what's in it. Coming next — diff/blame line tracking, parsing workflow YAML,
checking if actions are pinned safely, dependency graphs, and a web UI.

## Where things are

```
supplytrace/
├── cli.py                  the 3 commands + all the printing
├── core/
│   ├── git_command.py      the ONLY place git is ever run
│   ├── config.py           settings (limits, timeouts)
│   ├── errors.py           our exception types
│   └── logging.py          logging, hides tokens
├── models/
│   ├── file.py             file labels + the rules that assign them
│   ├── commit.py           what a commit looks like
│   ├── repository.py       the overall result object
│   ├── signal.py           the 7 rules + their point values
│   └── evidence.py         where each fact came from
└── analyzers/
    ├── git_analyzer.py     runs git, parses the output
    └── signal_analyzer.py  checks the rules, scores commits

examples/build_demo.py      builds the fake project
tests/                      218 tests
```

**Reading order if you're new:** `cli.py` (what the user sees) →
`analyzers/signal_analyzer.py` (the rules) → `models/file.py` (the labels).

## Adding a new rule

Say you want to flag commits that delete tests. Three small edits:

1. **`models/signal.py`** — add the name, a weight, and a description:
   ```python
   class Signal(str, Enum):
       TESTS_DELETED = "TESTS_DELETED"

   SIGNAL_WEIGHTS = {Signal.TESTS_DELETED: 2, ...}
   SIGNAL_DESCRIPTIONS = {Signal.TESTS_DELETED: "deleted test files", ...}
   ```

2. **`analyzers/signal_analyzer.py`** — inside `commit_signals()`, check for it:
   ```python
   deleted_tests = [c.path for c in commit.file_changes
                    if c.category is FileCategory.TEST
                    and c.status is ChangeStatus.DELETED]
   if deleted_tests:
       add(Signal.TESTS_DELETED, deleted_tests)
   ```

3. **`tests/test_signals.py`** — add a test for it.

That's it. The CLI picks it up automatically.

To add a new **file label** instead, edit `classify_file()` in `models/file.py`.

## Tests

```bash
pytest -q
```

Please run this before pushing — the tests also check that the demo still
works, so if you break the scoring you'll know straight away.
