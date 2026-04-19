# Framework

## Overview

`Framework` is the main entry point of the loader. You require it once per context (one `Script` on the server, one `LocalScript` on the client), and it pulls in the `Shared` data surface, walks the `Modules` folders, requires every `ModuleScript` it finds, and runs their `:Init()` then `:Start()` lifecycle methods. Everything the project exposes (utilities, functions, info tables, services, global/local events, config) lives on the returned table.

Beginners use it as a single "everything" import for user scripts. The senior dev uses it as the orchestration point: it decides which folders to crawl based on server vs client, hooks the `LoadModule` CollectionService tag, freezes inner tables to stop accidental mutation, and fires `OnLoaded` once the whole pipeline finishes.

## Requiring it

```lua
-- From a Script (server) or LocalScript (client):
local Framework = require(game:GetService("ReplicatedStorage"):WaitForChild("Framework"))

Framework:WaitUntilLoaded()

Framework:Log("MyScript", "hello world", "warn")
```

Inside a ModuleScript that is itself loaded *by* the framework, prefer `require(ReplicatedStorage.Shared)` instead to avoid circular requires.

## API reference

### `:IsServer(): boolean`
Returns `true` on the server. Thin wrapper over `RunService:IsServer()`. Safe to call any time.

### `:IsClient(): boolean`
Returns `true` on the client. Thin wrapper over `RunService:IsClient()`.

### `:IsStudio(): boolean`
Returns `true` when the game is running inside Roblox Studio (any context, play-solo or team-test).

### `:Log(title: string, info: string, logType: "warn" | "error" | nil)`
Prints a framework-formatted banner to Output. `"warn"` routes to `warn`, `"error"` routes to `error` (which halts the calling thread), anything else (including `nil`) routes to plain `print`. Each level is gated by a matching `Config` toggle (`LogsEnabled`, `WarningLogsEnabled`, `ErrorLogsEnabled`). No yield for `warn`/`print`. `"error"` does not return.

### `:SafeCall(func: (any) -> (any))`
Forwards to `Utilities.DebugUtils.SafeCall`. Calls `func` in a protected context so a thrown error does not propagate. Does not yield beyond whatever `func` itself yields on.

### `:Preload(IDs: {number}, callback: () -> ())`
Thin wrapper over `ContentProvider:PreloadAsync(IDs, callback)`. **Yields** the calling thread until the assets finish loading (or fail). Works on both client and server, though typical use is client-side.

### `:WaitUntilLoaded()`
Yields the calling thread until `Framework.Loaded == true`. Returns immediately if loading has already finished. Use this at the top of any user script that depends on modules having run `:Init`/`:Start`.

### `.GlobalEvents: {[string]: GlobalEvent}`
Alias of `Shared.GlobalEvents`. Map of named RemoteEvent wrappers (`Fire`, `FireAll`, `Connect`, invoke semantics). Cross-process (server↔client). Table is frozen; add new entries in `Shared/init.luau`, not at runtime.

### `.LocalEvents: {[string]: LocalEvent}`
Alias of `Shared.LocalEvents`. Map of in-process signals. Same process only. Table is frozen.

### `.Services: {[string]: Service}`
Alias of `Shared.Services`. Includes Roblox services used by the framework (e.g. `RunService`, `ContentProvider`) plus the framework's own services sourced from `Shared/Components/Services/`.

### `.Assets`
Alias of `Shared.Assets`. Lookup table for asset references declared in `Shared`.

### `.Data`
Alias of `Shared.Data`. Static data tables declared in `Shared`.

### `.Utilities`
Alias of `Shared.Utilities`. The bulk of the library: `Bezier`, `Pool`, `Trove`, `CameraShaker`, `Signal`, `DebugUtils`, and friends. Frozen at top level.

### `.Debris`
Alias of `Shared.Debris`. Framework's debris helper surface.

### `.Functions`
Alias of `Shared.Functions`. Small pure-ish helpers such as `FastTween`, `FormatNumber`, `AddCommas`, `GenerateUID`.

### `.Info`
Alias of `Shared.Info`. Shared runtime state tables (`PlayerState`, `GameState`).

### `.Config`
Alias of `Shared.Config`. Framework toggles: `LogsEnabled`, `WarningLogsEnabled`, `ErrorLogsEnabled`, `LogModuleLoadTimes`, etc.

