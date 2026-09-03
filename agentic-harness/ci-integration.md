# CI integration — how the harness maps to the pipeline

## Pre-commit (local, every commit)

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.15
    hooks:
      - id: ruff-check    # includes C901, PLR09xx structural rules
        args: [--fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: pyright
        entry: uv run pyright
        language: system
```

**Habit Hooks is NOT in pre-commit.** `habit-hooks run` is a Stop hook (Claude Code)
and a `make check-design` target — not a pre-commit hook. Pre-commit runs on every
commit; the design sensor runs on every agent turn. Different frequencies, different
audiences (human vs. agent).

## GitHub Actions

### Tests (unit + e2e matrix)

`.github/workflows/tests.yml`:

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: uv run pytest -m unit

  e2e:
    needs: unit
    runs-on: ubuntu-latest
    steps:
      - run: uv run pytest -m e2e
```

154 unit tests (~7s) and 106 e2e tests (~19s), auto-marked by fixture in
`tests/conftest.py`.

### Security

`.github/workflows/security.yml`:

```yaml
jobs:
  audit:
    steps:
      - run: uv export --frozen --no-dev --format requirements-txt -o req.txt
      - run: uvx pip-audit -r req.txt --strict

  sbom:
    steps:
      - run: uvx cyclonedx-py requirements req.txt -o sbom.json
      - uses: actions/upload-artifact@v4
        with: { name: sbom-cyclonedx, path: sbom.json }
```

Runs on push, PR, and weekly (Mondays 06:00 UTC).

### Design sensor — NOT in CI (yet)

`habit-hooks run` is an agent-time gate, not a PR merge gate. Roadmap phase H3 adds
it as an informative (non-blocking) CI job per PR. Informative first, then promoted
to blocking after calibration.

Rationale: the sensor's exit-2 behavior is designed to re-inject into the agent's
context. In CI it would just be a plain fail — same signal you get from ruff, minus
the coaching guides. Adding it to CI is worthwhile as an accountability check
(catches design regressions in human commits), but not urgent.

### Frontend CI

`.github/workflows/frontend-ci.yml` — separate pipeline, path-filtered to
`frontend/**`. See [`docs/exec-plans/49-frontend-ci-testing.md`](../exec-plans/49-frontend-ci-testing.md).

### AI code review

`.github/workflows/code-review.yml` — `anthropics/claude-code-action@v1` invoking
the `.claude/skills/review-pr/` skill on every PR. Advisory only, not a required
check. See [`docs/exec-plans/47-ai-code-review-ci.md`](../exec-plans/47-ai-code-review-ci.md).
