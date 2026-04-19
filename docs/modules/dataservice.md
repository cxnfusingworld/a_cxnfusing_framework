# DataService

DataService is the framework's per-player persistence layer. It wraps the vendored ProfileStore for saving and loading, keeps an in-memory observable mirror of each player's data, and automatically replicates mutations to the owning client over a small RPC helper called Networker. You require one module and get two halves: `.server` owns the real data and does the writes, `.client` holds a local mirror that receives replicated changes and exposes the same read API plus change signals you can bind UI to.

## Overview

- Built on a vendored copy of `ProfileStore` (sitting next to the module) for the actual DataStore-backed saving, session locking, template reconciliation, and cross-server messaging.
- Split into two halves by run context. `DataService.server` is the live module on the server and an empty stub on the client; `DataService.client` is the reverse. Grab the half that matches your script context.
- In-memory reads/writes go through an observable `Data` object (per player on the server, one total on the client) that exposes `get`/`set`/`update`/`arrayInsert`/`arrayRemove` plus four change-signal getters.
- Server-side mutations auto-replicate to the owning client unless you opt out with `dontReplicate = true`. Client-side mutations are local-only.

## Requiring it

```lua
-- server
local DataService = require(path.to.DataService).server
DataService:init({
    template = require(path.to.DataTemplate),
    profileStoreIndex = "Live",
})

-- client
local DataService = require(path.to.DataService).client
DataService:init()
print(DataService:get({ "currency" }))
```

`:init(...)` must be called before any other method on either half. The client `:init` yields until the initial snapshot arrives from the server.

Paths are either a single string key (`"currency"`) or an array of string/number keys for nested walks (`{ "tutorial", "step" }`, `{ "inventory", 3 }`).

## API reference

### Server (`DataService.server`)

#### `init(options)`
Initializes the server service. Must be called exactly once. Throws if called twice.

`options` fields:
- `template: DataTemplate` - the default data shape. New keys are reconciled into existing profiles on join.
- `profileStoreIndex: string?` - ProfileStore index name (default: `"Default"`).
- `profileStoreDataPrefix: string?` - key prefix for profiles (default: `"PLAYER_"`).
- `useMock: boolean?` - use ProfileStore's Mock store for testing (no real saves).
- `viewedUserId: number?` - read-only view of another user's save.
- `overridenUserId: number?` - play on another user's save (writes as them).
- `dontSave: boolean?` - load via `GetAsync` so nothing saves.
- `resetData: boolean?` - wipe the profile on join.

#### Reads and writes (all take a `Player`)
- `get(player, path?) -> any` - value at `path`, or the full data table when `path` is nil.
- `set(player, path, value, dontReplicate?)` - set at `path`.
- `update(player, path, callback, dontReplicate?) -> any` - callback receives the current value, returns the new one; returns the new value.
- `arrayInsert(player, path, value, index?, dontReplicate?)` - mimics `table.insert`.
- `arrayRemove(player, path, index, dontReplicate?) -> any` - mimics `table.remove`; returns removed value.

Pass `dontReplicate = true` to mutate server-only; you're then responsible for any client sync.

#### Change signals (server-side, per player)
- `getChangedSignal(player, path) -> Signal` - fires `(new, prev)` when the value at `path` changes.
- `getIndexChangedSignal(player, path) -> Signal` - fires `(index, new, prev)` when a child of a table at `path` changes.
- `getArrayInsertedSignal(player, path) -> Signal` - fires `(index, value)` after inserts.
- `getArrayRemovedSignal(player, path) -> Signal` - fires `(index, value)` after removes.

#### Lifecycle / hooks
- `onPlayerInit(player, data)` - overridable. Runs after profile load but before the client receives the snapshot. Safe to mutate `data` directly (no signals yet).
- `addPlayerRemovingCallback(fn) -> disconnect` - register a cleanup/save hook `(player, data) -> ()` to run when a player leaves. Returns a function that removes the hook.
- `waitForData(player) -> Data` - yields until the player's in-memory Data is ready.
- `hasProfile(player) -> boolean`
- `getProfile(player) -> Profile` - throws if the player has no profile.
- `resetData(player)` - removes the profile from the store and kicks the player.
- `asyncGetProfile(userId) -> Profile?` - `GetAsync` an arbitrary user's profile (read-only).

#### Cross-server messaging
- `sendGlobalMessage(key, userId, data) -> boolean` - send via ProfileStore's `MessageAsync`.
- `addGlobalCallback(key, cb)` - register `(player, data) -> boolean?` to handle messages with that `key`. Returning `false` explicitly prevents ack; any other return acks. Keys can only be registered once; messages that arrive before a callback is registered wait for it.

### Client (`DataService.client`)

#### `init(data?)`
Yields until the server has sent the initial snapshot. You typically call it with no arguments.

#### Reads
- `get(path?) -> any` - yields until data is initialized, then reads from the local mirror.
- `fetch(path?, dontSet?) -> any` - round-trip to the server for a fresh value. Sets the local mirror unless `dontSet` is true. Usually only needed if you think you've desynced.
- `waitForData() -> Data` - yields until init finishes.

#### Writes (local-only, do not replicate back to the server)
- `set(path, value)`
- `update(path, callback) -> any`
- `arrayInsert(path, value, index?)`
- `arrayRemove(path, index) -> any`

Use these for UI predictions and purely client-side state (settings, animations). The authoritative copy lives on the server.

