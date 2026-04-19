# Shared

## Overview

`Shared` is the leaf require surface of the framework. It aggregates every `Components/*` sub-table (Utilities, Functions, Info, Services), the `Config` flags, and every registered Global/Local network event into a single frozen table. It has zero framework-internal dependencies, so it is safe to require from modules that are themselves loaded by the framework without causing circular requires.

Use `Shared` from ModuleScripts that run inside the framework load pipeline (tag-loaded modules, sub-services, utilities that depend on other utilities). Use `Framework` from regular user `Script` / `LocalScript` code where you also want helpers like `:WaitUntilLoaded`, `:Log`, `:SafeCall`, and `:IsServer`. Both surfaces expose the same data fields, so downstream code reads the same values either way.

## Requiring it

From inside a ModuleScript that the framework loads (for example, anything tagged `LoadModule`):

```lua
-- MyTaggedModule.luau
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))

local Trove = Shared.Utilities.Trove
local FastTween = Shared.Functions.FastTween

local module = {}

function module:Init()
    self._trove = Trove.new()
end

function module:Start()
    Shared.GlobalEvents.VFX:FireAll("Explosion", Vector3.new(0, 10, 0))
end

return module
```

## API reference

`Shared` returns one frozen table with the fields below. Sub-tables are also frozen; see the linked module docs for the contents of each.

| Field | Type | What it is |
| --- | --- | --- |
| `GlobalEvents` | table | Named RemoteEvent wrappers created via `Network.newGlobal(name)`. Server and client share one name. Ships with `VFX`. See `network.md`. |
| `LocalEvents` | table | Named in-process signals created via `Network.newLocal()`. Same-process only, no replication. Ships with `VFX`. See `network.md`. |
| `Services` | table | Both Roblox services (Players, ReplicatedStorage, ...) and framework services (`VFX`, `DataService`). See `framework.md`. |
| `Assets` | Folder | `ReplicatedStorage.Assets`. Populated via `WaitForChild`. |
| `Data` | Folder | `Shared.Data` sibling folder for static data ModuleScripts. Populated via `WaitForChild`. |
| `Utilities` | table | The `Components/Util` grab-bag: `Trove`, `Signal`, `Spring`, `Bezier`, `Pool`, `Promise`, `CameraShaker`, `StateMachine`, `Ragdoll`, `ViewCheck`, `TimeFormatter`, `ExtendedLibraries`, and more. Bigger or stateful helpers. |
| `Debris` | Service | `game:GetService("Debris")`, surfaced at the top level for convenience. |
| `Functions` | table | Small pure-ish one-shot helpers: `FastTween`, `FormatNumber`, `AddCommas`, `GenerateUID`. |
| `Info` | table | Shared state tables (`PlayerState`, `GameState`) used to pass context between modules on the same side. |
| `Config` | table | Flags from `Shared/Config.luau` (`LogsEnabled`, `WarningLogsEnabled`, `ErrorLogsEnabled`, `LogModuleLoadTimes`). |
| `Version` | string | Framework version string. Currently `"1.2.0 - Open Source, Added Utilities"`. |

For the contents of each sub-table, see `framework.md` (overall surface and helpers) and `network.md` (Global/Local event APIs).

### `Config` fields

Declared in `Shared/Config.luau`:

- `LogsEnabled: boolean` - gate normal prints from the framework and your own `Framework:Log` calls.
- `WarningLogsEnabled: boolean` - gate warn-level output.
- `ErrorLogsEnabled: boolean` - gate error-level output.
- `LogModuleLoadTimes: boolean` - when true, `ModuleLoader` prints how long each module's `:Init` / `:Start` took.

Add your own game-wide flags under the `Game Config` section of `Config.luau`. The table is frozen after require, so edit the source file and restart to change values.

## Examples

### Read config and a utility from a tag-loaded module

```lua
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))

local module = {}

function module:Start()
    if Shared.Config.LogsEnabled then
        print("[MyModule] starting, framework version", Shared.Version)
    end

    local trove = Shared.Utilities.Trove.new()
    trove:Add(Shared.LocalEvents.VFX:Connect(function(name, ...)
        print("local VFX fired:", name, ...)
    end))

    self._trove = trove
end

function module:Stop()
    if self._trove then
        self._trove:Destroy()
    end
end

return module
```

### Fire a Global event from a server module

```lua
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))

local module = {}

function module:Start()
    task.delay(2, function()
        Shared.GlobalEvents.VFX:FireAll("Sparkle", Vector3.new(0, 5, 0))
    end)
end

return module
```

### Format a number with Functions

```lua
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))

local formatted = Shared.Functions.FormatNumber(1_250_000)
local padded = Shared.Functions.AddCommas(1250000)

print(formatted, padded)
```

## Gotchas

- `Shared` freezes the outer table and every inner sub-table at the end of `init.luau`. You cannot add events, utilities, or config fields at runtime. Edit the source and restart.
- Network event tables (things with both `Fire` and `Connect`) still need to mutate their connection lists internally. In practice those entries are objects returned from `Network.*` constructors, not plain tables, so freezing the outer registration table is safe.
- `Assets = rs:WaitForChild("Assets")` and `Data = script:WaitForChild("Data")` will yield indefinitely if those children do not exist. Create a `ReplicatedStorage.Assets` folder and a `Shared.Data` folder, or delete those lines from `Shared/init.luau`.
- To register a new event, add a line under `GlobalEvents` or `LocalEvents` inside `Shared/init.luau`. That line is literally what makes `Framework.GlobalEvents.YourThing` exist.
- From a user `Script` / `LocalScript`, prefer `require(ReplicatedStorage.Framework)` so you get `:WaitUntilLoaded` and the other helpers. Use `Shared` from inside framework-loaded ModuleScripts to avoid circular requires.

## Related

- `framework.md` - the full loader surface, helpers (`:WaitUntilLoaded`, `:Log`, `:SafeCall`, `:Preload`, `:IsServer`, `:IsClient`, `:IsStudio`), and module loading rules.
- `network.md` - `Network.newGlobal` / `Network.newLocal` constructors and the `Fire` / `FireAll` / `Connect` / `Once` / `Wait` / invoke API on the returned objects.
