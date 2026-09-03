# Habit Hooks (Layer 3)

**Layer:** Hard enforcement · Claude Code Stop hook · exit 2
**Product:** [github.com/habit-hooks/habit-hooks](https://github.com/habit-hooks/habit-hooks) · MIT · on PyPI
**Related ADR:** [ADR-001](../adr/001-adopt-habit-hooks.md) — why we adopted the product instead of a custom script

## What it does

After every agent turn, `habit-hooks run` executes as a Stop hook. It runs your
existing linter, filters for structural design violations (complexity, argument bloat,
etc.), maps each rule code to a tool-agnostic "smell" key, and attaches a
community-maintained coaching guide.

If violations exist, the hook exits `2` and Claude Code re-injects the coaching
report into the agent's context. The turn cannot close until the violations are
fixed — the agent must apply the guides, not just silence the rule.

## Why a product, not a script

We built a custom `scripts/habit_hook.py` first (Plan 50) and immediately discovered
the habit-hooks product does exactly the same thing with three structural advantages:

1. **Community-maintained coaching guides** — no need to write and update the mappings
   ourselves.
2. **Multi-linter normalization** — `PLR0913`, `max-params`, `max-arguments` all
   normalize to `too-many-parameters`, useful when the stack grows (TypeScript in the
   frontend, etc).
3. **Plugin architecture** — sensor and mapper are decoupled and independently testable.

See [ADR-001](../adr/001-adopt-habit-hooks.md) for the full reasoning.

## Configuration

### Install

```bash
uv add --dev habit-hooks
habit-hooks init                # one-time; writes .habit-hooks/config.toml
```

Commit `.habit-hooks/config.toml`.

### Linter thresholds

habit-hooks uses the linter you already have — you configure the structural rules in
your linter's config. In this repo (`pyproject.toml`):

```toml
[tool.ruff.lint]
# Use SPECIFIC codes — the broad "PLR" category enables PLR2004 (magic values)
# which generated 195 false positives on our first run.
select = ["E", "F", "I", "UP", "B", "SIM",
          "C901",    # McCabe cyclomatic complexity
          "PLR0911", # Too many return statements
          "PLR0912", # Too many branches
          "PLR0913", # Too many arguments
          "PLR0915"] # Too many statements

[tool.ruff.lint.mccabe]
max-complexity = 10

[tool.ruff.lint.pylint]
max-args = 8       # bumped from 7 to accommodate this repo's DI patterns
max-branches = 12
max-returns = 6
max-statements = 50
```

### Wire the Stop hook

`.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "habit-hooks run",
        "timeout": 30
      }]
    }]
  }
}
```

### Task runner

`Makefile`:

```makefile
check-design: ## Design quality sensor + agent coaching guides
	habit-hooks run
```

Added as the last step of `make check` so the full quality gate runs it too.

## Pre-existing violations

The baseline codebase had 17 structural violations, most of them pre-existing complex
functions. We use `# noqa: C901,PLR0915` inline comments to suppress them, always
with a note explaining why the complexity is inherent:

```python
async def callback(...):  # noqa: C901,PLR0912,PLR0915
    """OAuth callback — must handle state validation, code exchange,
    session creation, and error redirection in a single request cycle."""
```

This documents known tech debt without hiding the rule. If a suppressed function
grows worse, adding a new violation type will still trigger the hook — only the
suppressed codes are silenced.

## What it does NOT do

- Pure unit testing (no HTTP, no fixtures — habit-hooks doesn't measure test quality)
- Mutation testing (that's Layer 7 of the [testing pyramid](testing-pyramid.md))
- Style rules already covered by ruff/format (formatting is Layer 1, not this)
- Security scanning (that's pip-audit in [ci-integration.md](ci-integration.md))

## Verification

```bash
habit-hooks run
echo "exit: $?"    # expect 0 on a clean codebase, 2 with violations to fix
```

## Known limitations

- The 30-second timeout may be tight for very large codebases; raise if needed.
- The coaching guides are in English and generic. For repo-specific advice (e.g.
  "use Pydantic AI for this pattern"), extend with a plugin or annotate the smell
  inline.
- `# noqa` comments have to fit within the 100-char line limit — long explanations
  don't work in the comment itself. Use a comment on the line above for context.
