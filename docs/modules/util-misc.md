# Util - Misc

## Overview

A collection of five utilities grouped here because they don't fit neatly into their own doc page: `DebugUtils` (visual raycast/hitbox debug parts plus `SafeCall`/`TimeBlock`/`ProfileInstance`), `TimeFormatter` (turn seconds into clock or long-form strings), `IllusionIAS` (the input action system built on Roblox's new `InputAction`/`InputContext` APIs, with hold/toggle/tap/cooldown/input-buffer on top of named binds), `ExtendedLibraries` (drop-in metatable wrappers that glue extra helpers onto `math`, `Random`, `table`, and `Vector3`), and `SpectateService` (client-side camera-follow helper plus a small scripted `CameraManager` for cinematic/fixed cameras). Mixed bag by intent; each entry is small enough to grok in one sitting.

## Requiring it

Everything in this page lives on `Utilities`, same on both surfaces:

```luau
-- From user Scripts / LocalScripts
local Framework = require(game.ReplicatedStorage.Framework)
local DebugUtils = Framework.Utilities.DebugUtils
local TimeFormatter = Framework.Utilities.TimeFormatter
local IIAS = Framework.Utilities.IllusionIAS
local Ext = Framework.Utilities.ExtendedLibraries
local Spectate = Framework.Utilities.SpectateService

-- From a ModuleScript loaded by the framework
local Shared = require(game.ReplicatedStorage.Shared)
local DebugUtils = Shared.Utilities.DebugUtils
```

Individual files sit at `ReplicatedStorage.Shared.Components.Util.<Name>` but `Utilities` is the canonical path.

## API reference

### DebugUtils

Grab-bag of dev-time helpers. All debug shapes spawn under an auto-created `workspace.Debug` folder so you can nuke the whole lot at once.

- `DebugUtils.SafeCall(func, ...)` - `pcall`s `func` with the varargs; on failure prints a `[SafeCall Error]` warn and swallows the error.
- `DebugUtils.TimeBlock(label, func)` - runs `func` inside a `tick()`-timed block, prints `[TimeBlock] <label> -> <ms> ms`. Errors inside `func` are pcalled and warned but timing still prints.
- `DebugUtils.ProfileInstance(inst)` - prints the instance's full name, every attribute (key + value), and every child (name + class). Dev-time inspection only.
- `DebugUtils.NewLine(from: Vector3, to: Vector3, config?)` - spawns a thin neon Part stretched between `from` and `to`. Config: `lifetime` (default 10s; `0` = forever), `color`, `thickness` (X/Y size in studs), `transparency`. Returns the Part.
- `DebugUtils.NewSphere(at: Vector3, config?)` - neon Ball Part centered at `at`. Config: `lifetime`, `color`, `size` (scalar; applied uniformly), `transparency`. Returns the Part.
- `DebugUtils.NewBox(at: CFrame, config?)` - neon Part at `at`. Config: `lifetime`, `color`, `size` (Vector3), `transparency`. Returns the Part. **Note:** current implementation sets `Shape = Ball`, so the resulting visual is a ball, not a box. See gotchas.

### TimeFormatter

Whole-seconds-in, text-out. Nothing yields. All functions take an optional `padding` bool controlling whether the top field is zero-padded to two digits (e.g. `"01:05"` vs `"1:05"`).

- `TimeFormatter.FormatMS(seconds: number, padMinutes: boolean): string` - `MM:SS` clock. Zero or negative input returns `"00:00"` / `"0:00"`.
- `TimeFormatter.FormatHMS(seconds: number, padHours: boolean): string` - `HH:MM:SS` clock. Minutes and seconds are always two-digit; only hours respect `padHours`. Zero/negative returns `"0:00:00"`.
- `TimeFormatter.FormatDHMS(seconds: number, padDays: boolean): string` - `DD:HH:MM:SS`. Same rule: only the leading field optionally pads. Zero/negative returns `"0:00:00:00"`.
- `TimeFormatter.FormatLong(seconds: number): string` - `"1d 2h 3m 4s"` style with zero fields dropped. Zero/negative returns `"0s"`.
- `TimeFormatter.SmartFormat(seconds: number, addPadding: boolean): string` - picks the smallest format that fits: under a minute → `"0:30"`/`"00:30"`, under an hour → `MM:SS`, under a day → `HMS`, otherwise `DHMS`.

