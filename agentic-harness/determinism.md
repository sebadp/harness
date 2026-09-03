# Context Reading (Layer 1) & Agent Determinism

**Layer:** Soft — instruction-enforced via CLAUDE.md
**File:** `CLAUDE.md` § "Before starting any task"

## The gap this closes

`AGENTS.md` existed in this repo (excellent content: code map, agent/human roles,
dev cycle) but `CLAUDE.md` never instructed the agent to read it. An agent that
skips context repeats decisions already made, violates patterns already established,
and misses ADRs that explain why the current design exists.

The reading protocol is the fix.

## The protocol

`CLAUDE.md § Before starting any task`:

```markdown
Execute before touching any file:

1. Read AGENTS.md (or equivalent) — code map, agent/human roles, dev cycle.
2. Read docs/adr/README.md — scan for ADRs relevant to the module you'll touch.
3. Read docs/exec-plans/README.md — identify any 🚧 in-progress plan that touches
   the same area.

Skipping this produces context-free changes that repeat decisions already made.
```

Includes a search helper:

```bash
grep -r "module-name" docs/adr/README.md docs/exec-plans/README.md
```

## Why each source matters

### AGENTS.md — code map + roles

Defines who owns what module, the dev cycle (PLAN → IMPLEMENT → DOCUMENT →
EVALUATE → DELIVER), and what the agent must never do (commit, push, hallucinate
values). Without it, agents violate the protocol on the first try.

### docs/adr/ — avoid blind reversal

Nygard, 2011: *"Changing the decision without understanding its motivation could
mean damaging the project's overall value without realizing it."*

Concrete example: [ADR-001](../adr/001-adopt-habit-hooks.md) explains why we use
habit-hooks instead of a custom script. Without reading it, an agent would generate
a custom `scripts/habit_hook.py` again, replicating work we already discarded.

### docs/exec-plans/ — in-progress work

If Plan 43 (trace diagnosis agent) is 🚧 and the current task touches
`routers/traces.py`, reading Plan 43 prevents incompatible changes or duplicated
work.

## Enforcement — instruction, not hook

A hook cannot verify semantic reading — it cannot know if the agent actually loaded
context or just skimmed. The agent follows this because CLAUDE.md says to, and the
result is visible in the quality of the work.

This is the same enforcement class as ADR discipline. The PreToolUse gate does NOT
open until after the reading is done, but the reading itself is not the gate.

## What "context reading" is NOT

- Not reading the entire codebase every task (that's wasteful and impractical)
- Not memorizing all ADRs (they exist for lookup, not recall)
- Not required for trivial fixes (typos, config tweaks with no architectural impact)

## Broader determinism ideas (not yet implemented)

Beyond context reading, other soft-enforcement patterns being considered:

- **Session log** of what files/docs the agent has read this session
- **Pre-work hook** that verifies AGENTS.md was accessed before app-code writes
  (would need a way to track reads, which Claude Code doesn't currently expose)
- **Reading receipt** — the agent adds a comment at the top of its response
  listing which ADRs/plans it consulted

These are open ideas — none are in the harness today.
