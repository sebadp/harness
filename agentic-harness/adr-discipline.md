# ADR Discipline (Layer 5)

**Layer:** Soft — instruction-enforced via CLAUDE.md
**Files:** `docs/adr/README.md` · `docs/adr/TEMPLATE.md` · `docs/adr/NNN-*.md`
**Origin:** [Michael Nygard, "Documenting Architecture Decisions", 2011](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

## What it is

Architecture Decision Records capture the *why* behind every significant decision,
permanently, alongside the code. Unlike exec plans (pre-implementation alignment),
ADRs are post-decision records that never go stale — they are deprecated or
superseded, never deleted.

> "One of the hardest things to track during the life of a project is the
> motivation behind certain decisions. A new person coming on to a project may be
> perplexed, baffled, delighted, or infuriated by some past decision."
> — Michael Nygard, 2011

## When to write an ADR

Write one when the task:

- Adopts or deprecates a tool, library, or framework
- Introduces a cross-cutting architectural pattern (affects multiple future features)
- Explicitly rejects an alternative that would seem reasonable to a future reader
- Accepts known negative consequences (tech debt, coupling, performance trade-off)

Do NOT write an ADR for:

- Routine feature additions (those go in exec plans only)
- Bug fixes and local refactors
- Config changes with no architectural impact

## Timing — post-decision, not pre-implementation

ADRs are written *alongside or after* the decision — not necessarily before (unlike
exec plans). The typical flow:

1. Create the exec plan (`docs/exec-plans/NN-feature.md`)
2. During planning or implementation, a significant decision crystallizes
3. Extract that decision into an ADR

## Lifecycle

| Status | Meaning |
|---|---|
| `Proposed` | Draft, in discussion |
| `Accepted` | Decision taken, in force |
| `Deprecated` | No longer applies, no direct successor |
| `Superseded by [ADR-NNN]` | Reversed or refined; link to the new ADR |

**ADRs are never deleted.** If a decision is reversed, create a new ADR and update
the old one's status. The reasoning chain stays intact — future readers see the
history of the decision.

## Template

`docs/adr/TEMPLATE.md`:

```markdown
# ADR-NNN: Title (active voice, ≤ 10 words)

**Status:** Proposed | Accepted | Deprecated | Superseded by [ADR-NNN](NNN-name.md)
**Date:** YYYY-MM-DD
**Related:** [Exec Plan NN](../exec-plans/NN-name.md)

## Context
What forces are at play. The problem forcing this decision.
Do NOT include the decision here — only the context.

## Decision
Active voice. "We adopt X." / "We use Y instead of Z." One or two sentences.

## Consequences

### Positive
- ...

### Negative / Accepted trade-offs
- ...

### Neutral
- ...
```

## Enforcement — instruction, not hook

The planning gate (`require-plan.sh`) does NOT block on missing ADRs. Whether a
decision is "architecturally significant" requires semantic context no hook can
provide. The gate's error message mentions ADRs when it blocks, as a reminder to
the agent that if the task adopts a tool or introduces a pattern, an ADR is
appropriate.

The CLAUDE.md rule is the enforcement layer:

```markdown
## Architecture Decision Records (ADRs)

ADRs live in `docs/adr/`. See `docs/adr/README.md` for the index.

Write an ADR when the task adopts/deprecates a tool, introduces a cross-cutting
pattern, rejects a seemingly obvious alternative, or accepts known trade-offs.
ADRs complement exec plans — they are not a replacement.

ADRs are never deleted. If a decision is reversed, create a new ADR with status
`Superseded` and update the old one's status line.
```

## Relationship to exec plans and proposals

```
docs/proposals/NNN-*.md   →  strategic, multi-phase, multiple decisions
docs/exec-plans/NN-*.md   →  concrete implementation plan (scope, changes, checklist)
docs/adr/NNN-*.md         →  one decision, permanent, with context + consequences
```

An exec plan may reference an ADR that already exists, or trigger the creation of
one when it closes. ADRs do NOT replace exec plans — they complement them.

## Current ADRs

See [`docs/adr/README.md`](../adr/README.md) for the index. As of the harness's
initial rollout:

- ADR-001 — Adopt habit-hooks (product) instead of a custom script
- ADR-002 — Seven-layer testing pyramid strategy
- ADR-003 — TestSprite for E2E/API regression (Layer 4)
- ADR-004 — Context7 MCP for library documentation
- ADR-005 — Archify skill for verified architecture diagrams
