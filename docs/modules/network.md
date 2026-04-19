# Network

Two tiny constructors for events. `Network.newGlobal` creates a networked event (server <-> client) backed by a shared `RemoteEvent` + `RemoteFunction` pair. `Network.newLocal` creates an in-process signal with no networking at all. Events are usually declared by name in `Shared/init.luau` under `GlobalEvents` / `LocalEvents`, then consumed anywhere as `Framework.GlobalEvents.X` or `Framework.LocalEvents.X`. Both constructors live under `Shared/Network/` and are re-exported through the framework aggregate.

## Overview

Two problems, two constructors:

- **Global events** (`Network.newGlobal`) cross the server/client boundary. Use these anytime the server needs to tell a client something, or a client needs to ask/tell the server. A single global event exposes BOTH fire-and-forget (`RemoteEvent`) and request/response (`RemoteFunction`) APIs on the same object.
- **Local events** (`Network.newLocal`) are in-process only. Use these to decouple systems running in the same context (e.g. one client system firing an event that other client systems listen for). No replication, just a fast callback list.

If you fire a local event on the server, the client will not see it, and vice versa. Reach for `newGlobal` whenever you need replication.

## Requiring it

You rarely call the constructors directly. The normal flow is:

1. Declare events by name in `Shared/init.luau`:

```lua
-- ReplicatedStorage/Shared/init.luau
local Network = require(script:WaitForChild("Network"))

return {
    GlobalEvents = {
        CoinsChanged = Network.newGlobal("CoinsChanged"),
        RequestPurchase = Network.newGlobal("RequestPurchase"),
    },
    LocalEvents = {
        PlayerHurt = Network.newLocal(),
    },
    -- ... rest of Shared
}
```

2. Consume them from user code via `Framework` (or `Shared`):

```lua
local Framework = require(game:GetService("ReplicatedStorage"):WaitForChild("Framework"))

Framework.GlobalEvents.CoinsChanged:Connect(function(newAmount)
    print("coins:", newAmount)
end)
```

If you need the constructors directly (rare), require the Network module:

```lua
local Network = require(game:GetService("ReplicatedStorage").Shared.Network)
local myEvent = Network.newGlobal("MyEvent")
local mySignal = Network.newLocal()
```

## API reference

### `Network.newLocal()`

Returns an in-process signal (SignalPlus under the hood). No networking.

Returned object methods:

- `signal:Connect(callback)` -> `Connection` - runs `callback(...)` every time the signal fires.
- `signal:Once(callback)` -> `Connection` - runs `callback(...)` the next time only, then auto-disconnects.
- `signal:Wait()` -> `...` - yields the current thread until the next fire; returns the fired args.
- `signal:Fire(...)` - invokes every connected callback with the given args.
- `signal:DisconnectAll()` - drops every connection in one sweep.
- `signal:Destroy()` - disconnects everything and unlinks the metatable. Further method calls will error.

`Connection` objects returned by `Connect` / `Once` have:

- `connection.Connected: boolean`
- `connection:Disconnect()`

Note: connections fire in reverse insertion order (linked-list tail-to-head). Do not rely on listener ordering.

### `Network.newGlobal(name)`

- `name: string` - becomes the `RemoteEvent` name under the shared `Remotes` folder. A paired `RemoteFunction` named `<name>_Invoke` is created alongside it.

Returns an event object whose available methods depend on the run context (server or client). Calling this factory for the same `name` from both sides reuses the same underlying instances.

**Server-only methods:**

- `event:FireClient(player, ...)` - fires the remote to one player. Silently returns if `player` is not a valid `Player` instance (watch for this when debugging).
- `event:FireAllClients(...)` - fires the remote to every connected client.
- `event:InvokeClient(player, ...)` -> `...` - request/response to a single client. Yields; can throw if the client errors. Silently returns on a bad player arg. Use sparingly.

**Client-only methods:**

- `event:FireServer(...)` - fires the remote to the server.
- `event:InvokeServer(...)` -> `...` - request/response to the server; yields for the reply.

