# Prompt: Bootstrap Agentic Quality Harness

Pega este prompt en una sesión de Claude Code dentro del repositorio destino.
El agente explora, adapta e instala — el humano revisa y commitea.

---

## PROMPT (copiar desde aquí)

You are setting up an **agentic quality harness** in this repository. The harness has five components that map to the agent's work cycle:

| # | Component | When | Enforcement |
|---|---|---|---|
| 1 | **Planning Gate** | Before writing code | Hard — PreToolUse hook, exit 2 |
| 2 | **ADR Discipline** | When making arch decisions | Soft — CLAUDE.md instruction |
| 3 | **Context7 MCP** | When using library APIs | Soft — available tool |
| 4 | **Habit Hooks** | After every agent turn | Hard — Stop hook, exit 2 |
| 5 | **Before-work reading protocol** | Before starting any task | Soft — CLAUDE.md instruction |

Exit code `2` is the only code that hard-blocks a tool call or prevents an agent turn from completing — CLAUDE.md instructions are soft context, not enforcement. Never build a custom linter-to-guide script for Habit Hooks; the product already exists and is community-maintained.

---

### Step 0 — Explore before touching anything

Before creating any file, read and understand this repository. Do this now, in order:

1. Read `CLAUDE.md` (or equivalent: `AGENTS.md`, `.clinerules`) if it exists.
2. Check if `docs/adr/` exists. If yes, read `docs/adr/README.md` to understand prior decisions.
3. Check if a `docs/exec-plans/` or `docs/plans/` directory exists for planning convention.
4. Run `ls` at the repo root. Identify: main source dir, test dir, CI config dir.
5. Check if `.claude/settings.json` and `.mcp.json` already exist — read both, never overwrite.
6. Identify language + linter: Python+ruff, JavaScript+ESLint, TypeScript+Biome, Go+staticcheck, etc.
7. Run `git log --oneline -5` to understand commit style and naming.

Report what you found in one paragraph before proceeding.

---

### Step 1 — Planning Gate (PreToolUse hook, exit 2)

#### What it does
Fires before every `Write` tool call. If the destination is application code, checks:
- (a) `.claude/plan-ref` exists with a plan identifier, OR
- (b) `git status` shows a new/modified planning doc this session.

If neither passes → exit `2` (hard block, tool call cancelled).

#### Create `.claude/hooks/require-plan.sh`

```bash
#!/usr/bin/env bash
# PreToolUse hook — planning gate.
# Exit 0 = allow.  Exit 2 = HARD BLOCK (cancels the tool call).

set -uo pipefail

INPUT=$(cat)

FILE_PATH=$(printf '%s' "$INPUT" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    print(d.get('tool_input', {}).get('file_path', ''))
except Exception:
    print('')
" 2>/dev/null || echo "")

# ── ADAPT: replace with this repo's actual source directories ─────────────────
if ! printf '%s' "$FILE_PATH" | grep -qE "^(src|lib|app|tests|__SOURCE_DIR__|\.github/workflows)/"; then
    exit 0
fi

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"

# Gate 1: session plan reference
if [ -f "$REPO_ROOT/.claude/plan-ref" ] && [ -s "$REPO_ROOT/.claude/plan-ref" ]; then
    exit 0
fi

# Gate 2: plan document created/modified this session (uncommitted)
# ── ADAPT: replace docs/ with the planning directory convention of this repo ───
PLAN_STATUS=$(git -C "$REPO_ROOT" status --short "docs/" 2>/dev/null \
  | grep -E "\.(md|txt|rst|adoc)$" | grep -v "^$" || true)
if [ -n "$PLAN_STATUS" ]; then
    exit 0
fi

# ── ADAPT: update path names in the message to match this repo ────────────────
cat >&2 <<EOF
╔══════════════════════════════════════════════════════════════════╗
║  PLANNING GATE — write blocked                                   ║
╚══════════════════════════════════════════════════════════════════╝

Attempted to write: $FILE_PATH

Before modifying application code:

  1. Create a planning doc (exec-plan, ADR, spec — see this repo's convention)
  2. If the task adopts/deprecates a tool or introduces a cross-cutting pattern,
     also create docs/adr/NNN-name.md (see docs/adr/TEMPLATE.md)
  3. Present the plan to the human and wait for acknowledgment
  4. Unlock: echo "plan-id" > .claude/plan-ref

Auto-unlock when:
  - .claude/plan-ref is non-empty, OR
  - git status shows a new/modified doc in docs/

EOF
exit 2
```

```bash
chmod +x .claude/hooks/require-plan.sh
```

**Adapt the regex:** after Step 0, replace `__SOURCE_DIR__` with the actual source directories.
Examples: Python → `^(mypackage|tests|scripts)/` · Node monorepo → `^(packages|apps|src)/` · Rails → `^(app|spec|config)/`

