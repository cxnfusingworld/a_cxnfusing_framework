# Util - Collections

## Overview

Four collection / lifecycle utilities that do the plumbing most classes end up rewriting: **Pool** (Pooler+ object pooling), **Trove** (bag of cleanup tasks), **Signal** (fast typed event), and **Promise** (async/A+ promises with cancellation). Pool is the heaviest of the four and gets the most space below. Trove, Signal, and Promise are vendored from well-known Luau libraries (Sleitnick/Trove, AlexanderLindholt/SignalPlus, fewkz+evaera/roblox-lua-promise) and are re-exported as-is.

## Requiring it

Everything lives under `Shared.Utilities`:

```lua
local Shared = require(game.ReplicatedStorage.Shared)
local Pool    = Shared.Utilities.Pool     -- the Pooler+ manager singleton
local Trove   = Shared.Utilities.Trove
local Signal  = Shared.Utilities.Signal   -- callable: Signal() returns a new signal
local Promise = Shared.Utilities.Promise
```

If you are inside a framework-loaded module, prefer `Framework.Utilities.<name>` for the same values.

---

## API reference

### Pool (Pooler+)

Central manager for named object pools. You register a pool once at startup with create / cleanup / destroy callbacks, then hit `:Get()` and `:Return()` on the returned `ObjectPool` in hot code paths. A Heartbeat loop auto-optimizes, adaptively preloads, and validates registered pools based on per-pool config.

#### `Pooler:CreatePool(name, createFn, cleanupFn?, destroyFn?, config?) -> ObjectPool`

Register a new pool. Duplicate names warn and return the existing pool.

- `name: string` - unique key.
- `createFn: () -> T` - builds a fresh object. Called on misses and during preload.
- `cleanupFn: (T) -> ()?` - runs on `Return` to reset the object for reuse.
- `destroyFn: (T) -> ()?` - runs when an object is permanently dropped (pool saturated, `Clear`, etc.). Defaults to `object:Destroy()` for Instances.
- `config: PoolConfig?`

`PoolConfig` fields (all optional):

| Field | Default | Purpose |
| --- | --- | --- |
| `maxSize` | `math.huge` | Cap on available + in-use. Exceeding on Return destroys the extra object. |
| `preloadCount` | `0` | Objects to create up front. |
| `enableAnalytics` | `false` | Track detailed stats (hit rate, rates, lifetimes). |
| `enableAdaptivePreload` | `false` | Heartbeat task grows the pool toward recent peak usage. |
| `enableValidation` | `false` | Periodic health checks. |
| `instanceConfig` | `nil` | For Roblox Instance pools: `{ template, resetProperties, cleanupConnections, safeParent }`. Auto-synthesizes a cleanupFn when `cleanupFn` is not provided. |

#### Manager methods

| Method | Returns | Notes |
| --- | --- | --- |
| `GetPool(name)` | `ObjectPool?` | Lookup by name. |
| `Get(name)` | `T?` | Shortcut for `GetPool(name):Get()`. Warns if pool missing. |
| `GetWithPriority(name, priority)` | `T?` | Forwards to pool. Priority: `"low" \| "normal" \| "high" \| "critical"`. |
| `Return(name, object)` | `()` | Warns if pool missing; destroys Instance as fallback. |
| `BatchGet({{ poolName, count }})` | `{ [string]: {T} }` | Bulk `GetBatch`. |
| `GetMemoryPressure()` | `"none" \| "moderate" \| "critical"` | Uses Luau `gcinfo()` + thresholds. |
| `OptimizeAllPools()` / `OptimizeAllPoolsWithGC()` | `()` | Shrink idle lists. |
| `EmergencyCleanup()` | `()` | Clamps every pool to ~1.2x live count, optimizes, restores maxSize. |
| `GetGlobalStats()` / `GetGlobalEnhancedStats()` | `{ [string]: Stats }` | Per-pool stats. |
| `GetSystemStatus()` | `SystemStatus` | Aggregate: pressure, totals, globalHitRate, poolsWithIssues. |
| `ValidateAllPools()` | `{ [string]: boolean }` | Per-pool validity. |
| `HealthCheck()` | `{ overallHealth, issues, warnings, recommendations }` | Full report. |
| `AutoRepairAllPools()` | `{ [string]: {string} }` | Applies auto-repair steps. |
| `DebugPrintAllPools()` / `GetPerformanceReport()` / `ExportPoolStates()` | `()` / `string` / `string` | Debug output. |

