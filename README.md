# ci-shared

Shared GitHub Actions workflows for Rick's Python sync/monitor repos. This repo
is the **single source of truth** for CI across:

- [`sync-clients-gcal-airtable`](https://github.com/rickarm/sync-clients-gcal-airtable)
- [`sync-peloton-airtable`](https://github.com/rickarm/sync-peloton-airtable)
- [`sleep-airtable`](https://github.com/rickarm/sleep-airtable)
- [`system-monitor`](https://github.com/rickarm/system-monitor)
- [`craft-sort-unsorted`](https://github.com/rickarm/craft-sort-unsorted)
- [`alfred`](https://github.com/rickarm/alfred)

If you change CI behaviour, you change it **here**, once. Every repo above calls
this workflow and picks up the change on its next run.

## What the CI does and why

These repos are unattended cron/launchd jobs and watchers. The failure mode that
hurts isn't a failing test (most have none) — it's a **silent regression**: a
renamed constant, a bad import, a syntax slip that only blows up at 3am when the
job actually runs.

So CI is a cheap **regression tripwire**, three gates:

| Gate | What it catches | Tool |
|---|---|---|
| **Lint** | unused imports/vars, ambiguous names, dead `f""` strings, style | `ruff check .` |
| **Compile smoke** | syntax errors and renamed/missing imports **even with zero tests** | `python -m compileall` over every tracked `.py` |
| **Tests** | logic regressions — **only runs if a suite exists** | `pytest -q` |

The compile-smoke gate is the important one: it gives every repo regression
coverage on the import surface without anyone writing a single test.

### Auto-adapting install

| Repo has… | Install command used |
|---|---|
| `uv.lock` | `uv sync --frozen` |
| `pyproject.toml` | `uv sync` |
| `requirements.txt` | `uv venv && uv pip install -r requirements.txt` |
| none of the above | `uv venv` (lint + compile still run) |

Then `ruff` and `pytest` are installed on top regardless, so the gates always run.

## How it's wired

    rickarm/ci-shared
    └── .github/workflows/python-ci.yml      ← the real steps (on: workflow_call)

    rickarm/<each-repo>
    └── .github/workflows/ci.yml             ← thin caller, runs on PR + push to main
            jobs:
              ci:
                uses: rickarm/ci-shared/.github/workflows/python-ci.yml@main

The caller fires on every pull request and every push to `main`. It delegates
the actual work to the reusable workflow here, pinned to `@main`.

## Adopting this in a NEW repo

Drop this file in at `.github/workflows/ci.yml`:

    name: CI
    on:
      pull_request:
      push:
        branches: [main]
    jobs:
      ci:
        uses: rickarm/ci-shared/.github/workflows/python-ci.yml@main

Then make sure this repo's access setting allows it (see below), and open a PR.

### Optional caller inputs

Most repos need nothing beyond the snippet above. Two inputs exist for repos that
don't fit the zero-config mold (e.g. `alfred`, a service rather than a script):

| Input | Type | Default | When you need it |
|---|---|---|---|
| `python-version` | string | `""` (uv default) | Pin a specific interpreter, e.g. `"3.12"`. |
| `all-extras` | boolean | `false` | Test deps live behind a `pyproject` **extra** (e.g. `[project.optional-dependencies] test`) instead of the default dependency group. Adds `--all-extras` to `uv sync` so `pytest-asyncio` & friends actually install. |
| `test-env` | string | `""` | The package can't even be imported without an env var (e.g. a `pydantic-settings` field with no default that's read at module load). Pass **throwaway** `KEY=VALUE` lines — exported before lint/compile/test. Never put a real secret here; for real secrets use `secrets:` (see "Adding tests later"). |

`alfred`'s caller, for reference:

    jobs:
      ci:
        uses: rickarm/ci-shared/.github/workflows/python-ci.yml@main
        with:
          all-extras: true
          test-env: |
            THINGS_AGENT_API_KEY=ci-test-key

## ⚙️ Required one-time setting: private cross-repo access

All these repos are **private**. A private repo's reusable workflow can only be
called by sibling repos if `ci-shared` explicitly allows it. Set it once:

**`ci-shared` → Settings → Actions → General → Access →
"Accessible from repositories owned by the user rickarm"** → Save.

Without this, caller runs fail with `workflow ... is not allowed to be called`.

Equivalent via API:

    gh api -X PUT repos/rickarm/ci-shared/actions/permissions/access -f access_level=user

## Adding tests later

Nothing to configure — the workflow auto-discovers `test_*.py`, `*_test.py`, and
`tests/` directories and runs `pytest`. If a future suite needs secrets, declare
them under `secrets:` in `python-ci.yml`'s `workflow_call` block and forward them
from the caller; use throwaway values and mock real API calls. No repo's suite
needs secrets today (system-monitor's 48 tests pass with none).

## Updating CI for everyone

Edit `.github/workflows/python-ci.yml` here and merge. Callers pin `@main`, so
every repo uses the new definition on its next run. For change control, tag a
release (e.g. `v1`) and switch callers to `@v1`.

## Making CI a merge gate (branch protection)

By default these checks are **advisory**. To require the `test` check before
merge, add a branch protection rule on `main` in EACH repo:

UI: repo → Settings → Branches → Add rule → pattern `main` → "Require status
checks to pass before merging" → select **`test`** → Save.

CLI, per repo:

    for repo in sync-clients-gcal-airtable sync-peloton-airtable sleep-airtable system-monitor craft-sort-unsorted alfred; do
      gh api -X PUT "repos/rickarm/$repo/branches/main/protection" \
        -H "Accept: application/vnd.github+json" \
        -f 'required_status_checks[strict]=true' \
        -f 'required_status_checks[checks][][context]=test' \
        -F 'enforce_admins=false' \
        -F 'required_pull_request_reviews=null' \
        -F 'restrictions=null'
    done

The check is named `test` (the job id in the reusable workflow).