**Shared methods (both contexts):**

- `event:Connect(callback)` -> connection - binds to `OnClientEvent` on the client, `OnServerEvent` on the server. Server callbacks receive `(player, ...)`; client callbacks receive `(...)`.
- `event:Once(callback)` -> connection - same as `Connect` but auto-disconnects before the callback runs the first time.
- `event:Wait()` -> `...` - yields until the next fire; returns the fired args.
- `event.OnInvoke = function(player, ...) return ... end` - **dot-call, not colon.** Sets the handler for `InvokeClient` / `InvokeServer`. Only one handler can exist at a time; calling this again overwrites the previous one.
- `event:DisconnectAll()` - disconnects every `Connect`/`Once` handed out by this object.
- `event:Destroy()` - disconnects everything, then destroys the underlying `RemoteEvent` and `RemoteFunction`. This tears the channel down for BOTH sides of the wire (see Gotchas).

The connection object returned by `:Connect`/`:Once` has `:Disconnect()` which removes it from the event's tracking table and disconnects the underlying Roblox connection.

## Examples

### Global event: server `FireClient` + client `Connect`

```lua
-- Shared/init.luau
GlobalEvents = {
    CoinsChanged = Network.newGlobal("CoinsChanged"),
}

-- server script
local Framework = require(game.ReplicatedStorage.Framework)
Framework.GlobalEvents.CoinsChanged:FireClient(player, 50)

-- local script
local Framework = require(game.ReplicatedStorage.Framework)
Framework.GlobalEvents.CoinsChanged:Connect(function(newAmount)
    print("I now have", newAmount, "coins")
end)
```

### Global event: client request/response with `OnInvoke`

```lua
-- server
Framework.GlobalEvents.RequestPurchase.OnInvoke = function(player, itemId)
    return tryPurchase(player, itemId) -- returns true/false
end

-- client
local ok = Framework.GlobalEvents.RequestPurchase:InvokeServer("potion")
print("purchased:", ok)
```

### Local event: same-context `Fire` + `Connect`

```lua
-- Shared/init.luau
LocalEvents = {
    PlayerHurt = Network.newLocal(),
}

-- anywhere in the same context
Framework.LocalEvents.PlayerHurt:Connect(function(dmg)
    print("ouch:", dmg)
end)
Framework.LocalEvents.PlayerHurt:Fire(25)
```

## Gotchas

- **The `Remotes` folder must exist under `Network`.** `SignalWrapper` resolves it via `WaitForChild` at require time. If the folder is missing, the entire module will yield forever and the framework never finishes loading. The repo ships with the folder; do not delete it.
- **Global events have BOTH a `RemoteEvent` and a `RemoteFunction`.** Even if you only use `Fire`, the `_Invoke` `RemoteFunction` is still created next to it. This is intentional so every global can also be invoked without a separate declaration.
- **`FireClient` / `InvokeClient` silently no-op on bad player args.** If nothing seems to fire, double-check you passed an actual `Player` instance.
- **`OnInvoke` is dot-call, not colon.** Write `event.OnInvoke = function(player, ...) ... end`, not `event:OnInvoke(...)`. The rest of the API uses colon style; this one is intentionally different because only one handler can be set.
- **`:Destroy` kills both sides of the wire.** Since server and client share the same underlying `RemoteEvent`/`RemoteFunction` instances, destroying on one side tears the channel down for everyone. Do not call `Destroy` unless you really want the event gone.
- **Local events do not replicate.** Firing on the server will not reach clients, and vice versa. If you need cross-boundary delivery, use `newGlobal`.
- **Client objects lack `FireClient`/`FireAllClients`; server objects lack `FireServer`/`InvokeServer`.** The split is static per context. Calling the wrong one errors immediately rather than silently doing nothing.

## Related

- [framework.md](./framework.md) - the loader that exposes `Framework.GlobalEvents` / `Framework.LocalEvents` and runs `:Init`/`:Start` on tag-loaded modules.
- [shared.md](./shared.md) - where you declare event names so they flow up into the framework aggregate.