#### `ObjectPool` methods

Returned by `Pooler:CreatePool`. This is the handle you use in hot paths.

| Method | Returns | Notes |
| --- | --- | --- |
| `Get()` | `T` | Pops from `_available` if present, else creates (counted as a miss). Always succeeds, even past `maxSize`. |
| `GetSafe()` | `(T?, PoolError?)` | `pcall`-wrapped Get. |
| `GetWithPriority(priority)` | `T?` | Returns `nil` for `"low"` / `"normal"` when memory pressure is `"critical"`. |
| `GetBatch(count)` | `{ T }` | Multi-get. |
| `Return(object)` | `()` | Runs cleanupFn, pushes to `_available`. If at `maxSize` the object is destroyed instead. Warns on untracked / double returns. |
| `ReturnBatch(objects)` | `()` | |
| `ReturnAll()` | `()` | Returns every in-use object. |
| `Preload(count)` | `()` | Eagerly create + cleanup + park. |
| `AdaptivePreload()` | `()` | Manager-driven. Grows toward `peakUsage * PEAK_USAGE_RATIO`. Hard 10s cooldown per pool. |
| `Clear()` | `()` | Destroys every tracked object. |
| `Optimize()` / `OptimizeWithGC()` | `()` | Shrink idle list based on memory pressure. |
| `SetMaxSize(n)` | `()` | Resizes and trims overflow. |
| `GetName()` | `string` | |
| `GetStatus()` | `{ Available, InUse, Total }` | |
| `GetStats()` / `GetEnhancedStats()` | `PoolStats` / `EnhancedPoolStats` | Clone of the stats table. |
| `GetActiveObjects()` | `{ T }` | Snapshot of in-use keys. |
| `Validate()` | `boolean` | Health check. |
| `SerializeState()` | `string` | JSON snapshot (debug only). |

Each pool also exposes four signals you can `Connect` to: `OnGet(object)`, `OnReturn(object)`, `OnCreate(object)`, `OnOptimize()`. These fire on the hot path, so keep listeners cheap.

> Internal support modules (`Types`, `Utils`, `Analytics`, `Constants`, `Debug`, `SignalPlus`, `Validation`) live under `Util/Pool/`. They are implementation detail - use the Pooler + ObjectPool API above.

---

### Trove

A bag of cleanup tasks. Add anything (Instances, connections, functions, threads, tables with `Destroy`/`Disconnect`, promises, futures) and call `:Clean()` / `:Destroy()` once to tear it all down.

| Method | Purpose |
| --- | --- |
| `Trove.new()` | Construct. |
| `trove:Add(object, cleanupMethod?)` | Track object. Auto-detects cleanup: Instance->Destroy, RBXScriptConnection->Disconnect, function->call, thread->`task.cancel`, table->Destroy/Disconnect/destroy/disconnect. Override by passing `cleanupMethod` name. |
| `trove:Remove(object)` | Remove and clean up one object. |
| `trove:Extend()` | Construct a sub-Trove and add it. |
| `trove:Clone(instance)` | `Add(instance:Clone())`. |
| `trove:Construct(class, ...)` | `Add(class.new(...))` or `Add(class(...))`. |
| `trove:Connect(signal, fn)` | Connect to a signal and Add the connection. |
| `trove:Once(signal, fn)` | Connect once; auto-removes. |
| `trove:BindToRenderStep(name, priority, fn)` | Binds via RunService; unbinds on cleanup. |
| `trove:AddPromise(promise)` | Cancels promise on cleanup. Asserts roblox-lua-promise v4 shape. |
| `trove:AddFuture(future)` | Cancels future on cleanup. Asserts the forked Future shape. |
| `trove:AttachToInstance(instance)` | Cleans when the instance is removed from the game. Errors if instance is not already a descendant of `game`. |
| `trove:IsCleaning()` | `boolean`. |
| `trove:Clean()` / `trove:Destroy()` | Run every cleanup. `Destroy` is an alias. |

