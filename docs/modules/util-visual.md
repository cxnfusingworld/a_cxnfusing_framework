# Util - Visual

## Overview

The visual-flavored utilities in `Shared.Utilities` are the grab-bag of modules that push pixels, shake cameras, stamp decals, play sounds, ragdoll bodies, and gate client work by visibility. They are the "make the game feel good" half of the Util folder: `CameraShaker` (vendored Sleitnick perlin-noise screen shake with a preset library), `FlashController` (one-call highlight + full-screen color wash), `DecalProjector` (bullet-hole/blood-splat stamper), `SFXHandler` (clone-and-play sound wrapper keyed off `ReplicatedStorage.Assets.SFX`), `Imagine` (scrolling tiled image animator for UI), `ViewCheck` (on-screen + line-of-sight test), `LODManager` (distance-based model swapper), and `Ragdoll` (Motor6D to BallSocketConstraint death-flop). Most are client-flavored; a few (`LODManager`, `ViewCheck`, `Imagine`, `FlashController`, `CameraShaker`) assume `workspace.CurrentCamera` or Lighting/RenderStepped and will misbehave on the server.

## Requiring it

Same rules as the rest of the framework: pick based on where you're calling from.

```luau
-- From user Scripts / LocalScripts (full loader)
local Framework = require(game.ReplicatedStorage.Framework)
local CameraShaker = Framework.Utilities.CameraShaker
local FlashController = Framework.Utilities.FlashController

-- From a ModuleScript that is itself loaded by the framework
local Shared = require(game.ReplicatedStorage.Shared)
local DecalProjector = Shared.Utilities.DecalProjector
```

Individual files live at `ReplicatedStorage.Shared.Components.Util.<Name>`, but `Framework.Utilities` / `Shared.Utilities` is the canonical path.

## API reference

### CameraShaker

Vendored third-party module from Stephen Leitnick (`RbxCameraShaker`). Perlin-noise-driven camera shake that runs on `RenderStep`, sums active shake instances each frame, and hands a `CFrame` offset to a user-supplied callback. The module itself does not touch `workspace.CurrentCamera`; your callback is responsible for applying the offset.

**Signature**

```luau
CameraShaker.new(renderPriority: number, callback: (shakeCFrame: CFrame) -> ()): CameraShaker
```

**Instance methods**

- `:Start()` - binds to `RenderStep` and starts feeding offsets to the callback.
- `:Stop()` - unbinds from `RenderStep`.
- `:StopSustained(fadeOutTime: number?)` - fades out every currently-sustained shake. `fadeOutTime` defaults to each shake's `fadeInDuration`.
- `:Shake(shakeInstance)` - adds a one-shot shake instance (e.g. from `Presets.Explosion`).
- `:ShakeSustain(shakeInstance)` - adds a sustained shake and starts its fade-in.
- `:ShakeOnce(magnitude, roughness, fadeInTime?, fadeOutTime?, posInfluence?, rotInfluence?)` - build + enqueue a one-shot shake without a preset. Fade-in is instantaneous.
- `:StartShake(magnitude, roughness, fadeInTime?, posInfluence?, rotInfluence?)` - same but sustained, fades in gradually.

**Fields**

- `CameraShaker.Presets` - preset factory table (see below).
- `CameraShaker.CameraShakeInstance` - the inner class, exposed for advanced callers.

**Presets** (each access returns a fresh instance via `__index`):

- `Bump` - high-magnitude, short, smooth. One-shot.
- `Explosion` - intense, rough, instant fade-in. One-shot.
- `Earthquake` - continuous rough shake. Sustained.
- `BadTrip` - high magnitude, low roughness, slow. Sustained.
- `HandheldCamera` - subtle, slow. Sustained.
- `Vibration` - very rough, low magnitude. Sustained.
- `RoughDriving` - medium magnitude, slight roughness. Sustained.

### FlashController

Quick flash VFX for hit-feedback, pickups, flashbangs, damage pulses. Combines a temporary `Highlight` on an Instance with a full-screen `ColorCorrectionEffect` color wash. One call, self-cleans after `FlashTime + FadeOut`.

**Signature**

```luau
FlashController:Flash(config: {
    Object: Instance?,
    Lighting: boolean?,
    Preset: string?,
    HighlightConfig: { FillTransparency: number?, OutlineTransparency: number?,
        FillColor: Color3?, OutlineColor: Color3?,
        FlashTime: number?, FadeOut: number?,
        DepthMode: Enum.HighlightDepthMode? }?,
    LightingConfig: { FlashTime: number?, FadeOut: number?, Color: Color3? }?,
}): Highlight?
```

**Behavior**

