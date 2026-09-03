# Agentic Quality Harness

  A five-component system that keeps AI coding agents from degrading the codebase.
  Two hard hooks that cannot be bypassed. Three instruction disciplines that require judgment.

   [**Live landing:**](https://sebadp.github.io/harness)

  ## What this is

  A public preview of the *Agentic Quality Harness* — a set of hooks, instructions, and conventions
  originally built for a FastAPI codebase to keep Claude Code (and other AI coding agents) from
  optimizing for the visible metric (passing the linter) at the cost of design quality.

  ## The five components 

  | # | Component | When it acts | Enforcement |
  |---|---|---|---|
  | 1 | **Context reading** | Before starting any task | Instruction (CLAUDE.md) |
  | 2 | **Planning Gate** | Before writing app code | Hook · exit 2 · hard block |
  | 3 | **Habit Hooks** | After every agent turn | Hook · exit 2 · re-inject |
  | 4 | **Testing Pyramid** | After implementing | CI + on-demand + deferred |
  | 5 | **ADR Discipline** | When making a decision | Instruction (CLAUDE.md) |

  ## Navigate

  - [Interactive landing](./landing/agentic-harness.html) — visual overview with the vertical helix
  - [Docs index](./agentic-harness/README.html) — reference for every component
  - [ADRs](./adr/) — decisions behind the design
  - [Bootstrap prompt](./prompts/bootstrap-agentic-harness.md) — install the harness in another repo
  
  ## Component docs

  | Doc | Topic |
  |---|---|
  | [Planning Gate](./agentic-harness/planning-gate.html) | The PreToolUse hook that blocks writes without a plan |
  | [Habit Hooks](./agentic-harness/habit-hooks.html) | The Stop hook wrapping the habit-hooks product |
  | [ADR Discipline](./agentic-harness/adr-discipline.html) | When to write an ADR and the lifecycle |
  | [Testing Pyramid](./agentic-harness/testing-pyramid.html) | Seven layers from types to mutation |
  | [Context Reading](./agentic-harness/determinism.html) | The before-work reading protocol |
  | [Enforcement](./agentic-harness/enforcement.html) | Hard vs soft enforcement matrix |
  | [Anatomy](./agentic-harness/anatomy.html) | Every file the harness touches |
  | [Commands](./agentic-harness/commands.html) | Make targets and manual operations |
  | [CI Integration](./agentic-harness/ci-integration.html) | pre-commit + GitHub Actions |
  
  ## Related products

  - [habit-hooks](https://github.com/habit-hooks/habit-hooks) — MIT product wrapping linter output with coaching guides
  - [Claude Code hooks](https://code.claude.com/docs/en/hooks-guide.md) — the mechanism that powers the exit-2 enforcement
  - [Context7 MCP](https://context7.com) — up-to-date library docs

  ## License

  MIT. This is a preview / mirror. The harness patterns are meant to be adapted, not adopted verbatim.