### IllusionIAS

Illusion's Input Action System. Thin but opinionated wrapper over Roblox's `InputAction` / `InputContext` / `InputBinding`. Declare a named action, bind one or more key combos to it, listen to `Started` / `Ended` / `Activated` signals. Adds hold-vs-toggle, multi-tap activation, cooldown, input buffering, and grouped "contexts" you can enable or disable in bulk.

**Client-only.** Requiring on the server returns an empty stub, so shared code can still `require` it without crashing.

**Module (static) functions**

- `IIAS.new(name: string, inputType: Enum.InputActionType?) -> Object` - creates (or returns the existing) action. Default `inputType` is `Bool`. Same name twice returns the same action (dedup).
- `IIAS.get(name: string) -> Object?` - fetch by name.
- `IIAS.getAll() -> { [string]: Object }` - every live action.
- `IIAS.newContext(name)` / `IIAS.getContext(name)` / `IIAS.addContext(name, ...actions)` / `IIAS.removeFromContext(name, action)` / `IIAS.removeAllFromContext(name)` / `IIAS.removeContext(name)` / `IIAS.clearContexts()` - context bookkeeping. A context is a named group of actions you can bulk-toggle.
- `IIAS.enableContext(name: string, enabled: boolean)` - calls `SetEnabled(enabled)` on every action in the context.
- `IIAS.isContextEnabled(name: string) -> boolean`

**Action object (`Object`) methods**

- Signals: `Activated(value, pressed)`, `Started(value)`, `Ended(value)` - each is an `IAScriptSignal` with `:Connect` / `:Once` / `:Wait` / `:DisconnectAll`.
- `obj:Fire(value, pressed)` - manually dispatch the signals (useful for on-screen buttons).
- `obj:AddBind(mainKey, primaryMod?, secondaryMod?)` - add a key combo. If the same key is already bound, updates its modifiers. First bind fills the canonical `MainAction` slot; later binds become `AltAction1`, `AltAction2`, ...
- `obj:SetBind(mainKey, primaryMod?, secondaryMod?)` - clear all binds, then add this one.
- `obj:RemoveBind(mainKey, primaryMod?, secondaryMod?)` - remove the bind matching this exact key+mods combo.
- `obj:EditBind(oldMain, oldMods, newMain, newMods)` - rebind in place.
- `obj:ClearBinds()` - wipe everything, reset `MainAction` to empty.
- `obj:GetBinds() -> Binds` - deep copy of the current bind table.
- `obj:SetHold(bool)` - `true` (default): fires while key held. `false`: press toggles state.
- `obj:SetUIButton(button: GuiButton?)` - route a `GuiButton` to this action (Bool only).
- `obj:SetTapActivation(required, window)` - require N presses inside `window` seconds to fire (Bool only).
- `obj:SetCooldown(seconds)` / `obj:ResetCooldown()` - minimum gap between fires (Bool/ViewportPosition).
- `obj:SetEnabled(bool)` / `obj:IsEnabled()` - disable mutes signals and turns off the `InputContext`.
- `obj:SetPriority(n)` / `obj:GetPriority()` / `obj:SetSink(bool)` / `obj:GetSink()` - passthrough to the underlying `InputContext`.
- `obj:SetScale(n)` / `obj:GetScale()` / `obj:SetVectorScale(v)` / `obj:GetVectorScale()` / `obj:SetResponseCurve(n)` / `obj:GetResponseCurve()` / `obj:SetPressedThreshold(n)` / `obj:GetPressedThreshold()` / `obj:SetReleasedThreshold(n)` / `obj:GetReleasedThreshold()` / `obj:SetPointerIndex(n)` / `obj:GetPointerIndex()` / `obj:SetClampMagnitudeToOne(bool)` / `obj:GetClampMagnitudeToOne()` - passthroughs to the underlying `InputBinding`.
- `obj:SetInputBufferEnabled(bool)` / `obj:SetInputBufferTime(seconds)` - when a press is swallowed (cooldown active or another key holding), buffer it and replay as soon as the gate clears. Bool only.
- `obj:GetState() -> variant` - current active value.
- `obj:SetCompositeDirections(up, down, left, right, forward, backward)` - composite bindings for `Direction1D` / `Direction2D` / `Direction3D` (WASD-style vectors).
- `obj:SetCompositeModifiers(primaryMod, secondaryMod)` - modifier keys for composite binds.
- `obj:Destroy() -> boolean` - disconnect every signal, destroy every `InputAction` / `InputContext`, and deregister from all contexts.