- If `Lighting` is truthy, tints the shared `ColorCorrectionEffect`. A pure-black `Color` drives `Brightness = -1` (darken pulse); any other color drives `Brightness = 1` (brighten pulse).
- If `Object` is set, spawns a `Highlight` parented to it with the given fill/outline colors, then tweens it out after `FlashTime`.
- Returns the `Highlight` when created, else `nil`.
- `Preset` is accepted but `presets` is currently empty; treat it as not-yet-wired.

### DecalProjector

Stamps a textured decal onto whatever a raycast hit, aligned to the surface normal. The decal lives on an anchored, non-collidable `Part` parented to `workspace.Debris`, and auto-destroys after `Lifetime + FadeTime`.

**Signature**

```luau
DecalProjector:Project(result: RaycastResult, config: {
    Texture: string,
    Size: Vector2?,
    Color: Color3?,
    Lifetime: number?,
    FadeTime: number?,
}): Part?
```

**Behavior**

- No-op if `result` or `config.Texture` is missing.
- Defaults: `Size = Vector2.new(2, 2)`, `Color = Color3.new(1,1,1)`, `Lifetime = 3`, `FadeTime = 1`.
- Nudges the part `0.05` studs along `result.Normal` to avoid z-fighting with the hit surface.
- Part is `Anchored`, `CanCollide`/`CanTouch`/`CanQuery = false` so it doesn't interfere with gameplay raycasts.

### SFXHandler

Plays one-shot sounds by name or by category. Each play clones the source `Sound` so overlapping plays don't stomp each other's playback position; clones are Debris'd after `TimeLength + 1`.

**Signatures**

```luau
SFXHandler:Play(obj: string | Sound, parent: Instance?, properties: {
    Volume: number?, Speed: number?, Pitch: {number} | number | nil,
    SoundGroup: string?, RollOffDist: number?, RandomPitch: boolean?,
}?): Sound?

SFXHandler:PlayByCategory(name: string, parent: Instance?, properties: ...): Sound?
```

**Behavior**

- `:Play` with a string searches `ReplicatedStorage.Assets.SFX:GetDescendants()` for a `Sound` whose `Name` matches. With a `Sound` directly, skips the lookup.
- `:PlayByCategory` looks for a `Folder` of the given name under `ReplicatedStorage.Assets.SFX.Categories` and picks a random child.
- `Pitch` accepts a number (fixed octave percent, `100 = 1.0`) or a `{min, max}` table (random in range). `RandomPitch = true` uses the default `{90, 110}` range.
- `parent` defaults to `SoundService`.
- Returns the clone (already `:Play()`ed), or `nil` if nothing matched.

### Imagine

Scrolls a tiled `ImageLabel` / `ImageButton` every frame so the texture looks like it's flowing (parallax backgrounds, conveyor belts, animated loading screens).

**Signatures**

```luau
Imagine.apply(base: ImageLabel | ImageButton): Image
Image:Animate(config: { XSpeed: number?, YSpeed: number?, XTileAmount: number?, YTileAmount: number? })
Image:Unapply()
```

**Behavior**

- `apply` returns a handle; `Animate` starts a `RenderStepped` loop that offsets the image position and wraps at tile boundaries.
- Defaults: `XSpeed = 100`, `YSpeed = 100`, `XTileAmount = 2`, `YTileAmount = 2`. Speeds are in pixels per second.
- `Animate` overwrites `base.Size` with `UDim2.fromScale(XTileAmount, YTileAmount)` because the wrap math relies on `Size.X.Scale` being non-zero.
- `Unapply` disconnects the render connection, clears the handle's fields, and nils the metatable. Don't reuse the handle after calling.

### ViewCheck

Small "can the camera actually see this world position?" helper. Combines a viewport on-screen check with a camera-to-target line-of-sight raycast.

**Signatures**

```luau
ViewCheck.IsOnScreen(target: Vector3): boolean
ViewCheck.IsBlocked(target: Vector3, ignoreList: {Instance}?): boolean
ViewCheck.IsVisible(target: Vector3, ignoreList: {Instance}?): boolean
```

**Behavior**

- `IsOnScreen` uses `Camera:WorldToViewportPoint`'s second return.
- `IsBlocked` raycasts from `cam.CFrame.Position` to `target` with a shared `RaycastParams` set to `Exclude` filtering over `ignoreList`.
- `IsVisible` returns `IsOnScreen AND NOT IsBlocked`.

### LODManager

Distance-based model swapper. Register a world CFrame and a list of `{Model, Distance}` pairs; a background loop picks the best model based on distance from `workspace.CurrentCamera` and parents it into the scene, parking inactive LODs in `ReplicatedStorage.LODDebris`.

**Signatures**

