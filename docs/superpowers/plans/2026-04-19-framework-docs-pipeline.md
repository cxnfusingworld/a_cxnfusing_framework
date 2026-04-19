# Framework Docs & Review Pipeline - Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a full code rating, in-code comment upgrade, and external markdown documentation set for the `a_cxnfusing_framework` Roblox Luau framework so it is ready for public open-source release.

**Architecture:** Four phases. Phase 0 = main-agent recon + initial CLAUDE.md. Phase 1 = four parallel rating subagents (one per top-level area) producing review files. Phase 2 = parallel commenting subagents (one per module) editing sources in place. Phase 3 = parallel doc subagents (one per module) writing `docs/modules/<module>.md`. Phase 4 = main agent produces root README, LICENSE, and `docs/` nav pages.

**Tech Stack:** Roblox Luau, Rojo project layout, plain Markdown docs. No build system or tests are added in this run.

**Spec:** `docs/superpowers/specs/2026-04-19-docs-pipeline-design.md`

**Repo is not currently a git repo.** Commits are deferred. User review gates replace commit checkpoints.

---

## File Structure

New:

```
CLAUDE.md                                                (root, Phase 0 draft + Phase 4 polish)
README.md                                                (root, Phase 4)
LICENSE                                                  (root, Phase 4, MIT)
docs/index.md                                            (Phase 4)
docs/getting-started.md                                  (Phase 4)
docs/architecture.md                                     (Phase 4)
docs/modules/<module>.md                                 (Phase 3, one per module)
docs/reviews/framework-review.md                         (Phase 1)
docs/reviews/shared-review.md                            (Phase 1)
docs/reviews/services-review.md                          (Phase 1)
docs/reviews/client-modules-review.md                    (Phase 1)
```

Modified:

```
src/ReplicatedStorage/Framework/**/*.luau                (Phase 2, comments only)
src/ReplicatedStorage/Shared/**/*.luau                   (Phase 2, comments only)
src/ReplicatedStorage/Services/**/*.luau                 (Phase 2, comments only)
src/ReplicatedStorage/Modules/Client/**/*.luau           (Phase 2, comments only)
```

Module list is finalised in Task 1.

---

## Phase 0 - Recon

### Task 1: Enumerate every module in the repo

**Files:**
- Read: `src/ReplicatedStorage/**/*.luau`, `src/ServerScriptService/**/*.luau`, `src/StarterGui/**/*.luau`, `src/StarterPlayer/**/*.luau`, `src/TestService/**/*.luau`, `src/Workspace/**/*.luau`
- Create: working inventory (in conversation scratch, not a file)

- [ ] **Step 1: Glob every `.luau` file under `src/`**

Run (via Glob tool):

Pattern: `src/**/*.luau`

Expected: list of all Luau source files.

- [ ] **Step 2: Group files into four rating areas**

Produce a grouping with exact paths:

- `Framework/` → everything under `src/ReplicatedStorage/Framework/`
- `Shared/` → everything under `src/ReplicatedStorage/Shared/`
- `Services/` → everything under `src/ReplicatedStorage/Services/`
- `Modules/Client/` → everything under `src/ReplicatedStorage/Modules/Client/`
- `Other/` → anything outside the above (ServerScriptService, StarterGui, StarterPlayer, TestService, Workspace). If `Other/` is non-empty, note it and decide in Step 3 whether to add a fifth rating area.

- [ ] **Step 3: Build module-level list for Phases 2 and 3**

Each "module" = one logical unit that will get its own `docs/modules/<module>.md` and its own commenting subagent. Granularity rule: anything with an `init.luau` or that is referenced by name in the Read Me file counts as a module. Single-file utilities under `Shared/Components/Util/` can be grouped into one `utilities.md` doc page to avoid doc spam.

Expected output: a numbered list like

```
1. Framework (core loader)
2. Shared (root require surface)
3. Network (Global + Local events)
4. DialogueService
5. ExpressivePrompts
6. GameFeel
7. TagLoader
8. Utilities (grouped: ActionBuffer, Bezier, CameraShaker, ConnectToAdded, ConnectToTag, ...)
9. Info (grouped: PlayerState, GameState, ...)
10. Functions (grouped)
11. Services (grouped: VFX, DataService, ...)
```

Actual numbering depends on what Step 1 finds.

- [ ] **Step 4: User review of module list**

