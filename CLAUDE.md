# a_cxnfusing_framework

A batteries-included Roblox Luau framework by `cxnfusing`. Provides a loader that wires up utilities, functions, info tables, services, network events, and tag-discovered modules into a single require surface. Designed to be dropped into a Rojo project and used on both client and server.

Current version string (from `Shared/init.luau`): `1.2.0 - Open Source, Added Utilities`.

---

## Directory Layout

```
src/
  ReplicatedStorage/
    Framework/             Loader. Require this from user Scripts/LocalScripts.
      init.luau            Exposes helpers + delegates module loading to Dependencies.
      Dependencies/
        ModuleLoader.luau  Walks Instances, requires ModuleScripts, runs :Init()/:Start().
        Log.luau           Uniform output formatting.
    Shared/                Leaf require surface. No framework-internal dependencies.
      init.luau            Aggregates Components + Config + Network into one table.
      Config.luau          Game/framework configuration flags.
      Network/             Global (Remote) + Local (Signal) event helpers + SignalWrapper.
      Components/
        Util/              Utilities (Bezier, Pool, Trove, CameraShaker, ...). Bulk of the repo.
        Functions/         Small pure-ish functions (FastTween, FormatNumber, AddCommas, GenerateUID).
        Info/              Shared state tables (PlayerState, GameState).
        Services/          Framework-provided services (VFX, DataService).
    Services/              Higher-level, opinionated services (DialogueService).
    Modules/
      Client/              Tag-or-path loaded client-side modules.
        TagLoader/         Attaches behavior to tagged instances (animated textures, UI animate, presets).
        ExpressivePrompts/ Input prompt UI (keyboard/gamepad/touch). Vendors the Seam UI library.
        GameFeel/          Platformer-flavor feel helpers (CoyoteTime, JumpBuffer, ApexGravity, LandingFX).
    Read Me.server.luau    Author's original overview.
  ServerScriptService/     Server-side entry points and server-only modules.
  StarterGui/              UI handlers (Dialogue, Spectating).
  StarterPlayer/           Client loaders and example usage scripts.
  TestService/             Test scripts.
  Workspace/               In-world example parts (Spring demo).
default.project.json       Rojo project mapping.
aftman.toml                Aftman toolchain pins (Rojo, StyLua, Selene).
selene.toml, stylua.toml   Lint/format config.
Framework.rbxlx            Roblox place file for Studio users.
```

Vendored third-party code (do NOT modify or document as framework surface):

- `src/ReplicatedStorage/Modules/Client/ExpressivePrompts/Packages/_Index/miagobble_seam@0.5.1/**` - Seam UI library.
- `aftman.luau`, `wally.luau` files scattered alongside - package manifests.

---

## Core Concepts

### Two require surfaces

The framework exposes two different entry points depending on what you are writing:

- **`Framework`** (at `ReplicatedStorage.Framework`) - the full loader. Requires `Shared`, runs `ModuleLoader`, exposes all utilities/functions/services plus helper methods (`:WaitUntilLoaded`, `:Log`, `:SafeCall`, `:Preload`, `:IsServer`, `:IsClient`, `:IsStudio`). Use this from regular `Script`/`LocalScript` user code.
- **`Shared`** (at `ReplicatedStorage.Shared`) - the leaf surface. Has the same data tables (utilities, functions, info, services, config, network events) but performs no module loading and depends on nothing framework-internal. Use this from ModuleScripts that are themselves loaded *by* the framework (to avoid circular requires).

`Framework` inherits every data field from `Shared`, so downstream code reads the same values regardless of which one it required.

### `Framework:WaitUntilLoaded()`

`Framework.Loaded` becomes `true` after `ModuleLoader:LoadAll()` finishes running every discovered module's `:Init()` and `:Start()`. Call `:WaitUntilLoaded()` at the top of user scripts that depend on a loaded state (input bindings, UI ready, data profiles available, ...). The method yields via `Framework.OnLoaded:Wait()` if loading is still in progress.

### Tag-loaded Modules