```luau
LODManager:Init()
LODManager:Register(CF: CFrame, Models: { { Model: Model | BasePart, Distance: number } }, Parent: Instance?): string
LODManager:Deregister(ID: string)
LODManager:UpdateCF(ID: string, newCF: CFrame)
LODManager:UpdateModels(ID: string, newModels: { ... }, Parent: Instance?)
```

**Behavior**

- Must call `:Init()` once to start the background update loop; without it, registrations sit idle.
- Distance picking: walks the `Models` list and selects the entry with the highest `Distance` still `<= current distance`. The lowest-distance entry (typically `0`) is your nearest/highest-detail model.
- `Deregister` destroys the registered models.
- `Config` fields are baked in: `UpdateSpeed = 0.5`, `BatchUpdateDelay = 0`, `BatchSize = 10`.

### Ragdoll

Minimal Motor6D-to-BallSocketConstraint ragdoll. `Start` disables the motors, parents sockets with +/-90 deg twist limits, and drops the humanoid into `PlatformStand`. `Stop` reverses it.

**Signatures**

```luau
Ragdoll.Start(character: Model)
Ragdoll.Stop(character: Model)
```

**Behavior**

- `Start` sets a `Ragdolling` attribute on the character and early-returns if it's already truthy (dedupe).
- Disables `BreakJointsOnDeath`, zeroes `WalkSpeed`/`JumpPower`, disables `AutoRotate`, sets `PlatformStand = true`, changes state to `Ragdoll`, and disables the `GettingUp` state so the humanoid doesn't auto-recover.
- `Stop` destroys all `BallSocketConstraint`s, re-enables `Motor6D`s, restores default humanoid settings (`WalkSpeed = 16`, `JumpPower = 50`), changes state to `GettingUp`, and disables the `Ragdoll` state.

## Examples

### Camera shake on explosion

```luau
local Framework = require(game.ReplicatedStorage.Framework)
local CameraShaker = Framework.Utilities.CameraShaker

local camera = workspace.CurrentCamera
local shaker = CameraShaker.new(Enum.RenderPriority.Camera.Value, function(shakeCFrame)
    camera.CFrame = camera.CFrame * shakeCFrame
end)
shaker:Start()

-- One-shot preset:
shaker:Shake(CameraShaker.Presets.Explosion)

-- Sustained handheld cam feel:
local handheld = shaker:ShakeSustain(CameraShaker.Presets.HandheldCamera)
task.wait(5)
handheld:StartFadeOut(1)
```

### Hit-flash highlight + flashbang

```luau
local Flash = Framework.Utilities.FlashController

-- Red highlight on the part that got hit
Flash:Flash({
    Object = hitPart,
    Lighting = false,
    HighlightConfig = { FillColor = Color3.new(1, 0, 0), FlashTime = 0.08, FadeOut = 0.1 },
})

-- Full-screen white flashbang
Flash:Flash({
    Lighting = true,
    LightingConfig = { Color = Color3.new(1, 1, 1), FlashTime = 0.1, FadeOut = 0.4 },
})
```

### Bullet-hole decals from a raycast

```luau
local DecalProjector = Framework.Utilities.DecalProjector
local result = workspace:Raycast(origin, direction * 500)
DecalProjector:Project(result, {
    Texture = "rbxassetid://12345678",
    Size = Vector2.new(0.8, 0.8),
    Lifetime = 6,
    FadeTime = 1.5,
})
```

### Pooled SFX playback

```luau
local SFX = Framework.Utilities.SFXHandler

SFX:Play("Gunshot_Pistol", gun.Handle, { Volume = 0.8, RandomPitch = true })
SFX:PlayByCategory("Footstep_Grass", character.HumanoidRootPart, { Volume = 0.5 })
```

### Scrolling UI background

```luau
local Imagine = Framework.Utilities.Imagine

local anim = Imagine.apply(backgroundImage)
anim:Animate({ XSpeed = 40, YSpeed = 0, XTileAmount = 3, YTileAmount = 3 })
-- later:
anim:Unapply()
```

### Skip work on offscreen NPCs

```luau
local ViewCheck = Framework.Utilities.ViewCheck

game:GetService("RunService").Heartbeat:Connect(function()
    for _, npc in pairs(npcs) do
        if ViewCheck.IsVisible(npc.Head.Position, { localPlayer.Character }) then
            npc.Nameplate.Enabled = true
        else
            npc.Nameplate.Enabled = false
        end
    end
end)
```

### LOD swap for props

```luau
local LOD = Framework.Utilities.LODManager
LOD:Init()

local id = LOD:Register(trunk.CFrame, {
    { Model = treeHigh, Distance = 0 },
    { Model = treeMed,  Distance = 75 },
    { Model = treeLow,  Distance = 200 },
}, workspace.Scenery)
```

