# Agentic Quality Harness

A five-component system that keeps AI coding agents from degrading the codebase.
Two hard hooks that cannot be bypassed. Three instruction disciplines that require judgment.

**Landing page:** [`docs/landing/agentic-harness.html`](../landing/agentic-harness.html) — open in a browser for the visual overview.

---

## The five components

| # | Component | When it acts | Enforcement | Doc |
|---|---|---|---|---|
| 1 | **Context reading** | Before starting any task | Instruction | [determinism.md](determinism.md) |
| 2 | **Planning Gate** | Before writing app code | Hook, exit 2 | [planning-gate.md](planning-gate.md) |
| 3 | **Habit Hooks** | After every agent turn | Hook, exit 2 | [habit-hooks.md](habit-hooks.md) |
| 4 | **Testing Pyramid** | After implementing | CI + on-demand | [testing-pyramid.md](testing-pyramid.md) |
| 5 | **ADR Discipline** | When making a decision | Instruction | [adr-discipline.md](adr-discipline.md) |

## Reference

- [enforcement.md](enforcement.md) — hook exit codes, hard vs soft enforcement matrix
- [anatomy.md](anatomy.md) — every file the harness touches, grouped by layer
- [commands.md](commands.md) — `make` targets and manual operations
- [ci-integration.md](ci-integration.md) — how the harness maps to CI and pre-commit

## Portability

The harness ships with a self-contained Claude Code prompt that installs it in any
other repository. See [`../prompts/bootstrap-agentic-harness.md`](../prompts/bootstrap-agentic-harness.md).

## ADRs behind the design

- [ADR-001](../adr/001-adopt-habit-hooks.md) — why we adopted habit-hooks instead of a custom script
- [ADR-002](../adr/002-testing-pyramid.md) — the seven-layer testing pyramid decision
- [ADR-003](../adr/003-testsprite-e2e-regression.md) — TestSprite as Layer 4
- [ADR-004](../adr/004-context7-mcp.md) — Context7 MCP for library docs
- [ADR-005](../adr/005-archify-diagrams.md) — Archify skill for verified diagrams

## Why this exists

AI coding agents iterate fast and drift faster. Left unchecked, they optimize for the
visible metric (passing the linter) rather than the actual goal (design that reads
well and holds up under change). At scale, this produces code with clean metrics and
degraded design — the "slopocalypse".

The harness closes the feedback loop at every stage of the agent's work cycle: it
loads context before starting, blocks writes without a plan, coaches design after
every turn, gates merges through a testing pyramid, and forces decisions to be
recorded as ADRs. The hooks catch what the model would otherwise skip.