Post the numbered list to the user and ask: "This is the module list I'll use for Phases 1-3. Merge/split anything before I dispatch?"

Wait for response. Update list if requested.

### Task 2: Write initial CLAUDE.md

**Files:**
- Create: `CLAUDE.md` (repo root)

- [ ] **Step 1: Write the file**

Sections, in order:

1. `# a_cxnfusing_framework` - one-paragraph project description derived from `src/ReplicatedStorage/Read Me.server.luau`.
2. `## Directory Layout` - tree of `src/` with one-line purpose per top-level folder.
3. `## Core Concepts` - short prose on:
   - `Framework` module vs `Shared` module (the two require surfaces).
   - `:WaitUntilLoaded()` and why.
   - Tag-loaded Modules (`LoadModule` tag, `NonRecursive`, `Type` attribute).
   - Global vs Local events via `Network`.
4. `## Conventions` - file naming (`init.luau`, `.meta.json`), how new utilities/functions/info/services are registered (append to parent `init.luau` `require(...)` tables), Luau style (match existing `stylua.toml` / `selene.toml`).
5. `## How to Add X` - bullets mirroring the Read Me file's guidance.
6. `## Non-Obvious Things for Future Agents` - notes captured during Task 1 (e.g., undocumented files, Other/ folder contents, anything surprising).

Keep total under 300 lines. Plain markdown, no emojis.

- [ ] **Step 2: Spot-check**

Grep the new file for "TODO", "TBD", "???". Must return zero matches.

- [ ] **Step 3: User checkpoint**

Show the file path and summary to the user. Ask: "CLAUDE.md drafted. Proceed to Phase 1 rating?"

Wait for confirmation.

---

## Phase 1 - Rating (parallel subagents)

### Task 3: Dispatch four rating subagents in parallel

**Files:**
- Create: `docs/reviews/framework-review.md`
- Create: `docs/reviews/shared-review.md`
- Create: `docs/reviews/services-review.md`
- Create: `docs/reviews/client-modules-review.md`

- [ ] **Step 1: Single message, four Agent tool calls**

Use `subagent_type: "general-purpose"` for all four. Dispatch in parallel (one message, four tool blocks).

Each agent gets the same prompt template, with `<AREA>` and `<PATHS>` substituted:

```
You are rating the code quality of one area of a Roblox Luau framework that is about to be open-sourced for a public audience mixing one senior developer and many new programmers.

AREA: <AREA>
PATHS: <PATHS>

Project context:
- The framework is `a_cxnfusing_framework`, a Roblox Luau project using Rojo layout.
- Two primary require surfaces exist: `Framework` (with dependency loading) and `Shared` (leaf utilities only).
- Tag-based Modules load via `LoadModule` CollectionService tag.
- See repo `CLAUDE.md` for conventions.

Your job:
1. Read every file in PATHS. Do NOT read files outside PATHS.
2. Produce a markdown review file at `docs/reviews/<AREA-slug>-review.md` with this exact structure:

    # <AREA> Review

    ## Summary
    One paragraph overall impression.

    ## Modules

    ### <module-name>
    Path: `<path>`

    Scores (1-10, 1 = bad, 10 = great):
    - Clarity: N - one sentence justification
    - Robustness: N - one sentence justification
    - API design: N - one sentence justification
    - Noob-friendliness: N - one sentence justification
    - Style consistency: N - one sentence justification

    Fixes:
    - [ ] `<relative/path>:<line>` - <concrete actionable fix, one line>
    - [ ] ...

    (repeat for every module in this area)

    ## Cross-cutting Issues
    Bullets for problems that span multiple modules in this area.

3. Do NOT edit any source file. You are read-only except for the review file.
4. Fix bullets must be specific enough that another agent can act on them without re-reading the code. Prefer `file.luau:42 - rename local variable 'x' to something descriptive` over `improve naming`.
5. Return a one-paragraph summary of the review as your final message.
```

Area substitutions:

- Area `Framework core` → paths `src/ReplicatedStorage/Framework/**` → output `framework-review.md`
- Area `Shared` → paths `src/ReplicatedStorage/Shared/**` → output `shared-review.md`
- Area `Services` → paths `src/ReplicatedStorage/Services/**` → output `services-review.md`
- Area `Modules/Client` → paths `src/ReplicatedStorage/Modules/Client/**` → output `client-modules-review.md`

- [ ] **Step 2: Verify all four review files exist**

