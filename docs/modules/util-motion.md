# Util - Motion

## Overview

Four small, focused utilities for motion, state, and reactivity. `Bezier` evaluates smooth curves through a list of `Vector3` waypoints (two flavors: pull-toward and pass-through). `Spring` is a physics-style, lazily-evaluated spring for any n-lerpable value (`number`, `Vector2`, `Vector3`). `StateMachine` is a tiny finite state machine that loads states from a `States` sibling folder of ModuleScripts. `Observer` wraps a table in a userdata proxy that fires a callback on real field changes. Together these cover the common "something has to move, switch mode, or react to a value" jobs without reaching for heavier frameworks.

## Requiring it

All four live under `Shared.Utilities` once the framework is loaded:

```lua
local Framework = require(game:GetService("ReplicatedStorage"):WaitForChild("Framework"))
local Bezier = Framework.Utilities.Bezier
local Spring = Framework.Utilities.Spring
local StateMachine = Framework.Utilities.StateMachine
local Observer = Framework.Utilities.Observer
```

From a framework-loaded ModuleScript, use `Shared` instead of `Framework` to avoid circular requires.

## API reference

### Bezier

Pure functions. No constructor, no state.

- `Bezier.GetMagnetPath(t: number, points: {Vector3}) -> Vector3`
  - de Casteljau evaluation. Returns a point that starts at `points[1]` (t=0) and ends at `points[#points]` (t=1). Inner points pull the curve toward themselves but the curve does NOT pass through them.
  - Short-circuits: returns `points[1]` if fewer than 2 points; a plain `Lerp` if exactly 2.
- `Bezier.GetSmoothSpline(t: number, points: {Vector3}) -> Vector3`
  - Catmull-Rom spline over the point list. The curve passes through every input point. `t` is mapped across segments so t=0 hits `points[1]` and t=1 hits `points[#points]`.

### Spring

Lazily-evaluated physics spring. Reading `Position`/`Velocity` re-integrates from the last snapshot each access, so you drive it by reading it (typically from a `RenderStepped` handler).

- `Spring.new<T>(initial: T, damping: number?, speed: number?, clock: (() -> number)?) -> Spring<T>`
  - `T` may be `number`, `Vector2`, or `Vector3`. `damping` defaults to `1` (critically damped), `speed` defaults to `1`, `clock` defaults to `os.clock`.