### `.Version: string`
Alias of `Shared.Version`. Human-readable version string (e.g. `"1.2.0 - Open Source, Added Utilities"`).

### `.Loaded: boolean`
`false` while modules are still loading, flipped to `true` after `ModuleLoader:LoadAll()` completes. Read-only from user code.

### `.OnLoaded: Signal`
Fires exactly once, after loading finishes, with one argument: the total load time in seconds. Prefer `:WaitUntilLoaded()` over connecting directly unless you need the timing value.

### `.LoadedModules: {[string]: any}`
Map of `ModuleScript:GetFullName()` → whatever that module returned. Populated after `:LoadAll()` finishes. Useful for debugging; no autocomplete because the keys are dynamic.

## Examples

A client script that waits for load, preloads some assets, then connects a global event:

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage:WaitForChild("Framework"))

Framework:WaitUntilLoaded()

Framework.OnLoaded:Connect(function(loadTimeSeconds)
    Framework:Log("Boot", string.format("Loaded in %.2fs", loadTimeSeconds))
end)

Framework:Preload({ 123456789 }, function()
    Framework:Log("Preload", "Assets ready")
end)

if Framework:IsClient() then
    Framework.GlobalEvents.MyEvent:Connect(function(payload)
        Framework:Log("MyEvent", tostring(payload), "warn")
    end)
end

Framework:SafeCall(function()
    error("this will not crash the script")
end)
```

A tag-loaded module (tag the `ModuleScript` with `LoadModule`) that plugs into the lifecycle:

```lua
local MyModule = {}

MyModule.Priority = 10 -- higher runs earlier

function MyModule:Init()
    -- siblings' Init has NOT finished yet; do setup here
end

function MyModule:Start()
    -- every module's Init has finished by the time Start runs
end

return MyModule
```

## Gotchas

- Requiring `Framework` triggers the full module-load pipeline. Do it **once per context**: one server `Script`, one `LocalScript`. Requiring it a second time is harmless (Roblox caches the return) but conceptually misleading.
- Inside a `ModuleScript` that is itself loaded by the framework, require `Shared` instead of `Framework` to avoid circular requires.
- The inner tables (`Utilities`, `Functions`, `Services`, `Config`, etc.) are `table.freeze`d before the framework returns. Signal-like tables (anything with both `Fire` and `Connect`) are skipped so they can mutate their connection lists. Do not mutate the frozen tables at runtime, register new entries in `Shared/init.luau` instead.
- Server crawls `ServerScriptService.Modules`, `ServerScriptService.Services`, and `ReplicatedStorage.Modules.Combined`. Client crawls `ReplicatedStorage.Modules.Client` and `ReplicatedStorage.Modules.Combined`. A fresh project missing one of those folders will error on the server path (the server branch does not use `WaitForChild`).
- The `Type` attribute (`"Client"` or `"Server"`) works on **container folders**, not just `ModuleScript`s. A folder with `Type="Server"` will silently skip its entire subtree on the client. Surprising but intentional.
- There is **no error isolation** around the load pipeline. One bad `require`, `:Init`, or `:Start` will abort loading, which means `Loaded` never flips true and `:WaitUntilLoaded` yields forever. Keep module tops clean.
- A module tagged `LoadModule` **after** `:LoadAll` has run will be required, but its `:Init`/`:Start` will **not** run. Do your tagging before the framework finishes loading, or call the lifecycle yourself.
- `:Log(..., "error")` calls `error()` and halts the calling thread. If you want a non-fatal warning, pass `"warn"` (or nothing).
- The success banner that prints when the framework finishes loading uses `"warn"` log level, so it shows up yellow in Output. Not a bug, just stylistic.
- `Framework.Loaded` is set unconditionally at the end of the pipeline. There is currently no `LoadErrors` surface to inspect partial failures.
- `OnLoaded` fires exactly once. Connecting to it after loading has already finished will never run the callback, use `:WaitUntilLoaded()` + inline code for that case.

## Related

- [shared.md](./shared.md) - the leaf require surface, use this from modules that are loaded by the framework
- [network.md](./network.md) - `GlobalEvents` / `LocalEvents` constructors and semantics