Run (via Glob): `docs/reviews/*-review.md`

Expected: four files.

If any agent failed, re-dispatch only that one.

### Task 4: Review gate 1 - summarise ratings to user

**Files:**
- Read: all four `docs/reviews/*-review.md`

- [ ] **Step 1: Extract summaries**

From each review file, pull the `## Summary` paragraph and the lowest-scored module per area.

- [ ] **Step 2: Post aggregate to user**

Format:

```
Phase 1 done. Per-area impression:
- Framework core: <summary>
- Shared: <summary>
- Services: <summary>
- Modules/Client: <summary>

Weakest modules flagged:
- <module> (score <avg>) - <one line>
- ...

Proceed to Phase 2 (commenting) + Phase 3 (external docs)?
```

- [ ] **Step 3: Wait for user approval**

Do not dispatch Phase 2/3 until user says go.

---

## Phase 2 - Commenting (parallel subagents)

### Task 5: Dispatch one commenting subagent per module

**Files:**
- Modify: every `.luau` file owned by each module (comments only)

- [ ] **Step 1: Build per-module dispatch list**

For each module from Task 1 Step 3, compute:

- `<module-name>` (for logging)
- `<module-paths>` (glob or explicit file list)
- `<review-file>` (the Phase 1 review that covers it)

- [ ] **Step 2: Dispatch all commenting subagents in parallel**

One message, one Agent tool call per module. Use `subagent_type: "general-purpose"`.

Prompt template (substitute `<MODULE>`, `<PATHS>`, `<REVIEW_FILE>`):

```
You are upgrading in-code comments for ONE module of a Roblox Luau framework being open-sourced for a mixed audience (one senior dev, many beginners). The audience skews toward beginners.

MODULE: <MODULE>
PATHS: <PATHS>
REVIEW FILE (for context only, do not act on fix bullets): <REVIEW_FILE>

Rules:
1. Edit ONLY the files in PATHS. Do not touch anything else.
2. Do NOT change behavior. Only comments. If you see a bug, LEAVE IT; it is already in the review file.
3. For every `.luau` file in PATHS, ensure the top of the file has a header comment block with these sections (use Luau `--[[ ... ]]` block comments):

    Purpose: what this module/file is for, one or two sentences, plain language.
    When to use: concrete situations a beginner would recognise.
    Example: a short require+call snippet. Must compile.
    Gotchas: anything non-obvious (yield behavior, server/client only, tag requirements). Omit the section if none.

4. Inside the file, add inline comments ONLY where logic is non-obvious. Explain WHY, not WHAT. Do not comment `local x = 5 -- set x to 5`. Do comment `-- We retry once because :GetAsync can transiently 500 on cold cache`.
5. Preserve the original author's voice if it is casual (the project uses informal tone). Do not rewrite existing good comments.
6. Do not create new files. Do not rename anything. Do not add type annotations.
7. When done, return a one-paragraph summary listing which files you touched.

Reference `CLAUDE.md` at the repo root for conventions.
```

- [ ] **Step 3: Verify no behavior changes**

For each edited file, compute: the set of non-comment lines must be unchanged vs the pre-edit version. Do this by asking a fresh subagent (or using git diff if repo is initialised by then).

If the repo has no git history, run this check per-file via a verification subagent with the prompt:

```
Compare the current contents of <file> against your understanding of the pre-edit version. Confirm that only comments, whitespace inside block comments, or header doc blocks changed. Report any line where executable code differs.
```

Flag any file where non-comment lines changed and revert/re-run that module.

### Task 6: User checkpoint after commenting

- [ ] **Step 1: Post module-by-module status to user**

```
Phase 2 done. Modules commented:
- <module> - <file count> files touched
- ...

Ready for Phase 3 external docs, or want to spot-check comments first?
```

- [ ] **Step 2: Wait for user response**

If spot-check requested, pause and let user read. Otherwise proceed.

---

## Phase 3 - External docs (parallel subagents)

### Task 7: Dispatch one external-doc subagent per module

**Files:**
- Create: `docs/modules/<module>.md` (one per module from Task 1 Step 3)

- [ ] **Step 1: Dispatch in parallel**

One message, one Agent tool call per module. `subagent_type: "general-purpose"`.

Prompt template (substitute `<MODULE>`, `<PATHS>`, `<REVIEW_FILE>`, `<OUTPUT_FILE>`):