### Death ragdoll

```luau
local Ragdoll = Framework.Utilities.Ragdoll

humanoid.Died:Connect(function()
    Ragdoll.Start(character)
    task.wait(3)
    Ragdoll.Stop(character)
end)
```

## Gotchas

- **`CameraShaker` is vendored.** Don't restyle the source to match the rest of the codebase; it's kept as-is to minimize diff noise if upstream ever gets pulled in. Your callback owns applying the offset; the shaker never touches `workspace.CurrentCamera` itself.
- **`CameraShaker.Presets` returns a fresh instance every access** via a `__index` metamethod. `local p = Presets.Explosion; shaker:Shake(p); shaker:Shake(p)` reuses the same fading instance for the second call. Grab a new one each time by indexing in the call: `shaker:Shake(Presets.Explosion)`.
- **`FlashController` creates a `ColorCorrectionEffect` in Lighting at require time.** Requiring on the server parents a stray effect. Client-only in practice.
- **`FlashController.Preset` arg is currently non-functional** - the internal `presets` table is empty. Pass `HighlightConfig` / `LightingConfig` directly for now.
- **`FlashController`'s pure-black `LightingConfig.Color` drives a darken pulse** (brightness `-1`) instead of a brighten pulse. Use any non-zero channel for a normal flash.
- **`DecalProjector` silently no-ops** if `result` or `config.Texture` is missing. Check your raycast result before calling if you need the failure to be loud.
- **`SFXHandler` requires the exact asset layout** `ReplicatedStorage.Assets.SFX` with a `Categories` sub-folder. The `WaitForChild` at require time will yield forever if the layout isn't there; create the folders before the module loads.
- **`SFXHandler:Play` does a linear `GetDescendants` search by name** each call. Fine for small libraries; cache the `Sound` reference yourself if you're calling thousands of times a second.
- **`SFXHandler:PlayByCategory` will error** if the found `Folder` is empty (`math.random(1, 0)`). Make sure every category has at least one sound.
- **`Imagine` is client-only** (uses `RenderStepped`). It also reaches the RunService via `game["Run Service"]`, which is the stringly-typed form but still works. `Animate` overwrites `base.Size` to `UDim2.fromScale(XTileAmount, YTileAmount)`; if you need a fixed pixel size, wrap the image in a container and size the container.
- **`Imagine:Unapply` nukes the metatable and clears fields** by iterating `self`. Don't call methods on the handle after `:Unapply`.
- **`ViewCheck` shares a module-level `RaycastParams`** across all callers. Concurrent/parallel callers will clobber each other's `FilterDescendantsInstances`. Fine in single-threaded Luau.
- **`ViewCheck` captures `workspace.CurrentCamera` at require time.** On the server `CurrentCamera` is nil; don't require on the server.
- **`LODManager:Init` must be called exactly once** to start the update loop. Without it, registrations pile up but nothing swaps.
- **`LODManager` has known bugs** (see `docs/reviews/shared-review.md`): the batch-index math uses an operator-precedence-bugged expression that lumps every registration into batch 1, and `Deregister` does some aggressive `table.clear`s on shared refs. Works for small worlds; expect uneven batching on large ones.
- **`Ragdoll.Start`'s created `Attachment`s leak on `Stop`.** Only the `BallSocketConstraint`s get cleaned up; the per-joint attachments stay on the limbs. Usually harmless but will accumulate if you ragdoll-loop a character.
- **`Ragdoll.Stop` explicitly disables the `Ragdoll` humanoid state** after forcing `GettingUp`. This is intentional: it stops Roblox's built-in state machine from snapping the character back into ragdoll on its own. If you want the humanoid to auto-ragdoll again later, re-enable it yourself.

## Related

- `Shared.Utilities.Trove` - clean up `CameraShaker`, `Imagine` handles, `Highlight`s, etc. on teardown by registering them with a Trove.
- `Shared.Utilities.Pool` - pair with `DecalProjector` or `SFXHandler` if you want to reuse parts/sounds rather than clone-and-debris each time.
- `Shared.Services.VFX` - particle-flavored VFX service, the companion to the visual utilities here.
- `Shared.Functions.FastTween` - what `FlashController`, `DecalProjector`, and `Imagine` use internally; reach for it directly when you need a quick tween without the flash-wrapper semantics.
- `docs/reviews/shared-review.md` - open-source-pass review with known bugs and targeted fixes for these modules.
- Sleitnick's original [`RbxCameraShaker`](https://github.com/Sleitnick/RbxCameraShaker) - upstream source for the vendored `CameraShaker`.