**Action types**

- `Enum.InputActionType.Bool` - default. Boolean pressed/released. All extras (Hold, TapRequired, Cooldown, InputBuffer) work here.
- `Enum.InputActionType.Direction1D` / `Direction2D` / `Direction3D` - numeric vector inputs (WASD, thumbstick, ...). Currently fires the raw variant via `Activated`; Hold/Tap/Cooldown are not finished for these types.
- `Enum.InputActionType.ViewportPosition` - pointer position. Cooldown is respected, Hold/Tap are not.

### ExtendedLibraries

Namespace-extension module. Returns `{ math, Random, table, Vector3 }`, each a wrapper around the real global. Originals still reachable via a metatable `__index`, so `math.pi`, `Random.new(seed)`, `table.clone(t)`, `Vector3.new(x,y,z)`, etc. all work on the wrappers. Added functions per namespace are listed below.

**`ExtendedLibraries.math`** (`MathExtended.luau`) - all helpers assert numeric inputs and throw on non-numbers:

- `math.roundToNearest(value, step)` / `math.ceilToNearest(value, step)` / `math.floorToNearest(value, step)` - snap to a multiple of `step`. `step == 0` returns `value` unchanged.
- `math.average(...: number)` - numeric mean of the varargs. Errors if called with no values.
- `math.lerp(a, b, t)` - linear interpolate. No clamping on `t`.
- `math.inverseLerp(a, b, val)` - returns how far `val` lies between `a` and `b` in `[0, 1]` ideal, unclamped. Returns `0` if `a == b`.
- `math.map(val, inMin, inMax, outMin, outMax)` - remap one range to another. Returns `outMin` if `inMin == inMax`.
- `math.normalizeAngle(angle)` - wrap into `[0, 360)`.

**`ExtendedLibraries.Random`** (`RandomExtended.luau`):

- `Random.weighted({ {value=..., weight=...}, ... })` - picks a `value` proportional to its `weight`. Errors on empty array, non-numeric weight, or a total weight of zero.
- `Random.chooseFromArray(t)` - random element of a numeric-keyed array. Warns and returns `nil` if `t` isn't an array or is empty (silenceable via `Random.Config.Warnings = false`).
- `Random.chooseFromDictionary(t)` - random value from a dictionary. Uniformity depends on iteration order.

**`ExtendedLibraries.table`** (`TableExtended.luau`):

- `table.merge(t1, t2)` - returns a **new** shallow-merged table. Keys from `t2` win on conflicts. Nested tables are shared references (not deep-copied).

**`ExtendedLibraries.Vector3`** (`Vector3Extended.luau`) - asserts `Vector3`/`number` inputs, throws on mismatch:

- `Vector3.getDistance(a, b)` - `(a - b).Magnitude`.
- `Vector3.getDirection(a, b)` - normalized `a → b`. Returns `Vector3.zero` if the points are identical.
- `Vector3.getCenterPos(a, b)` - midpoint.
- `Vector3.roundVector(vec, roundTo)` - round each component to the nearest multiple. `roundTo == 0` returns `vec`.
- `Vector3.reflectOffNormal(vec, normal)` - standard reflection `r = v - 2(v·n)n`. Returns `vec` if `normal.Magnitude == 0`.
- `Vector3.flattenX(vec)` / `flattenY(vec)` / `flattenZ(vec)` - zero out one component.
- `Vector3.project(V1, V2)` - the component of `V1` along `V2`. Returns `Vector3.zero` if `V2` has zero magnitude.

### SpectateService (client-only)