#### Add to `.gitignore`

```
# Planning gate session state (never commit)
.claude/plan-ref
```

---

### Step 2 — ADR Discipline (instruction-enforced)

#### Create `docs/adr/README.md`

```markdown
# Architecture Decision Records

Permanent records of *why* significant decisions were made — alongside the code.
ADRs are never deleted; they are deprecated or superseded, never removed.

## When to write an ADR
- Adopting or deprecating a tool, library, or framework
- Introducing a cross-cutting architectural pattern
- Explicitly rejecting an alternative that seems obvious to a future reader
- Accepting known negative consequences (tech debt, coupling, performance)

## Status values
`Proposed` · `Accepted` · `Deprecated` · `Superseded by [ADR-NNN](NNN-name.md)`

## Index

| # | ADR | Status | Date | Area |
|---|---|---|---|---|
```

#### Create `docs/adr/TEMPLATE.md`

```markdown
# ADR-NNN: Title (active voice, ≤ 10 words)

**Status:** Proposed | Accepted | Deprecated | Superseded by [ADR-NNN](NNN-name.md)
**Date:** YYYY-MM-DD
**Related:** [Exec Plan NN](../exec-plans/NN-name.md)

## Context
What forces are at play. The problem forcing this decision.
Do NOT include the decision here — only the context that makes it necessary.

## Decision
Active voice. "We adopt X." / "We use Y instead of Z." One or two sentences.

## Consequences

### Positive
### Negative / Accepted trade-offs
### Neutral
```

---

### Step 3 — Context7 MCP (up-to-date library documentation)

Context7 (`@upstash/context7-mcp`) is an MCP server that injects current, version-specific
library documentation into the agent's context. It prevents hallucinated APIs and deprecated
patterns — critical for fast-moving libraries (pydantic-ai, SQLAlchemy async, FastAPI, etc.).

#### Create `.mcp.json` (project-level MCP config, committed to repo)

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"],
      "type": "stdio"
    }
  }
}
```

#### Auto-approve in `.claude/settings.json`

Add `"enabledMcpjsonServers": ["context7"]` to `.claude/settings.json` so the server
starts without a per-session confirmation prompt.

**Requirements:** Node.js must be installed. The server starts lazily (only when a tool
is called) and runs locally — no API key required.

**When to use Context7:** whenever writing code that depends on a library's specific API
(not stdlib, not business logic). Call `resolve-library-id` then `query-docs`.

---

### Step 4 — Habit Hooks (Stop hook, exit 2)

> **Use the product — do NOT write a custom script.**
> [habit-hooks/habit-hooks](https://github.com/habit-hooks/habit-hooks) · MIT · PyPI
>
> Runs your existing linter, normalizes violations into tool-agnostic smell keys
> (`too-many-parameters`, `high-complexity`…), and attaches community-maintained
> coaching guides tuned for AI agents. Exit 2 re-injects the report until fixed.

#### Install

```bash
# Python repos
uv add --dev habit-hooks     # or: pip install habit-hooks

# Node repos
npm install --save-dev habit-hooks

# macOS (any stack, all plugins)
brew install habit-hooks/tap/habit-hooks
```

One-time setup (auto-detects stack, writes `.habit-hooks/config.toml`):

```bash
habit-hooks init
```

Commit `.habit-hooks/config.toml`.

#### Configure linter thresholds

Habit Hooks uses your existing linter. Add structural complexity rules to it:

**Python — `pyproject.toml`:**
```toml
[tool.ruff.lint]
# Use specific codes — "PLR" broad category enables PLR2004 (magic values, 195 false positives)
select = ["E", "F", "I", "C901", "PLR0911", "PLR0912", "PLR0913", "PLR0915"]

[tool.ruff.lint.mccabe]
max-complexity = 10

[tool.ruff.lint.pylint]
max-args = 7       # raise to 8 if repo uses heavy dependency-injection patterns
max-branches = 12
max-returns = 6
max-statements = 50
```

**JavaScript/TypeScript — `eslint.config.js`:**
```js
rules: {
  "complexity":             ["warn", 10],
  "max-depth":              ["warn", 4],
  "max-params":             ["warn", 4],
  "max-lines-per-function": ["warn", { max: 50 }],
}
```

#### Add `check-design` to the task runner

**Makefile:**
```makefile
check-design: ## Design quality sensor + agent coaching guides
	habit-hooks run
```

**package.json:**
```json
"scripts": { "check-design": "habit-hooks run" }
```

---

### Step 5 — Wire hooks and MCP in `.claude/settings.json`

The final `.claude/settings.json` with all hooks and the Context7 auto-approval:

```json
{
  "enabledMcpjsonServers": ["context7"],
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"${CLAUDE_PROJECT_DIR}/.claude/hooks/require-plan.sh\"",
            "timeout": 10
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "habit-hooks run",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

If `.claude/settings.json` already exists, merge — never replace existing keys.

---

### Step 6 — CLAUDE.md (full template)

Add these sections to the project's main instructions file (`CLAUDE.md`, `AGENTS.md`,
`.clinerules`, etc.). Place them before any "agent limits" section.

