# Anatomy — every file the harness touches

Grouped by component. Nothing here is decorative — each file has one job.

## ◆ Planning Gate — 4 files

| File | Role |
|---|---|
| `.claude/hooks/require-plan.sh` | PreToolUse hook. Reads tool input JSON, checks plan-ref + git status, exits 2 if no plan found. |
| `.claude/settings.json` | Registers PreToolUse (Write → require-plan.sh) and Stop (habit-hooks run) hooks. |
| `.claude/plan-ref` *(gitignored)* | Session state — agent writes the current plan number here to unlock the gate. Never committed. |
| `CLAUDE.md § Planning discipline` | 4-step planning rule: exec plan → optional proposal → human approval → write plan-ref. |

## ◆ ADR Discipline — 4 files

| File | Role |
|---|---|
| `docs/adr/README.md` | ADR index with criteria for when to write one and the full lifecycle (proposed → accepted → superseded). |
| `docs/adr/TEMPLATE.md` | Nygard-style template: Context / Decision / Consequences (positive, negative, neutral). |
| `docs/adr/001-adopt-habit-hooks.md` | First real ADR: why we adopted the habit-hooks product instead of maintaining a custom script. |
| `CLAUDE.md § ADR Discipline` | When to write an ADR, lifecycle rules, explicit note that the gate does not block on missing ADRs. |

## ◆ Habit Hooks — 5 files

| File | Role |
|---|---|
| `pyproject.toml [tool.ruff.lint]` | Adds C901, PLR0911–PLR0915 to select. Configures mccabe + pylint thresholds. Uses specific codes (not "PLR") to avoid PLR2004 false positives. |
| `Makefile — check-design target` | `make check-design` runs `habit-hooks run`. Added as last step of `make check`. |
| `CLAUDE.md § Agent limits` | "DO NOT mark a task complete until make check-design passes." |
| `email_triage/routers/{auth,inbox,triage}.py` | `# noqa: C901,PLR09xx` on pre-existing complex functions — documents known tech debt without hiding the rule. |
| `evals/run_evals.py` | `# noqa: C901,PLR0915` on eval orchestrator and report formatter — pre-existing complexity. |

## ◆ Bootstrap prompt — 1 file

| File | Role |
|---|---|
| `docs/prompts/bootstrap-agentic-harness.md` | Self-contained Claude Code prompt to install the full harness in any other repository. Adapts to the target repo's stack (Python/Node/Rails). Recommends habit-hooks, not a custom script. |

## Totals

- **14 files** across 4 groups
- **7 new**, **7 modified**
- **1 gitignored** (`.claude/plan-ref`)
- **1 downloadable** (habit-hooks, installed via `uv add --dev`)