- `spring:Reset(target: T?)`
  - Snaps position and target to the given value (or the spring's initial value if omitted) and zeroes velocity.
- `spring:Impulse(velocity: T)`
  - Adds the given velocity to the current velocity. Useful for shakes/recoil.
- `spring:TimeSkip(delta: number)`
  - Advances the simulation forward by `delta` seconds without waiting real time.

Properties (read/write, with short aliases):

| Full | Alias | Notes |
| --- | --- | --- |
| `Position` | `p` | Current computed position. Writing sets position and preserves velocity. |
| `Velocity` | `v` | Current computed velocity. Writing sets velocity and preserves position. |
| `Target` | `t` | Where the spring is heading. Writing re-snapshots state first so motion stays continuous. |
| `Damping` | `d` | `<1` underdamped (bouncy), `=1` critical, `>1` overdamped. |
| `Speed` | `s` | Must be `>= 0`. Negative writes are clamped to `0`. |
| `Clock` | - | Swap the time source. |

Writing to an unknown field errors with `"<name>" is not a valid member of Spring.`

### StateMachine

Tiny finite-state machine. States are ModuleScripts under `script.States` (a sibling folder of `StateMachine.luau`), each returning a table with any of the optional hooks:

- `CanEnter(previousState, ...) -> boolean` - veto entry.
- `OnEnter(previousState, ...) -> any`
- `OnExit(...) -> any`
- `Update(...) -> any`

Constructor and methods:

- `StateMachine:Init()`
  - Call once at boot. Loads every ModuleScript child under `script.States` into the module-level state registry.
- `StateMachine.new(startingState: string?, ...) -> StateMachine`
  - Creates a new instance. If `startingState` is given, immediately calls `:TryEnter(startingState, ...)`.
- `sm:TryEnter(newState: string, ...) -> any`
  - Exits the current state (if any) and enters `newState`. Silently no-ops if the state isn't registered, is already current, or its `CanEnter` returns false. Returns whatever the new state's `OnEnter` returned.
- `sm:Exit(switchTo: string?, ...) -> any`
  - Exits the current state. With `switchTo`, delegates to `:TryEnter(switchTo, ...)` instead. Returns the value from `OnExit`.
- `sm:Update(...) -> any`
  - Calls `Update` on the current state (if defined) and fires `OnUpdate`.
- `sm:GetState() -> string?`
- `sm:GetPreviousState() -> string?`

Signals exposed on every instance:

- `sm.OnEnter` - fires `(currentState, previousState, ...)` after entry.
- `sm.OnExit` - fires `(previousState, ...)` after exit.
- `sm.OnUpdate` - fires `(currentState, ...)` from inside `:Update`.

Fields:

- `sm.CurrentState: string?`
- `sm.PreviousState: string?`

### Observer

Wraps a plain table in a userdata proxy. Writes to the proxy fire a callback; reads pass through.

- `Observer.new(tbl: {}, callback: (key, value) -> ()) -> userdata`
  - Returns a userdata proxy. Write to the proxy with `proxy.Field = value` and the callback fires as `callback(key, value)` after the underlying table has been updated. No-op writes (where `tbl[key] == value`) are skipped.

## Examples

### Bezier

```lua
local points = {
    Vector3.new(0, 0, 0),
    Vector3.new(5, 10, 0),
    Vector3.new(15, 10, 0),
    Vector3.new(20, 0, 0),
}

-- Smooth spline: the curve actually passes through every point.
for i = 0, 60 do
    local t = i / 60
    local pos = Bezier.GetSmoothSpline(t, points)
    -- use pos for a moving part, camera waypoint, etc.
end

-- Magnet path: starts at points[1], ends at points[#points],
-- middle points only pull the curve toward themselves.
local mid = Bezier.GetMagnetPath(0.5, points)
```

### Spring

```lua
local RunService = game:GetService("RunService")

local s = Spring.new(Vector3.zero, 0.6, 8) -- initial, damping, speed
s.Target = Vector3.new(10, 0, 0)

RunService.RenderStepped:Connect(function()
    part.Position = s.Position -- reading drives the simulation
end)

-- Knock it with an impulse to make it shake:
s:Impulse(Vector3.new(0, 30, 0))

-- Snap it back to origin, zero velocity:
s:Reset(Vector3.zero)
```

### StateMachine

Layout:

```
StateMachine.luau
States/
  Idle.luau
  Attacking.luau
```

`Idle.luau`:

```lua
return {
    CanEnter = function(prev) return true end,
    OnEnter = function(prev) print("-> Idle from", prev) end,
    OnExit = function() print("leaving Idle") end,
    Update = function(dt) --[[ idle tick ]] end,
}
```

Usage:

```lua
StateMachine:Init() -- once at boot

local sm = StateMachine.new("Idle")
sm.OnEnter:Connect(function(new, prev) print("entered", new) end)

sm:TryEnter("Attacking", target)

game:GetService("RunService").Heartbeat:Connect(function(dt)
    sm:Update(dt)
end)

print(sm:GetState()) -- "Attacking"
```

### Observer

```lua
local data = { Score = 0, Name = "Player" }

local watched = Observer.new(data, function(key, value)
    print(key, "->", value)
end)

watched.Score = 5       -- prints: Score -> 5
watched.Score = 5       -- no-op, callback does not fire
watched.Name  = "Bob"   -- prints: Name -> Bob

-- Reads pass through:
print(watched.Score)    -- 5

-- WRONG: bypasses the callback
data.Score = 999        -- nothing prints
```

## Gotchas

- **Spring is usually RenderStepped-driven.** The API is lazy: reading `.Position` or `.Velocity` re-integrates from the last snapshot. Callers wire that into `RunService.RenderStepped`, which is client-only. Do not drive a spring from a server `Script` and expect the same loop.
- **Reading a Spring is NOT free.** Every access to `.Position`/`.Velocity` runs the analytic solver. If you need the value several times in one frame, cache it into a local.
- **Spring `Speed` is clamped.** The `__newindex` forces negative speeds to `0`. Assigning an unknown field errors rather than silently doing nothing.
- **Spring damping regimes.** `<1` oscillates, `=1` is critically damped, `>1` is overdamped. Pick `1` when you want "fast, no bounce."
- **Bezier `GetMagnetPath` does not pass through inner points.** Only the first and last points are guaranteed on the curve. Use `GetSmoothSpline` when every point must be hit.
- **Bezier `t` is expected in `[0, 1]`.** Out-of-range values extrapolate in surprising ways. `GetSmoothSpline` clamps only at the very end (t >= 1 returns the last point).
- **Bezier was mostly AI-assisted.** The math (de Casteljau + Catmull-Rom) is standard but the author calls out to double-check before trusting it for anything critical.
- **StateMachine shares one state registry across all instances.** `states` is a module-level table. Two `StateMachine.new(...)` instances in the same game will see the same named states. Fork the module if you need per-instance registries.
- **StateMachine needs a `States` sibling folder.** `:Init()` does `script:WaitForChild("States")`. If that folder doesn't exist the require yields forever. Create it before calling `:Init()`.
- **StateMachine silently no-ops.** `TryEnter` does nothing on an unknown state name or when `CanEnter` returns false. Check `GetState()` afterwards if you need to confirm the transition happened.
- **Observer requires writing to the proxy.** Writing `data.Field = value` directly on the original table bypasses the callback. Only the returned userdata proxy triggers it.
- **Observer callbacks run inline.** They fire synchronously inside the write. A throw inside the callback propagates through the assignment.
- **Observer skips no-op writes.** Setting a field to the value it already has does not fire the callback. This is intentional but worth knowing if you use the callback as a "this field was touched" heartbeat.

## Related

- `Util.Trove` - cleanup helper; useful for managing the connections you wire up around springs, state machines, and observers.
- `Util.Signal` - the signal implementation used by `StateMachine.OnEnter` / `OnExit` / `OnUpdate`.
- `Util.CameraShaker` - another motion-focused utility; pair with `Spring:Impulse` for hitstop or recoil feel.
- `Util.ExtendedLibraries` - extra `math`/`Vector3` helpers if you want to build your own motion primitives alongside these.