Switch the local player's camera to follow another player, model, or part. **Client-only.** Requiring on the server returns the stub value `true`, so shared imports don't crash but server code should not try to call into it.

- `SpectateService:Init()` - wires auto-fallback hooks. When your character respawns or the player you're spectating leaves, the service calls `StopWatching` for you. Call once on the client at startup.
- `SpectateService:Watch(target, options?)` - start watching `target`. `target` can be a `Player`, `Model`, or `BasePart`. `options` (merged into the defaults):
  - `SpectateType: "Custom" | "LockFirstPerson"` - default `"Custom"`. Third-person default camera, or a first-person view.
  - `UseNativeFirstPerson: boolean` - default `true`. Use Roblox's `CameraMode.LockFirstPerson` when `SpectateType == "LockFirstPerson"`. Set `false` to force the scripted `CameraManager` path (mounts on the target's `Head`, then `PrimaryPart`, then the part itself).
  - `ReturnToDefaultOnSelf: boolean` - default `true`. When `target` is the local player, skip the spectate-camera mode and keep the player on their normal camera.
- `SpectateService:StopWatching()` - restore the pre-spectate `CameraMode`, destroy the scripted camera if any, then `Watch` the local player.
- `SpectateService:GetWatched() -> Player | Model | BasePart` - current watch target.
- `SpectateService:SetDefaultOptions(options)` - overwrite the defaults used when `Watch` is called with no `options` arg.

**CameraManager** (used internally by `SpectateService` for the scripted first-person path; also usable standalone for fixed/cinematic cameras):

