# Util - Input

## Overview

Three small helpers for wiring up input and instance-lifecycle reactions without re-writing the same boilerplate. `ConnectToAdded` handles "run this for every child/descendant, now and in the future" on any Instance. `ConnectToTag` is the same idea for CollectionService tags. `ActionBuffer` is a tiny timestamp-based input buffer so things like "press jump right before landing" feel lenient. They live alongside each other because they are the glue most input/gameplay code needs on top of Roblox's raw signals.

## Requiring it

All three are part of the Shared utility surface, so you can reach them via either entry point:

```lua
-- from user scripts (Script / LocalScript)
local Framework = require(game.ReplicatedStorage.Framework)
local ConnectToAdded = Framework.Utilities.ConnectToAdded
local ConnectToTag = Framework.Utilities.ConnectToTag
local ActionBuffer = Framework.Utilities.ActionBuffer

-- from framework-loaded ModuleScripts (avoid circular requires)
local Shared = require(game.ReplicatedStorage.Shared)
local ConnectToAdded = Shared.Utilities.ConnectToAdded
```

You can also require the files directly if you prefer:

```
ReplicatedStorage/Shared/Components/Util/ConnectToAdded.luau
ReplicatedStorage/Shared/Components/Util/ConnectToTag.luau
ReplicatedStorage/Shared/Components/Util/ActionBuffer.luau
```

## API reference

### ConnectToAdded

```lua
ConnectToAdded(obj: Instance, callback: (Instance) -> (), useDescendants: boolean?) -> RBXScriptConnection?
```

Fires `callback` once for every current child of `obj`, then connects to `ChildAdded` so `callback` keeps firing for every future child. Pass `useDescendants = true` to walk and listen to the whole subtree (`GetDescendants` + `DescendantAdded`) instead of just direct children.

- Returns the `ChildAdded`/`DescendantAdded` connection so you can `:Disconnect()` future firings.
- Returns `nil` silently if `obj` or `callback` is nil.
- The initial sweep over existing children/descendants fires synchronously before the function returns.

### ConnectToTag

```lua
ConnectToTag(tag: string, callback: (Instance) -> ()) -> RBXScriptConnection?
```

Fires `callback` once for every instance currently tagged with `tag` via CollectionService, then connects to `GetInstanceAddedSignal(tag)` so `callback` keeps firing for every future tagged instance.

- Returns the added-signal connection so you can `:Disconnect()` future firings.
- Returns `nil` silently if `tag` or `callback` is nil.
- Add-only. No removal hook. Wire `CollectionService:GetInstanceRemovedSignal(tag)` yourself if you need cleanup.

### ActionBuffer

Module-level singleton. Storage is shared across all requirers, so namespace your `actionName` strings if multiple systems buffer through the same instance.

```lua
ActionBuffer:Record(actionName: string)
```
Stamps `actionName` with the current `os.clock()`.

```lua
ActionBuffer:IsActive(actionName: string, customWindow: number?) -> boolean
```
Returns `true` if a `Record` for `actionName` happened within the last `customWindow` seconds (default `0.2`). Consuming on a hit: a successful call resets the stored timestamp so a single `Record` cannot satisfy two `IsActive` checks.

```lua
ActionBuffer:Clear(actionName: string)
```
Wipes the stored entry for `actionName`.

## Examples

### Wire up every current and future child of a folder

```lua
local ConnectToAdded = Framework.Utilities.ConnectToAdded

local conn = ConnectToAdded(workspace.Pickups, function(pickup)
    print("got pickup:", pickup.Name)
end)

-- later, when your system tears down:
conn:Disconnect()
```

### Apply a setting to a whole model's subtree

```lua
ConnectToAdded(model, function(desc)
    if desc:IsA("BasePart") then
        desc.CanCollide = false
    end
end, true) -- useDescendants
```

### React to CollectionService-tagged instances

```lua
local ConnectToTag = Framework.Utilities.ConnectToTag

ConnectToTag("Enemy", function(enemy)
    -- runs immediately for currently-tagged enemies,
    -- and again when new ones are tagged
    setupEnemy(enemy)
end)
```

### Forgive an early jump press

```lua
local ActionBuffer = Framework.Utilities.ActionBuffer
local UIS = game:GetService("UserInputService")

UIS.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.Space then
        ActionBuffer:Record("Jump")
    end
end)

-- in your movement step:
if landed and ActionBuffer:IsActive("Jump") then
    character:Jump()
end
```

### Custom window for a dash

```lua
if ActionBuffer:IsActive("Dash", 0.35) then
    doDash()
end
```

## Gotchas

- **Initial sweep fires before you get the connection.** Both `ConnectToAdded` and `ConnectToTag` iterate existing children/tagged instances synchronously before returning the connection. If your callback throws during that first pass, it throws from inside the call to the helper, not from the returned connection. Disconnecting the returned connection does not "undo" the initial sweep's effects.
- **Nil args silently return nil.** Passing a nil `obj`, `callback`, or `tag` to `ConnectToAdded`/`ConnectToTag` gets you back `nil` with no `warn` and no `error`. Check the return value if you want to be sure the connection happened.
- **`useDescendants = true` walks the whole subtree.** On large models the initial `GetDescendants` pass can be heavy. Scope narrowly when you can.
- **`ConnectToTag` is add-only.** There is no removal hook. For symmetric behavior, wire `CollectionService:GetInstanceRemovedSignal(tag)` yourself next to the `ConnectToTag` call.
- **`ActionBuffer:IsActive` consumes on hit.** A successful check resets the stored timestamp so future checks return `false` until the next `Record`. Treat `IsActive` as "try consume", not "peek". If you need to peek without consuming, `Clear` afterwards is not a substitute because the consume has already fired.
- **`ActionBuffer` storage is module-global.** Every requirer reads and writes the same `storedActions` table. Namespace `actionName` strings (`"Combat.Jump"`, `"UI.Confirm"`, etc.) if multiple independent systems use the buffer.
- **Default window is 200ms.** Pass `customWindow` per-call when you want tighter or looser leniency; there is no global override.

## Related

- `Shared/Components/Util/Trove.luau` - scoped cleanup for connections returned by these helpers.
- `Shared/Components/Util/Signal.luau` - in-process signal implementation if you want `ConnectToTag`-style add signals of your own.
- `Shared/Network/` - remote/local event wrappers for cross-boundary input.
- `Modules/Client/GameFeel/` - higher-level input feel helpers (CoyoteTime, JumpBuffer, ApexGravity, LandingFX) that compose on top of things like `ActionBuffer`.
- Roblox API: `CollectionService:GetTagged`, `CollectionService:GetInstanceAddedSignal`, `CollectionService:GetInstanceRemovedSignal`, `Instance.ChildAdded`, `Instance.DescendantAdded`.