`ModuleLoader:RequireTagged()` connects to the `LoadModule` CollectionService tag. Any `ModuleScript` gaining that tag is required. Optional tags/attributes on a tagged module:

- Tag `NonRecursive` - do not walk into descendants of this module; require only the tagged module itself.
- Tag `NoLoad` - skip this module and its descendants entirely.
- Attribute `Type` = `"Client"` or `"Server"` - only load on the matching RunService context.

A required module that returns a table with `:Init()` and/or `:Start()` has those methods called during `LoadAll()`. Modules may expose a numeric `Priority` field to sort load order (higher priority runs earlier). Init runs for all modules, then Start runs for all modules. `Config.LogModuleLoadTimes` enables per-module timing output.

### Network: Global vs Local events

`Shared/Network/init.luau` exposes two constructors:

- `Network.newGlobal(name: string)` - a RemoteEvent wrapper with `Fire`, `FireAll`, `Connect`, and invoke semantics. Server↔client.
- `Network.newLocal()` - an in-process Signal (no networking). Same process only.

New events are declared by name inside `Shared/init.luau` under `GlobalEvents` / `LocalEvents`. That registration is what makes them show up on `Framework.GlobalEvents` / `Framework.LocalEvents`.

### Freezing

Both `Shared/init.luau` and `Framework/init.luau` freeze their inner tables before returning. `Signal`/event-like tables (anything with both `Fire` and `Connect`) are skipped so they can mutate their connection lists.

---

## Conventions

- **File naming:** Luau source files use `.luau`. Tree is Rojo-mapped via `default.project.json`. Folders with an `init.luau` become ModuleScripts whose children are nested members.
- **`.meta.json` files** live next to folders to set Instance properties (Name, ClassName) for Rojo.
- **Registering new utilities/functions/info/services:** add the ModuleScript under the corresponding `Shared/Components/<folder>/` and append a `Name = require(script:WaitForChild("Name"))` line in that folder's `init.luau`. This is how everything flows up into `Shared` and then `Framework`.
- **Adding a new Global/Local event:** add a `Name = Network.newGlobal("Name")` (or `newLocal()`) entry to the corresponding table in `Shared/init.luau`.
- **Adding a tag-loaded Module:** place the ModuleScript under `ReplicatedStorage.Modules` (or wherever Rojo puts it) and tag it `LoadModule`. Use the `Type` attribute if it is client- or server-only.
- **Style:** StyLua config in `stylua.toml`, Selene lint config in `selene.toml`. Match existing tab-indented Luau style. Comments use `--` for single-line and `--[[ ... ]]` for blocks. The codebase's existing voice is casual/informal - preserve it when editing.
- **No type annotations by default.** Some files have `: Type` annotations but the codebase is not uniformly typed. Don't mass-add types.

---

## How to Add X

- **A utility** → drop file in `Shared/Components/Util/`, require it from `Util/init.luau`.
- **A function** → drop file in `Shared/Components/Functions/`, require it from `Functions/init.luau`.
- **An info table** → drop file in `Shared/Components/Info/`, require it from `Info/init.luau`.
- **A service** → drop file in `Shared/Components/Services/` (or top-level `Services/` for opinionated ones), require it from that folder's `init.luau`.
- **A Global event** → add `Name = Network.newGlobal("Name")` to `Shared/init.luau` → `GlobalEvents`.
- **A Local event** → add `Name = Network.newLocal()` to `Shared/init.luau` → `LocalEvents`.
- **A tag-loaded Module** → place ModuleScript under `Modules/` tree, add `LoadModule` tag. Optional: `NonRecursive`, `NoLoad`, attribute `Type`.

---

## Docs map

- `README.md` — project pitch, install, quickstart, module index, license.
- `docs/index.md` — documentation home / navigation.
- `docs/getting-started.md` — zero-to-booting walkthrough.
- `docs/architecture.md` — load order, two require surfaces, tag-loaded modules, composition diagram.
- `docs/modules/*.md` — one page per module (16 total). Match the module list in `README.md`.
- `docs/reviews/*.md` — per-area quality reviews with concrete fix bullets. Produced during the open-sourcing pass; fixes NOT yet applied.
- `docs/superpowers/specs/2026-04-19-docs-pipeline-design.md` — the spec that produced this docs/reviews/comments pass.
- `docs/superpowers/plans/2026-04-19-framework-docs-pipeline.md` — the implementation plan executed to produce the above.