```markdown
## Before starting any task — reading protocol

Execute before touching any file:

1. Read `AGENTS.md` (or equivalent) — code map, agent/human roles, dev cycle.
2. Read `docs/adr/README.md` — scan for ADRs relevant to the module you'll touch.
3. Read `docs/exec-plans/README.md` (or equivalent) — identify any 🚧 in-progress
   plan that touches the same area.

Skipping this produces context-free changes that repeat decisions already made.

```bash
# Quick search
grep -r "module-name" docs/adr/README.md docs/exec-plans/README.md
```

## Planning discipline

Applies to tasks that touch ≥3 files, add a dependency, change an architectural
pattern, or add a new public interface.

**Before writing a single line of application code:**

1. Create a planning doc (exec-plan, ADR, spec — whatever this repo's convention is).
   Include: intent, scope, concrete file changes, design decisions, done-when checklist.
2. Present the plan to the human and wait for explicit acknowledgment.
3. Write the plan identifier into `.claude/plan-ref`:
   `echo "plan-id" > .claude/plan-ref`

For trivial fixes (1-2 files, no new deps, no architectural impact) planning docs are
optional — but state the scope explicitly in your response.

The PreToolUse hook in `.claude/settings.json` enforces this deterministically.

## Architecture Decision Records (ADRs)

ADRs live in `docs/adr/`. See `docs/adr/README.md` for the index.

Write an ADR when the task: adopts/deprecates a tool · introduces a cross-cutting
pattern · rejects a seemingly obvious alternative · accepts known trade-offs.

ADRs are never deleted. If a decision is reversed, create a new ADR with status
`Superseded by ADR-NNN` and update the old one's status line.

## Design quality

After every turn, the Stop hook runs `habit-hooks run`. If it reports violations,
apply the refactoring guide for each one before considering the task complete.

Do NOT suppress with `# noqa` / `// eslint-disable` unless the smell is genuinely
intentional — if so, add a comment explaining the invariant.

## Agent tooling available in this repo

### Context7 MCP
Provides up-to-date, version-specific library docs. Use when writing code that
depends on a library's specific API (not stdlib, not business logic):

```
use context7   # in any prompt, or call resolve-library-id + query-docs directly
```

### Archify (optional — install separately)
Generates verified architecture diagrams as interactive HTML from typed JSON.
Install: `npx skills add tt-a1i/archify -g`. Use for ADRs with complex topology,
sequence diagrams for multi-step flows, and system architecture maps.
```

---

### Step 7 — Verify the installation

Run each check and confirm the expected exit code:

```bash
# 1. Planning gate: BLOCK (exit 2) — no plan exists yet
echo '{"tool_name":"Write","tool_input":{"file_path":"src/new_feature.py","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 2

# 2. Planning gate: ALLOW (exit 0) — writing to docs is fine
echo '{"tool_name":"Write","tool_input":{"file_path":"docs/plan.md","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 0

# 3. Habit Hooks: clean codebase → exit 0
habit-hooks run
echo "exit: $?"   # expect 0 (or 2 with violations — fix them first)

# 4. Context7 MCP: verify it starts
# In a Claude Code session: type "use context7" or call resolve-library-id("react")
# Expected: tool executes without error, returns a context7 library ID
```

**Troubleshooting:**
- Gate returns 0 when it should block → regex in `require-plan.sh` doesn't match the source dirs (Step 0 findings)
- `habit-hooks run` finds no violations on complex code → check `.habit-hooks/config.toml` includes the linter and thresholds are set in the linter config
- Context7 fails → check `node --version` (requires Node 20+) and network access

---

### Step 8 — Optional extras

These are not required for the core harness but are worth considering:

**TestSprite** (`testsprite.com`) — AI-generated E2E/API regression tests. Generates
10–50 tests from the codebase in ~10 min. Requires: account + credits + running server.
Install via MCP or CLI. Best used after exec plans close (new endpoints or UI flows).

```bash
# Check existing config before bootstrapping
ls testsprite_tests/tmp/config.json 2>/dev/null && echo "already configured"
```

**Archify** — architecture diagram skill. See CLAUDE.md § Agent tooling above.

---

### Step 9 — Report

Summarize:
- Regex pattern used in the planning gate and why it matches this repo's directories
- Whether an existing `settings.json` was merged (and what keys were preserved)
- Whether any ADRs were written and for what decisions
- Which Habit Hooks plugins were activated and linter thresholds set
- Whether Context7 was verified working
- Any stack-specific adaptations made

Do NOT commit. The human reviews and commits.

---

## END OF PROMPT
