# Info

Live "blackboard" tables that describe the current runtime context. `Info.PlayerState` tracks the local player's character on the client; `Info.GameState` is a shared scratchpad for game-wide values. Nothing in here does networking, persistence, or magic replication: they are plain Lua tables that multiple systems read and write.

## Overview

`Info` is a tiny registry under `Shared/Components/Info/` that returns two sub-tables:

- **`PlayerState`** - client-only snapshot of the local player's `Character`, `HumanoidRootPart`, and `Humanoid`. Kept in sync across respawns. Also exposes a framework `Signal` (`CharacterAdded`) that fires after those fields have been refreshed.
- **`GameState`** - empty-by-design shared table. Systems drop game-wide values on it (round number, match phase, global modifiers, etc.) so other systems can read them without a dedicated service.

Think of these as read-mostly shared state. Any writer mutates the same table every reader sees.

## Requiring it

```luau
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))
local Info = Shared.Info

-- Or, from inside an Info-consuming module:
local Info = require(Shared.Components.Info)

local PlayerState = Info.PlayerState
local GameState = Info.GameState
```

If you are writing user-level code (a `Script` or `LocalScript`), you can also reach these through the `Framework` surface: `Framework.Info.PlayerState`, `Framework.Info.GameState`. The values are identical.

## API reference

### `Info.PlayerState` (client only)

Populated at require time from `Players.LocalPlayer`. On the server, requiring this module returns the value `true` as an early bail-out - see [Gotchas](#gotchas).

Fields:

| Field | Type | Notes |
|---|---|---|
| `Character` | `Model` | The local player's current character. Re-assigned on respawn. |
| `HRP` | `BasePart` | `Character.HumanoidRootPart`. Re-waited-for on respawn. |
| `Humanoid` | `Humanoid` | `Character.Humanoid`. Re-waited-for on respawn. |
| `Connections` | `{ [string]: RBXScriptConnection }` | Reserved for future use. Currently always empty. |

Events:

| Event | Signature | Notes |
|---|---|---|
| `CharacterAdded` | `Signal<Model>` | Framework `Signal` (not `Player.CharacterAdded`). Fires **after** `Character`, `HRP`, and `Humanoid` have been updated, so listeners always observe a fully populated state. |

### `Info.GameState`

Empty table by design. Read and write arbitrary keys on it:

```luau
Info.GameState.Phase = "Intermission"
Info.GameState.CurrentRound = 3
```

No predefined fields. No events. No replication. Exists in both client and server contexts, but each context has its own copy; setting a value on the server does not update the client, and vice versa. Use `Network` if you need it on both sides.

## Examples

### Read the local character from anywhere (client)

```luau
local PlayerState = require(Shared.Components.Info).PlayerState

local humanoid = PlayerState.Humanoid
print("health:", humanoid.Health)
```

### React to respawns without wiring your own hook

```luau
local PlayerState = require(Shared.Components.Info).PlayerState

PlayerState.CharacterAdded:Connect(function(character)
    print("respawned as", character.Name)
    -- PlayerState.HRP / .Humanoid are already the new ones here.
end)
```

### Share a game-wide phase across systems

```luau
-- In your round controller:
local GameState = require(Shared.Components.Info).GameState
GameState.Phase = "Playing"
GameState.RoundStart = os.clock()

-- Anywhere else in the same context:
if GameState.Phase == "Playing" then
    local elapsed = os.clock() - GameState.RoundStart
    -- ...
end
```

### Guard cross-context code

```luau
local RunService = game:GetService("RunService")
local Info = require(Shared.Components.Info)

if RunService:IsClient() then
    local hrp = Info.PlayerState.HRP
    -- safe to use
end
```

## Gotchas

- **`PlayerState` is client-only.** On the server, `PlayerState.luau` returns `true` early (there is no `LocalPlayer`). That means `Info.PlayerState.Character` on the server is equivalent to indexing a boolean and will error. Guard with `RunService:IsClient()` if your code can run in both contexts.
- **`Info.PlayerState.CharacterAdded` is not `Player.CharacterAdded`.** It is a framework `Signal` that fires *after* `HRP` and `Humanoid` have been refreshed. Use it when you want to observe a coherent state; use `LocalPlayer.CharacterAdded` directly if you specifically need the Roblox event first.
- **`Connections` is a reserved empty table.** Nothing populates it today. Do not rely on it.
- **`GameState` is just a table.** There is no replication, no signal-on-change, and no validation. If two systems write the same key, the last writer wins. Treat it as a read-mostly blackboard and own each key from exactly one place.
- **Per-context copies.** `GameState` on the server and `GameState` on the client are separate tables. Writing on one side does not propagate. Use `Network` (see [framework.md](framework.md)) for cross-context sync.
- **Mutations are visible to every reader.** Because these are shared tables, anything you stick on `GameState` or (less commonly) `PlayerState` is immediately observed by every other consumer. Avoid storing per-system scratch data here.
- **Shared's `table.freeze` pass skips these.** The aggregator freezes most sub-tables, but the signal-detect rule (anything with `Fire` + `Connect`) keeps `PlayerState` mutable, and `GameState` currently slips through the same path. Do not rely on immutability.

## Related

- [framework.md](framework.md) - the loader that exposes `Info` on `Framework.Info`, plus `Framework:IsServer()` / `:IsClient()` helpers useful when guarding `PlayerState` access.
- [shared.md](shared.md) - the leaf require surface. `Info` is reached via `Shared.Info` from any module the framework loads.