## Known Bugs (deferred, logged in docs/reviews/)

The rating pass surfaced real issues. They were left alone during commenting so that `git blame` stays meaningful; fix them in a follow-up pass by picking bullets from the relevant review file.

- `Shared/Components/Info/GameState.luau` and `Shared/Components/Util/FlashController.luau` contain stray `::type` literal blocks that are likely syntax errors on strict Luau.
- `Shared/Components/Functions/FastTween.luau` applies an `EasingStyle` lookup where `EasingDirection` is expected.
- `Shared/Components/Util/LODManager.luau:Register` has an operator-precedence bug in the batch-size calculation.
- `Shared/Components/Util/DebugUtils.luau:NewBox` sets `Shape = Ball`.
- `Shared/Components/Util/Pool/Utils.luau` and `Pool/Validation.luau` still have `print("DEBUG: ...")` calls.
- `Services/DialogueService/DialogueHandler/init.luau` has a `repeat task.wait() until x` busy-wait and uses the `for i,v in t do t[i]=nil end` destroy pattern which is undefined behavior in Luau.
- `Services/DialogueService/Actions/Test.luau` prints "HAHAHAHAHAHA" and has no discovery wiring.
- `Services/DialogueService/` has no `init.luau`; the loader contract is implicit.
- `Modules/Client/ExpressivePrompts/NewInputConnections.luau` has a `Scope:New(Scope:New(...))` double-wrap and an unguarded `HOLD_SOUND.PlaybackSpeed` access plus a stale `HoldStart`.
- `Modules/Client/GameFeel/ApexGravity.luau` mutates `workspace.Gravity` globally with a stale-at-require default.
- Duplicate vendored SignalPlus copies exist across `DataService/Signal/` and `Util/Pool/SignalPlus.luau`, byte-identical.

Full bullet lists live in `docs/reviews/{framework,shared,services,client-modules}-review.md`.

## Non-Obvious Things for Future Agents

- The repo has two separate dialogue-related code paths: `Services/DialogueService/` (framework-provided service with a Typewriter, Effects, Conversations) and `StarterGui/Dialogue/` (example UI handler that consumes it).
- `ExpressivePrompts` vendors an entire UI library called Seam (`Packages/_Index/miagobble_seam@0.5.1/`). Treat those files as read-only third-party code. Do not rate, comment, or document them as framework surface.
- `Shared/Components/Services/DataService/` includes a vendored copy of `ProfileStore.luau` plus its own `Signal` and `Networker` subpackages. Treat the ProfileStore file as vendored.
- `StarterPlayer/StarterPlayerScripts/LocalScripts/Main/Bezier.client.luau`, `LODTest.client.luau`, `ViewCheck.client.luau` are example/demo scripts, not framework modules. They show how to consume specific utilities.
- `CameraShaker` was not written by the project author (per the author's Read Me) - treat any comment additions there conservatively.
- `Bezier.luau` was written mostly with AI per the author's note.
- `Framework.Loaded` flips true only after `LoadAll()` completes. Before that, `WaitUntilLoaded` yields on `OnLoaded`. After that, it returns immediately.
- Server-side module loading pulls from `ServerScriptService.Modules`, `ServerScriptService.Services`, and `ReplicatedStorage.Modules.Combined`. Client-side pulls from `ReplicatedStorage.Modules.Client` and `ReplicatedStorage.Modules.Combined`. The `Combined` folder is shared.
- `Framework.rbxlx.lock` exists in the repo; `.gitignore` now excludes `*.rbxlx.lock`.
- This project's docs and reviews live under `docs/` (see specs, plans, reviews, modules). `docs/reviews/` contains follow-up fix bullets produced during the open-sourcing pass; they are not yet applied.
