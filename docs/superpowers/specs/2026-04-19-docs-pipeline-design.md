# Framework Docs & Review Pipeline - Design

Date: 2026-04-19
Target repo: `a_cxnfusing_framework` (Roblox Luau framework, open-source)

## Goal

Produce a complete documentation and code-quality pass over the framework so it can be published on GitHub for public use by a mixed audience (one senior dev, many new programmers). Three outputs, all produced in one run:

1. A code rating with per-module scores and an actionable fix list (no code changes applied yet).
2. In-code comments upgraded to be noob-friendly (top-of-file headers + inline comments on non-obvious logic).
3. External markdown documentation suitable for direct GitHub rendering, upgradable to GitHub Pages later.

## Audience Assumptions

- Primary consumer: newer programmers who will copy/paste or require modules without reading internals.
- Secondary consumer: experienced devs auditing or extending the framework.
- Comments lean tutorial-style; external docs stay concise and reference-oriented.

## Repo Map (starting point)

```
src/ReplicatedStorage/
  Framework/       core loader, :WaitUntilLoaded(), :Log(), :SafeCall(), :Preload()
  Shared/          Config, Components/{Functions, Info, Services, Util}, Network
  Services/        DialogueService
  Modules/Client/  ExpressivePrompts, GameFeel, TagLoader
  Read Me.server.luau  author's original overview
src/ServerScriptService, StarterGui, StarterPlayer, TestService, Workspace
```

Phase 0 recon produces a complete module inventory before any subagents dispatch.

## Phases

### Phase 0 - Recon (main agent)

- Walk entire `src/` tree, enumerate every module with path + one-line purpose.
- Capture conventions (file naming, `init.luau` patterns, meta.json usage, tag-based loading).
- Write initial `CLAUDE.md` at repo root covering: what the project is, directory layout, core concepts (Framework vs Shared, tag-loaded Modules, Services), conventions, "how to add X" pointers.
- Output: repo inventory held in main context + `CLAUDE.md` at root.

### Phase 1 - Rating (parallel subagents, ~4)

One subagent per top-level area:

- Framework core (`Framework/`)
- Shared (`Shared/` including Components, Network, Config)
- Services (`Services/`)
- Client modules (`Modules/Client/`)

Each subagent:

- Reads every file in its area.
- Scores per-module on: clarity, robustness, API design, noob-friendliness, style consistency (1-10 each, plus one-line justification).
- Lists concrete fixes: bugs, dead code, confusing names, missing error handling, style drift. Each fix is a single actionable bullet with a file:line reference where applicable.
- Writes `docs/reviews/<area>-review.md`.
- Does NOT edit source files.

**Review gate 1:** main agent summarises ratings to user before Phase 2/3 dispatch.

### Phase 2 - Commenting (parallel subagents, one per module)

Each subagent:

- Receives its module's rating file as context.
- Adds a top-of-file header comment block: purpose, when to use, minimal usage example, common gotchas.
- Adds inline comments only on non-obvious logic, explaining *why* not *what*.
- Preserves existing author voice where present.
- Edits files directly. Does not change behavior.

### Phase 3 - External docs (parallel subagents, one per module)

Each subagent writes `docs/modules/<module>.md` with sections:

- Overview (1-2 paragraphs, plain language)
- Require snippet
- API reference (methods/functions with signatures + short descriptions)
- Examples (at least one runnable snippet)
- Gotchas (pulled from rating + code reading)
- Related modules

### Phase 4 - Integration (main agent)

Produce:

- Root `README.md`: project pitch, feature highlights, install/rojo setup, module index table, quickstart, license badge.
- `docs/index.md`: navigation hub linking getting-started, architecture, each module page.
- `docs/getting-started.md`: minimal working example from zero.
- `docs/architecture.md`: how Framework + Shared + tag-loaded Modules + Services compose at runtime.
- Root `LICENSE`: MIT.
- Final pass on `CLAUDE.md` to reflect anything learned in phases 1-3.

**Review gate 2:** user reviews diffs across edited sources and new docs before anything is committed. Commits handled via `/commit` skill if/when repo is initialised.

## Deliverables

```
CLAUDE.md
README.md
LICENSE
docs/
  index.md
  getting-started.md
  architecture.md
  modules/<module>.md              (one per module)
  reviews/<area>-review.md         (kept in repo for follow-up fix pass)
  superpowers/specs/2026-04-19-docs-pipeline-design.md
src/**/*.luau                      (comment edits only, no behavior change)
```

## Non-Goals (this run)

- Applying rating fixes. Logged only; separate run handles them.
- Setting up GitHub Pages. Decided after markdown docs exist.
- Adding tests, CI, or tooling changes.
- Refactoring module APIs or file layout.

## Risks & Mitigations

- **Subagent context drift on large modules.** Each subagent gets only its own module and its own rating file, not the full repo.
- **Inconsistent doc voice across parallel agents.** Phase 4 integration pass normalises tone in nav/index pages; module pages follow a fixed section template defined in Phase 3.
- **Comment edits silently changing behavior.** Phase 2 subagents are instructed to touch only comments; review gate 2 catches regressions before commit.
- **Repo is not currently a git repo** (per environment). Commits deferred until `/git-init` is run; all artifacts still written to disk.

## Success Criteria

- A new programmer can clone the repo, read `README.md` + `docs/getting-started.md`, and successfully require and use at least one module within minutes.
- Each module file opens with a header that answers: what is this, when do I use it, how do I call it.
- Each module has a matching `docs/modules/<module>.md` page.
- Every area has a review file listing concrete follow-up fixes.
- `CLAUDE.md` gives a future Claude session enough context to extend the framework without re-deriving conventions.
