# Enforcement — hard vs soft

The harness is honest about which rules are deterministic and which require judgment.

## Two enforcement classes

### Hard — hook enforced

Exit code `2` blocks the action. Cannot be bypassed by the agent, even with
`permissions.allow` set for the tool. Hooks fire *before* permission evaluation.

| Component | Event | File |
|---|---|---|
| Planning Gate | PreToolUse (Write) | `.claude/hooks/require-plan.sh` |
| Habit Hooks | Stop | `habit-hooks run` |

### Soft — instruction enforced

The agent follows the rule because CLAUDE.md says to. Judgment required — a hook
cannot know if a decision is "architecturally significant" or if a change is
"trivial enough to skip the plan".

| Rule | Where | Why not a hook |
|---|---|---|
| Context reading protocol | CLAUDE.md § Before starting | Reading is semantic, not observable |
| ADR discipline | CLAUDE.md § ADRs | Significance requires context |
| Testing pyramid choice | ADR-002 | Which layer to add a test in requires understanding the risk |
| Trivial fix exemption | CLAUDE.md § Planning discipline | Scope requires judgment |
| `# noqa` suppressions | CLAUDE.md § Design quality | Intentional smells need a comment, not a rule |

## Claude Code hook exit code semantics

| Event | Exit 0 | Exit 1 | Exit 2 |
|---|---|---|---|
| PreToolUse | Allow tool call | Non-blocking error | **Hard block — tool cancelled** |
| Stop | Turn ends normally | Non-blocking error | **Turn cannot finish — stdout re-injected** |
| PostToolUse | Continue | Log error | **Abort turn** |

Only exit `2` is deterministic. Exit `1` is a warning (logged, action continues).
This is Claude Code specific — Unix convention treats `1` as "error"; the hook
system uses `2` deliberately to distinguish "warning" from "block".

## Why this split matters

Attempting to enforce everything with hooks produces false positives (blocking work
that shouldn't be blocked) or false negatives (missing violations that require
context). Attempting to enforce everything with instructions produces drift (the
agent follows the rules when convenient, skips them under pressure).

The harness deliberately splits the workload:

- **Mechanical rules** (path prefix matches app code → require plan) go to hooks.
  These have no ambiguity and benefit from being uncrossable.
- **Judgment calls** (is this decision architecturally significant?) go to
  instructions. The agent uses context to decide, and the human catches misses in
  code review.

## Precedence when they conflict

Not applicable in the current design — hard and soft rules address different
questions. But if a future addition creates overlap: **hard wins**. A hook that
blocks a write cannot be overridden by an instruction that says "actually you can
do this in edge case X". If an edge case appears, either the hook's regex is wrong
(fix it) or the rule needs to be softened (make it an instruction).
