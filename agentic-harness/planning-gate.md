# Planning Gate

**Layer:** Hard enforcement · Claude Code PreToolUse hook · exit 2
**Files:** `.claude/hooks/require-plan.sh` · `.claude/settings.json` · `.claude/plan-ref`
**Related ADR:** none (fundamental primitive)

## What it does

Fires before every `Write` tool call. If the destination path is application code
(`email_triage/`, `tests/`, `frontend/src/`, `.github/workflows/`, `scripts/`), the
hook checks two conditions in order:

1. `.claude/plan-ref` exists and is non-empty (agent wrote a plan number here after human approval).
2. `git status --short docs/exec-plans/` shows a new/modified file (plan was created this session).

If neither passes, the hook writes a coaching message to stderr and exits `2`. Claude
Code cancels the tool call — the model does not even see the error, it sees only that
the write did not happen.

## Why exit 2

The Claude Code hook system treats exit codes as follows:

| Exit | Behavior |
|---|---|
| 0 | Allow the tool call |
| 1 | Non-blocking error (logged, tool continues) |
| **2** | **Hard block — tool call cancelled, stderr becomes the reason** |

Exit `2` is the only code that deterministically prevents an action. Even
`permissions.allow` in `settings.json` cannot override it — hooks fire before
permission evaluation.

## The hook script

`.claude/hooks/require-plan.sh`:

```bash
#!/usr/bin/env bash
set -uo pipefail

INPUT=$(cat)
FILE_PATH=$(printf '%s' "$INPUT" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    print(d.get('tool_input',{}).get('file_path',''))
except: print('')
" 2>/dev/null || echo "")

# Only gate writes to application code
if ! printf '%s' "$FILE_PATH" \
    | grep -qE "^(email_triage|tests|frontend/src|\.github/workflows|scripts)/"; then
  exit 0
fi

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"

# Gate 1: session plan reference
if [ -f "$REPO_ROOT/.claude/plan-ref" ] && [ -s "$REPO_ROOT/.claude/plan-ref" ]; then
  exit 0
fi

# Gate 2: exec plan created this session
PLAN_STATUS=$(git -C "$REPO_ROOT" status --short "docs/exec-plans/" 2>/dev/null | grep -v "^$" || true)
[ -n "$PLAN_STATUS" ] && exit 0

# Block
cat >&2 <<EOF
PLANNING GATE — write blocked
  ...coaching message with unlock instructions...
EOF
exit 2
```

## How the agent unlocks the gate

Two paths:

**Path A — new work in this session:** The agent creates `docs/exec-plans/NN-feature.md`.
Once the plan file appears in `git status`, Gate 2 opens automatically for the rest of
the session. This is the normal path.

**Path B — continuing approved work:** The agent writes the plan number to
`.claude/plan-ref`:

```bash
echo "50" > .claude/plan-ref
```

This unlocks Gate 1 for the rest of the session. Useful when resuming work on a plan
that already exists (e.g., picking up a 🚧 in-progress plan after a break).

`.claude/plan-ref` is in `.gitignore` — it's session state, never committed.

## Adapting the source-dir regex to another repo

The regex `^(email_triage|tests|frontend/src|\.github/workflows|scripts)/` matches this
repo's structure. For other projects:

| Stack | Suggested regex |
|---|---|
| Python package | `^(mypackage\|tests\|scripts)/` |
| Node monorepo | `^(packages\|apps\|src)/` |
| Rails | `^(app\|spec\|config)/` |
| Go project | `^(cmd\|internal\|pkg)/` |

The bootstrap prompt at [`../prompts/bootstrap-agentic-harness.md`](../prompts/bootstrap-agentic-harness.md)
instructs the agent to explore the target repo first and choose the right regex.

## Verification

```bash
# Should BLOCK (exit 2) if no plan-ref and no exec-plan changes
echo '{"tool_name":"Write","tool_input":{"file_path":"email_triage/x.py","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 2

# Should ALLOW (exit 0) — docs are always writable
echo '{"tool_name":"Write","tool_input":{"file_path":"docs/plan.md","content":"x"}}' \
  | bash .claude/hooks/require-plan.sh
echo "exit: $?"   # expect 0
```

## Known limitations

- The regex is a coarse filter. A path like `email_triage_docs/x.md` would match and
  block a write it shouldn't. In practice this hasn't been an issue but it's worth
  noting for repos with docs sitting under a package directory.
- The `git status` gate requires the repo to be initialized. Fresh clones without any
  commits will not have `git rev-parse --show-toplevel` succeed cleanly; the hook
  degrades to `pwd`.
- The hook cannot see `Edit` operations differently from `Write` — the matcher is
  `Write` only. `Edit` to app code files is currently not gated.