```
You are writing public-facing documentation for ONE module of a Roblox Luau framework being released as open source. The audience is beginners plus one senior dev. Keep tone friendly and direct.

MODULE: <MODULE>
SOURCE PATHS: <PATHS>
REVIEW FILE (context only): <REVIEW_FILE>
OUTPUT FILE: <OUTPUT_FILE>

Write exactly one markdown file at OUTPUT_FILE with these sections in order:

    # <MODULE>

    ## Overview
    One or two paragraphs in plain language. What problem does this solve? Who would use it?

    ## Requiring it
    Fenced `lua` code block showing the exact require path. Both `Framework` and `Shared` paths if applicable. Include `:WaitUntilLoaded()` when relevant.

    ## API reference
    For each public method/function/field, a subsection:
        ### `:MethodName(arg1: Type, arg2: Type): ReturnType`
        One-sentence description. Arguments list. Return value. Yields? Server/client only? Tag required?

    ## Examples
    At least one runnable example showing a realistic use case. Fenced `lua`. Include the require lines.

    ## Gotchas
    Bulleted list. Draw from the review file's "Fixes" bullets where they reflect real user-facing surprises, plus anything you discover while reading the code.

    ## Related
    Links to other `docs/modules/<other>.md` pages that pair with this one. Use relative links.

Rules:
1. Read only the files in SOURCE PATHS and the REVIEW FILE and repo `CLAUDE.md`.
2. Do NOT edit any source file.
3. Do NOT invent APIs. If a method exists but is undocumented, document it based on reading the code. If unsure about a method, put it under a final `## Unclear` section with a note.
4. Code blocks must compile against the framework as it exists.
5. Keep the whole file under roughly 400 lines.
6. Return a one-paragraph summary of what you wrote as the final message.
```

- [ ] **Step 2: Verify every module has a doc**

Run (via Glob): `docs/modules/*.md`

Expected: one file per module in the Task 1 list. Re-dispatch any missing.

- [ ] **Step 3: Cross-link audit**

Read every `docs/modules/*.md`. Check each `## Related` link resolves to an actual file in `docs/modules/`. Fix broken links inline (this is a main-agent edit, not a subagent dispatch).

---

## Phase 4 - Integration (main agent)

### Task 8: Write root README.md

**Files:**
- Create: `README.md` (repo root)

User indicated either `/readme` skill or inline is fine. Use inline for speed; the skill can be re-run later if desired.

- [ ] **Step 1: Draft the README**

Sections:

1. `# a_cxnfusing_framework` + one-line tagline ("A batteries-included Roblox Luau framework for beginners and senior devs alike").
2. `## What's in the box` - feature bullets from the Read Me file (Events, Services, Utilities, Functions, Info, Config, Debris, helpers).
3. `## Install` - instructions for both paths:
   - Rojo users: clone + `rojo build default.project.json`.
   - Plain Studio users: open `Framework.rbxlx`.
4. `## Quickstart` - minimal `require(Framework); Framework:WaitUntilLoaded()` example.
5. `## Module index` - markdown table with columns `Module | What it does | Docs`. One row per `docs/modules/*.md`. Link column points at the doc file.
6. `## Documentation` - pointer to `docs/index.md` and `docs/getting-started.md`.
7. `## License` - "MIT, see LICENSE."
8. `## Credits` - credit the original author (pulled from the Read Me ASCII art: `cxnfusing`).

- [ ] **Step 2: Grep for placeholders**

Run (via Grep): pattern `TODO|TBD|\?\?\?` in `README.md`.

Expected: no matches.

### Task 9: Write docs/index.md

**Files:**
- Create: `docs/index.md`

- [ ] **Step 1: Write the nav hub**

Sections:

1. `# Documentation` short intro.
2. `## Start here` - links to `getting-started.md` and `architecture.md`.
3. `## Modules` - same table as README's module index.
4. `## Reviews` - link to `reviews/` folder explaining it contains internal quality notes.

### Task 10: Write docs/getting-started.md

**Files:**
- Create: `docs/getting-started.md`

- [ ] **Step 1: Write the guide**

Must take a reader from empty Rojo project to successful `Framework:WaitUntilLoaded()` call. Steps:

1. Install Rojo + Aftman (match `aftman.toml`).
2. Clone repo.
3. Serve with Rojo.
4. In a `Script` in `ServerScriptService`, paste:
   ```lua
   local rs = game:GetService("ReplicatedStorage")
   local Framework = require(rs:WaitForChild("Framework"))
   Framework:WaitUntilLoaded()
   print("framework ready")
   ```
5. Play-test, expect `framework ready` in output.
6. "Next: read `architecture.md` or jump into a module doc."

### Task 11: Write docs/architecture.md

**Files:**
- Create: `docs/architecture.md`

- [ ] **Step 1: Write the architecture doc**

Sections:

1. `## The two require surfaces` - `Framework` (deps, loading, helpers) vs `Shared` (leaves only, no deps).
2. `## Load order` - Framework `init.luau` behavior, Dependencies folder, `:WaitUntilLoaded()`.
3. `## Tag-loaded Modules` - `LoadModule`, `NonRecursive`, `Type` attribute, what runs where.
4. `## Network` - Global (RemoteEvents with invoke) vs Local (Signal) events, how to add new ones.
5. `## Extending the framework` - pointer list to "How to Add X" bullets in `CLAUDE.md`.
6. Diagram (ASCII box-and-arrow) showing Framework → Shared, Framework → Services, Modules loaded via tag scanning.

### Task 12: Write LICENSE (MIT)

**Files:**
- Create: `LICENSE`

- [ ] **Step 1: Write standard MIT text**

Use the canonical MIT license text. Copyright line: `Copyright (c) 2026 cxnfusing`.

- [ ] **Step 2: Verify**

First line must be `MIT License`. Must contain the "Permission is hereby granted" paragraph and the "THE SOFTWARE IS PROVIDED" paragraph. No modifications to the license text.

### Task 13: Final CLAUDE.md polish

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add a `## Docs map` section**

Bulleted list pointing to:

- `README.md` - project pitch.
- `docs/index.md` - nav hub.
- `docs/getting-started.md` - quickstart.
- `docs/architecture.md` - composition model.
- `docs/modules/` - one page per module.
- `docs/reviews/` - actionable fix lists (follow-up pass).

- [ ] **Step 2: Update `## Non-Obvious Things for Future Agents`**

Append anything learned in Phases 1-3 that was not in the initial draft (e.g., modules that diverged from convention, missing pieces, deferred fixes).

### Task 14: Review gate 2 - user inspects everything

- [ ] **Step 1: List deliverables to user**

```
Phase 4 done. Deliverables:
- CLAUDE.md (updated)
- README.md
- LICENSE (MIT)
- docs/index.md, docs/getting-started.md, docs/architecture.md
- docs/modules/*.md (N files)
- docs/reviews/*.md (4 files)
- Comment edits across src/

Suggest next: run `/git-init` then `/commit` so this lands as a clean initial commit. After that, `/bepy-project-setup` + `/apply-styleguide` + `/favicon` when we build the GitHub Pages site.
```

- [ ] **Step 2: Wait for user review**

User reads, requests tweaks. Loop until approved.

---

## Self-Review

**Spec coverage:**
- Phase 0 recon + CLAUDE.md → Tasks 1-2. ✓
- Phase 1 rating with four subagents → Task 3. ✓
- Review gate 1 → Task 4. ✓
- Phase 2 commenting per module → Tasks 5-6. ✓
- Phase 3 external docs per module → Task 7. ✓
- Phase 4 integration deliverables (README, LICENSE, docs/index, getting-started, architecture, CLAUDE.md polish) → Tasks 8-13. ✓
- Review gate 2 → Task 14. ✓
- Non-goal: no fixes applied. Plan honours this: Phase 1 subagents read-only, Phase 2 subagents comments-only, verified in Task 5 Step 3.
- Non-goal: no GitHub Pages setup. Plan honours this: README points at markdown only; Pages deferred per user confirmation.

**Placeholder scan:** No `TBD`/`TODO`/`fill in later` in this plan. Every subagent prompt is complete. Every section shows exact structure.

**Type consistency:** Review file names consistent (`framework-review.md`, `shared-review.md`, `services-review.md`, `client-modules-review.md`). Module doc path consistent (`docs/modules/<module>.md`). Tag name consistent (`LoadModule`). `:WaitUntilLoaded()` spelled identically in README, getting-started, and CLAUDE.md references.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-19-framework-docs-pipeline.md`. Two execution options:

1. **Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration. Fits this plan well because Tasks 3, 5, and 7 already dispatch parallel subagents.
2. **Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