---

### Signal

Typed signal (SignalPlus). The module returns a callable: `Signal()` produces a new `Signal<Parameters...>`.

```lua
local onJump: Signal.Signal<Player, number> = Signal()
```

| Method | Purpose |
| --- | --- |
| `signal:Connect(callback)` | Returns a `Connection` with `Connected: boolean` and `Disconnect()`. |
| `signal:Once(callback)` | Fires at most once. |
| `signal:Wait()` | Yields until the next Fire; returns the fired args. |
| `signal:Fire(...)` | Invokes every connected callback on a pooled coroutine. |
| `signal:DisconnectAll()` | Drop all connections fast. |
| `signal:Destroy()` | DisconnectAll + unlink methods (the signal is unusable after). |

Setting a `Deferred` attribute on the Signal script flips `Fire` from `task.spawn` to `task.defer`. Callbacks added mid-Fire are not invoked in the same cycle (traversal is tail->head).

---

### Promise

Typed wrapper around evaera's roblox-lua-promise, re-exported through `init.luau` with chain-depth types `Promise1<T...>` through `Promise8<T...>` for type inference. Beyond ~8 chained calls the inferred type collapses to a loose `PromiseExhausted` (`andThen`/`catch`/etc still work, just untyped).

Statics exposed by the typed surface:

| Static | Returns | Purpose |
| --- | --- | --- |
| `Promise.new((resolve, reject, onCancel) -> ())` | `Promise1<T...>` | Wrap a callback-style API. |
| `Promise.resolve(...)` | `Promise1<T...>` | Already-resolved promise. |
| `Promise.delay(seconds)` | `Promise1<number>` | Resolves with elapsed time. |
| `Promise.fromEvent(event, predicate?)` | `Promise1<T...>` | First event fire matching predicate. |
| `Promise.all({promises})` | `Promise1<{T}>` | Resolve when all resolve; reject on any reject. |
| `Promise.try(callback, ...)` | `Promise1<T...>` | `pcall`-wrapped callback as a promise. |
| `Promise.Status` | `{ Started, Resolved, Rejected, Cancelled }` | Status enum. |

Runtime (`Main.luau`) also exposes statics not on the typed facade but usable in Luau: `Promise.defer`, `Promise.reject`, `Promise.race`, `Promise.some`, `Promise.any`, `Promise.allSettled`, `Promise.each`, `Promise.fold`, `Promise.is`, `Promise.promisify`, `Promise.retry`.

Instance methods: `andThen`, `andThenCall`, `andThenReturn`, `catch`, `tap`, `finally`, `finallyCall`, `finallyReturn`, `done`, `doneCall`, `doneReturn`, `now`, `timeout`, `cancel`, `getStatus`, `await`, `awaitStatus`, `expect`. Cancellation propagates both directions: a parent's cancel cascades to children, and when every consumer of a parent cancels, the parent auto-cancels too.

---

## Examples

### Pool a projectile

```lua
local Pool = Shared.Utilities.Pool

local bullets = Pool:CreatePool("Bullets",
    function() return ReplicatedStorage.BulletTemplate:Clone() end,
    function(b) b.Parent = nil end,
    function(b) b:Destroy() end,
    { maxSize = 200, preloadCount = 50, enableAnalytics = true }
)

local b = bullets:Get()
b.Parent = workspace
task.delay(2, function() bullets:Return(b) end)
```