- `CameraManager.new(camPart: BasePart | Attachment, config?) -> Camera` - wrap a part or attachment. Config keys: `FOV` (nil = don't change), `CanLookAroundX` / `CanLookAroundY` (default `true`), `RotLerpAmount` (default `0.05`; `1` = instant), `LookXDivider` / `LookYDivider` (default `1`; larger = smaller look-around angle).
- `camera:Open()` - make this camera the active one. Overrides any previously open one.
- `camera:Destroy()` - nil every field on the metatable and close if this camera was open. Do not use the handle after this.
- `CameraManager.close()` - close whichever camera is open; does not destroy anything.

Moving parts/attachments are supported: the camera snaps to their position and smooth-lerps the rotation.

## Examples

### Visualizing a raycast with DebugUtils

```luau
local DebugUtils = Framework.Utilities.DebugUtils

local origin = muzzle.WorldPosition
local hit = workspace:Raycast(origin, direction * 100)

DebugUtils.NewLine(origin, hit and hit.Position or origin + direction * 100, {
    color = Color3.new(1, 0, 0),
    lifetime = 2,
})

if hit then
    DebugUtils.NewSphere(hit.Position, { size = 0.5, color = Color3.new(0, 1, 0), lifetime = 2 })
end
```

### Countdown with TimeFormatter

```luau
local TimeFormatter = Framework.Utilities.TimeFormatter

for i = 10, 0, -1 do
    label.Text = TimeFormatter.FormatMS(i, true) -- "00:10" ... "00:00"
    task.wait(1)
end

playtimeLabel.Text = TimeFormatter.SmartFormat(player:GetAttribute("Playtime"), true)
-- 45 -> "0:45", 600 -> "10:00", 7200 -> "2:00:00"
```

### Declaring a jump action with IllusionIAS

```luau
local IIAS = Framework.Utilities.IllusionIAS

local jump = IIAS.new("Jump", Enum.InputActionType.Bool)
jump:AddBind(Enum.KeyCode.Space)
jump:SetCooldown(0.2)
jump:SetInputBufferEnabled(true)

jump.Started:Connect(function()
    character.Humanoid.Jump = true
end)

-- Group combat binds so a menu can mute them with one call.
IIAS.newContext("Combat")
IIAS.addContext("Combat", jump)
IIAS.enableContext("Combat", false) -- disable all combat binds at once
```

### ExtendedLibraries

```luau
local Ext = Framework.Utilities.ExtendedLibraries
local math, Vector3, Random = Ext.math, Ext.Vector3, Ext.Random

local pct = math.inverseLerp(0, 100, score)
local pixels = math.map(pct, 0, 1, 0, 500)

local mid = Vector3.getCenterPos(partA.Position, partB.Position)
local snapped = Vector3.roundVector(rawPos, 4)

local rarity = Random.weighted({
    { value = "Common",    weight = 70 },
    { value = "Rare",      weight = 25 },
    { value = "Legendary", weight = 5 },
})
```

### Spectating another player

```luau
local Spectate = Framework.Utilities.SpectateService
Spectate:Init()

-- Third-person spectate
Spectate:Watch(otherPlayer)

-- First-person spectate via native CameraMode
Spectate:Watch(otherPlayer, { SpectateType = "LockFirstPerson" })

-- Back to yourself
Spectate:StopWatching()
```

## Gotchas

- `DebugUtils.NewBox` currently sets `Shape = Ball`, so it renders as a ball despite the name. Use `NewSphere` for balls and accept the mis-shape here until it's fixed upstream, or set `Shape = Block` on the returned Part yourself.
- `DebugUtils` uses `tick()` for `TimeBlock`; fine today but Roblox has deprecated it. Swap to `os.clock()` in your own profiling code.
- Debug parts spawn under `workspace.Debug` on whichever side called it. Called on the server = every client sees it. Called on the client = local-only.
- `lifetime = 0` means "live forever". The Part will not be cleaned up unless you destroy it manually.
- `TimeFormatter` expects whole seconds. Fractional seconds truncate silently via `%02d`; pass `math.floor(t)` if you care.
- `IllusionIAS` is client-only. On the server `require` returns `{}`, so `IIAS.new` is `nil` there; guard with `RunService:IsClient()` in shared code.
- `IIAS` extras (Hold, TapRequired, Cooldown, InputBuffer) are Bool-only today. Direction1D/2D/3D actions fire raw values only.
- `IIAS.new(name, ...)` is idempotent: calling with the same name returns the existing action instead of creating a duplicate.
- `IIAS` signal handlers (`Started`, `Ended`, `Activated`) run synchronously on the caller's thread. An erroring handler will interrupt the rest; `pcall` inside the handler if that matters.
- `ExtendedLibraries` wrappers are *local* shadows. They do not monkey-patch the global `math`/`table`/`Random`/`Vector3`. Each script that wants the extras has to require the library.
- `ExtendedLibraries.table.merge` is **shallow**. Nested tables are shared by reference between inputs and output. Deep-merge yourself if you need it.
- `ExtendedLibraries.math` / `Vector3` assert types and throw on bad input; `pcall` if you want soft failure.
- `Random.chooseFromArray` silently returns `nil` (with a warn) on non-arrays or empty tables. Toggle `Random.Config.Warnings = false` to silence.
- `SpectateService` is client-only. On the server `require` returns `true`, which will crash any `Spectate:Watch(...)` call. Gate with `RunService:IsClient()`.
- `CameraManager` connects `RenderStepped` at require time, so requiring `SpectateService` on the server isn't safe today. If you import it from shared code, guard the require behind a client check.
- Only one `CameraManager` camera is "open" at a time. `:Open()` on a new one closes the previous. `:Destroy()` nils every field on the metatable, so don't keep using the handle after destroying.

## Related

- `Shared.Functions` - smaller pure helpers (`FastTween`, `FormatNumber`, `AddCommas`, `GenerateUID`). See `docs/modules/functions.md`.
- `Shared.Utilities.Trove` - lifecycle container; pair with `DebugUtils` and `IIAS` if you want to auto-clean debug parts and disconnect input signals on teardown.
- `Shared.Utilities.Signal` / `Shared.Utilities.CameraShaker` / `Shared.Utilities.Spring` - other utilities documented alongside in the Util surface.
- Roblox APIs behind the wrappers: `TweenService`, `InputAction` / `InputContext` / `InputBinding`, `Camera` / `CameraType`, `CollectionService`, `HttpService:GenerateGUID`. Drop down to them directly when the wrapper gets in the way.
- Review notes and follow-up fix bullets for these modules live in `docs/reviews/shared-review.md` (see the `DebugUtils`, `TimeFormatter`, `IllusionIAS`, `ExtendedLibraries`, and `SpectateService` sections).
