# Architecture

How the loader, require surfaces, and tag-loaded modules fit together at runtime.

## The two require surfaces

The framework exposes two different entry points. Picking the right one for each file is the single most important rule to learn.

- **`Framework`** (`ReplicatedStorage.Framework`) is the loader. It:
  1. Requires `Shared` (below).
  2. Walks user-code folders and requires every `ModuleScript` inside them.
  3. Runs each loaded module's `:Init()`, then `:Start()`, in priority order.
  4. Exposes helpers (`:WaitUntilLoaded`, `:Log`, `:SafeCall`, `:Preload`, `:IsServer`, `:IsClient`, `:IsStudio`) and data tables pulled from `Shared`.

  **Use from user code:** regular `Script` / `LocalScript` files where you consume the framework.

- **`Shared`** (`ReplicatedStorage.Shared`) is the leaf data surface. It:
  1. Requires `Components/{Util, Functions, Info, Services}`, `Config`, and `Network`.
  2. Builds one frozen table containing all of them plus event declarations.
  3. Does not load user modules, does not depend on `Framework`.

  **Use from ModuleScripts that are themselves loaded by the framework.** This avoids circular requires: the loader requires `Shared` first, then requires your module, and your module is allowed to require `Shared` back without any loop.

Both surfaces expose the same data fields. `Framework` adds the helpers and the loader state (`Loaded`, `OnLoaded`, `LoadedModules`).

## Load order

```
boot:
  require(ReplicatedStorage.Framework)
    ↓
  Framework/init.luau
    ├── require(ReplicatedStorage.Shared)
    │     Shared/init.luau freezes + returns { GlobalEvents, LocalEvents, Services, Assets, Data, Utilities, Debris, Functions, Info, Config, Version }
    ├── require(Framework.Dependencies.ModuleLoader)
    └── require(Framework.Dependencies.Log)

    branch on RunService:
      isClient → walk ReplicatedStorage.Modules.Client + ReplicatedStorage.Modules.Combined
      isServer → walk ServerScriptService.Modules + ServerScriptService.Services + ReplicatedStorage.Modules.Combined

    for each ModuleScript discovered:
      require it (once), record load time

    ModuleLoader:RequireTagged()    — connects to CollectionService("LoadModule")
    ModuleLoader:LoadAll():
      sort modules by .Priority (higher first)
      for each: if module.Init then module:Init() end
      for each: if module.Start then module:Start() end

    freeze Framework's inner tables (signals skipped)
    Framework.Loaded = true
    Framework.OnLoaded:Fire(elapsed)
```

`Framework:WaitUntilLoaded()` yields on `OnLoaded:Wait()` until that last `Fire` runs. Call it at the top of any user script that depends on the loaded state.

## Tag-loaded Modules

Any `ModuleScript` tagged with the `LoadModule` CollectionService tag gets required by `ModuleLoader:RequireTagged()`. Extra controls:

- Tag `NonRecursive` on the tagged module — do NOT walk descendants, require only the tagged one.
- Tag `NoLoad` on any module or container — skip it and its subtree entirely.
- Attribute `Type = "Client"` or `"Server"` on the module (or a container folder) — only require on the matching RunService context. Note: a `Type` attribute on a folder skips the whole subtree on the wrong context.

A required module that returns a table with `:Init()` and/or `:Start()` methods has those called during `LoadAll`. A module can set a numeric `Priority` field (default `0`); higher priority runs earlier. All `:Init()` runs complete before any `:Start()` runs.

## Network: Global vs Local events

`Shared/Network/init.luau` exposes two constructors. You declare events by name in `Shared/init.luau`:

```lua
GlobalEvents = {
    VFX = Network.newGlobal("VFX"),
    PlayerDied = Network.newGlobal("PlayerDied"),
},

LocalEvents = {
    VFX = Network.newLocal(),
    UIThemeChanged = Network.newLocal(),
},
```

- **`newGlobal(name)`** wraps a `RemoteEvent` + `RemoteFunction` pair under a shared `Remotes` folder. Supports `:Fire`, `:FireAll`, `:Connect`, `:Once`, and invoke (`:Invoke`, `:OnInvoke`). Server ↔ client.
- **`newLocal()`** is an in-process Signal. Same context only, no networking.

See [modules/network.md](modules/network.md) for full method signatures.

## Composition diagram

```
                       ┌─────────────────────────┐
                       │      Framework          │
                       │  (require from user      │
                       │   Scripts/LocalScripts)  │
                       └───────────┬─────────────┘
                                   │ requires
                                   ▼
                       ┌─────────────────────────┐
                       │        Shared           │
                       │ GlobalEvents, LocalEvents│
                       │ Services, Utilities,    │
                       │ Functions, Info, Config │
                       │ Network                 │
                       └───────────┬─────────────┘
                                   │ aggregates
         ┌─────────────────────────┼────────────────────────┐
         ▼                         ▼                        ▼
 ┌──────────────┐        ┌──────────────────┐      ┌───────────────┐
 │  Components  │        │     Network      │      │    Config     │
 │ (Util, Fns,  │        │ newGlobal,       │      │               │
 │  Info, Svcs) │        │ newLocal,        │      └───────────────┘
 └──────────────┘        │ SignalWrapper    │
                         └──────────────────┘

 Framework ALSO scans these containers and requires ModuleScripts inside them:
   isClient → ReplicatedStorage.Modules.Client   + ReplicatedStorage.Modules.Combined
   isServer → ServerScriptService.Modules        + ServerScriptService.Services
             + ReplicatedStorage.Modules.Combined

 PLUS any ModuleScript with the `LoadModule` tag anywhere in the DataModel.
```

## Extending the framework

Summary of the full "How to Add X" list (see [CLAUDE.md](../CLAUDE.md) for the canonical version):

- Utility → `Shared/Components/Util/`, require from `Util/init.luau`.
- Function → `Shared/Components/Functions/`, require from `Functions/init.luau`.
- Info table → `Shared/Components/Info/`, require from `Info/init.luau`.
- Service → `Shared/Components/Services/` for framework services, `Services/` for higher-level opinionated ones. Require from the folder's `init.luau`.
- Global event → add `Name = Network.newGlobal("Name")` to `Shared/init.luau` under `GlobalEvents`.
- Local event → add `Name = Network.newLocal()` to `Shared/init.luau` under `LocalEvents`.
- Tag-loaded module → place under `Modules/`, tag `LoadModule`, optionally set `Type` attribute or `NonRecursive` tag.