### Own everything a class touches with a Trove

```lua
local trove = Trove.new()
local part = trove:Add(Instance.new("Part"))
trove:Connect(part.Touched, function(hit) print(hit.Name) end)
trove:BindToRenderStep("Hover", Enum.RenderPriority.Camera.Value, function(dt) end)
-- on destruction:
trove:Destroy()
```

### Broadcast a local event with Signal

```lua
local onRoundEnded: Signal.Signal<string, number> = Signal()
local conn = onRoundEnded:Connect(function(winnerTeam, score)
    print(winnerTeam, score)
end)
onRoundEnded:Fire("Red", 42)
conn:Disconnect()
```

### Wrap an async call with Promise

```lua
Promise.new(function(resolve, reject)
    local ok, data = pcall(dataStore.GetAsync, dataStore, key)
    if ok then resolve(data) else reject(data) end
end)
    :andThen(function(data) print("got", data) end)
    :catch(warn)
```

### Tie a Promise's lifetime to a Trove

```lua
local trove = Trove.new()
trove:AddPromise(Promise.delay(5)):andThen(function()
    print("five seconds later")
end)
-- if trove:Destroy() runs before 5s, the delay is cancelled
```

---

## Gotchas

- **Pool: `Get` always succeeds.** Past `maxSize` it still creates a new object (counted as a miss). Use `GetWithPriority("low")` to let memory pressure refuse you, or `GetSafe` for an `(object?, err?)` pair.
- **Pool: double Return warns but no-ops.** `Return` on an object not tracked as in-use prints a warning and drops it.
- **Pool: Heartbeat overhead.** Auto-optimize / adaptive preload / validation run on Heartbeat for every registered pool, gated by interval constants. Disable per-pool via the config flags.
- **Pool: `EmergencyCleanup` is disruptive.** Temporarily shrinks every pool. Do not call mid-burst.
- **Pool: signals fire on hot paths.** `OnGet`/`OnReturn`/`OnCreate` run inside `Get`/`Return`. Heavy listeners slow the pool.
- **Trove: no operations mid-clean.** `Add`/`Remove`/`Construct`/etc. error while `:Clean()` is running.
- **Trove: cleanup order is not guaranteed.**
- **Trove: `AttachToInstance` requires a rooted Instance.** It errors if the Instance is not already a descendant of `game`.
- **Trove: `AddPromise` / `AddFuture` are shape-asserted.** Non-matching objects throw.
- **Signal: `Destroy` strips the metatable.** Further calls on the signal error.
- **Signal: `Wait` does not queue past fires.** It yields until the next `Fire` only.
- **Signal: callbacks run on pooled threads.** Don't rely on thread identity across fires.
- **Promise: chain type inference caps at 8.** After that chains infer as `PromiseExhausted` - break the chain up if you need types.
- **Promise: unhandled rejections warn after one Heartbeat.** Attach `catch` or `done` to silence.
- **Promise: consumer cancellation cascades up.** If every child cancels, the parent auto-cancels. Keep a reference somewhere if that is not what you want.

---

## Related

- `docs/modules/util-signal-async.md` - higher-level signal/async helpers (ActionBuffer, Observer, StateMachine).
- `docs/modules/util-instance.md` - Instance lifecycle helpers (ConnectToAdded, ConnectToTag).
- `docs/modules/services-vfx.md` - VFX service, which uses Pool under the hood.
- `docs/reviews/shared-review.md` - known issues and follow-up fixes for these modules.
- Upstream repos: [Sleitnick/Trove](https://github.com/Sleitnick/RbxUtil), [AlexanderLindholt/SignalPlus](https://github.com/AlexanderLindholt/SignalPlus), [evaera/roblox-lua-promise](https://eryn.io/roblox-lua-promise/).