#### Change signals
Same four getters as the server, without the `player` argument: `getChangedSignal(path)`, `getIndexChangedSignal(path)`, `getArrayInsertedSignal(path)`, `getArrayRemovedSignal(path)`. These fire for both locally-driven writes and server-replicated writes.

### Networker (brief)

Networker is the small RPC helper DataService uses internally for replication. It pairs a RemoteEvent and RemoteFunction under a tag and exposes `fire`/`fireAll`/`fireAllExcept`/`fireRecipients` on the server and `fire`/`fetch` on the client. The server controls what client methods are allowed via a whitelist passed at construction. DataService whitelists only `get` for client-to-server calls (that's what backs `:fetch`); every other replicated action is server-push. You can reuse Networker for other service-style replication, but most gameplay networking should go through `Shared/Network` instead.

## Examples

### Server: load template, read/write, react to changes

```lua
local Players = game:GetService("Players")
local DataService = require(path.to.DataService).server

DataService:init({
    template = {
        currency = 0,
        level = 1,
        inventory = {},
        tutorial = { completed = false, step = 0 },
    },
    profileStoreIndex = "Live",
})

function DataService:onPlayerInit(player, data)
    data.sessionJoinTime = os.time()
end

Players.PlayerAdded:Connect(function(player)
    DataService:waitForData(player)

    local currency = DataService:get(player, { "currency" })
    DataService:set(player, { "currency" }, currency + 10)

    DataService:update(player, { "level" }, function(lvl)
        return lvl + 1
    end)

    DataService:arrayInsert(player, { "inventory" }, "StarterSword")

    DataService:getChangedSignal(player, { "currency" }):Connect(function(new, prev)
        print(player.Name, "currency:", prev, "->", new)
    end)
end)
```

### Client: read data and bind UI to changes

```lua
local DataService = require(path.to.DataService).client
DataService:init()

local label = script.Parent.CurrencyLabel
label.Text = tostring(DataService:get({ "currency" }))

DataService:getChangedSignal({ "currency" }):Connect(function(new)
    label.Text = tostring(new)
end)

-- Purely local UI state, never sent to the server:
DataService:set({ "ui", "lastPanel" }, "Shop")
```

## Gotchas

- **ProfileStore is vendored** under the DataService folder. Don't edit it; pull fresh from upstream if you need updates.
- **Data must be JSON-safe.** No `Vector3`, `CFrame`, `Instance`, or other non-serializable values. ProfileStore won't save them.
- **Session lock.** The normal load path uses `StartSessionAsync`, which holds a session lock. A profile can only be loaded by one server at a time. If the player already has a live session elsewhere (another server or a stale lock from a crash), load can take a while as ProfileStore steals the lock. If a session ends mid-play, the player is kicked (`Profile session end - Please rejoin`).
- **`:init(...)` must run before anything else**, exactly once on the server. Calling other methods first produces nil errors (there's no per-player Data yet). Re-calling `:init` asserts.
- **Half-of-context silent stubs.** `DataService.server` required on the client returns an empty table. `DataService.client` required on the server returns an empty table. Every method will be nil with no loud error. Always pick the half that matches your script context.
- **Client writes don't replicate** back to the server. `DataService.client:set` and friends mutate only the local mirror. Use them for UI predictions and local-only state; use server-side writes for anything that needs to persist.
- **Template reconciliation is one-way.** New keys added to the template appear on existing profiles on next load. Removed or renamed keys won't migrate automatically; handle that yourself in `onPlayerInit` or a dedicated migration step.
- **Studio behavior.** There's no automatic Studio detection built into DataService. If you want no-save runs in Studio, pass `useMock = true` (uses ProfileStore's Mock store) or `dontSave = true` (uses `GetAsync`). Live DataStore keys will be used otherwise, even in Studio.
- **`onPlayerInit` runs before the client snapshot is sent.** Direct mutation of `data` there is safe and won't fire change signals (none exist yet). Once the snapshot is out and the Data object is built, use `set`/`update`/`arrayInsert`/`arrayRemove` so signals fire and replication happens.
- **Signals are weakly held until connected.** `getChangedSignal` and friends hand back a signal that's kept alive while at least one connection exists, and can be GC'd once all connections disconnect. Don't stash a returned signal long-term without a live connection.
- **Action ids are part of the wire protocol.** `DataServiceUtils.enums.actions` maps mutation names to numeric ids used over the wire. Append new ops; don't renumber existing ones, or you'll break mismatched client/server versions.

## Unclear

- There's no automatic "am I in Studio?" branch in `:init`. Users need to set `useMock`/`dontSave` themselves; the source doesn't document a recommended Studio preset, so the exact intended Studio workflow is left to the caller.
- `update`'s replication fires a `set` with the new value after the local mutation, so the server's change signal fires before replication goes out. Whether callers should treat this ordering as guaranteed isn't stated in the source.
- `addGlobalCallback` asserts that no callback exists for a given key, so callbacks can't be replaced or removed at runtime - whether that's intentional isn't documented.

## Related

- `ProfileStore` (vendored inside this folder) - underlying persistence, session locking, cross-server messaging.
- `Networker` (inside this folder) - the server/client RPC helper DataService uses for replication. Reusable for other service-style replication.
- `Data.luau` (inside this folder) - the observable table implementation backing both halves. Usually consumed indirectly through DataService.
- `Shared/Network` - general-purpose Global/Local events for gameplay networking that isn't service-mirror replication.
- `Shared/Components/Util/Signal.luau` - the framework's general-purpose Signal. DataService vendors its own copy under `DataService/Signal` for internal use; prefer the Util one for your own code to avoid two copies in the module graph.
