# Commands — make targets and manual ops

| Command | Description | Layer |
|---|---|---|
| `make check-design` | Runs `habit-hooks run`. Design quality sensor: exits 0 if clean, 2 with coaching report if violations found. | Habit Hooks |
| `make check` | Full gate: `lint → format → typecheck → tests → check-design`. All must pass. | All |
| `make lint` | `uv run ruff check --fix` — includes the C901/PLR09xx structural rules. | Habit Hooks |
| `make test` | Full pytest suite (260 tests, ~26s). | Testing Pyramid L2+L3 |
| `make test-unit` | `pytest -m unit` (154 tests, ~7s) — mirrors CI. | Testing Pyramid L2 |
| `make test-e2e` | `pytest -m e2e` (106 tests, ~19s) — mirrors CI. | Testing Pyramid L3 |
| `make precommit` | `uv run pre-commit run --all-files` — ruff + pyright. Runs before every commit. | Pre-commit |
| `make audit` | `pip-audit` for CVEs in prod dependencies. Mirrors CI. | Security |
| `make sbom` | Generates a CycloneDX SBOM. Mirrors CI. | Security |
| `habit-hooks init` | One-time setup. Auto-detects stack, writes `.habit-hooks/config.toml`. | Habit Hooks |
| `habit-hooks run` | Manual run of the design sensor. Same as `make check-design`. | Habit Hooks |
| `echo "51" > .claude/plan-ref` | Unlock the planning gate for plan 51. Written by the agent after human approval. | Planning Gate |

## Verification commands

Used during setup or debugging.

```bash
# 1. Planning gate BLOCKS (exit 2) when no plan-ref exists
echo '{"tool_name":"Write","tool_input":{"file_path":"email_triage/x.py","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 2

# 2. Planning gate ALLOWS (exit 0) for paths outside gated dirs
echo '{"tool_name":"Write","tool_input":{"file_path":"docs/plan.md","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 0

# 3. Habit Hooks — clean codebase → exit 0
habit-hooks run
echo "exit: $?"   # expect 0 (or 2 with violations to fix)

# 4. Context7 MCP — verify it starts (in Claude Code session)
# Type "use context7" or call resolve-library-id("react")
```
