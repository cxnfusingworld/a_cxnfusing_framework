# VFX

## Overview

The VFX service plays pooled visual effects (ParticleEmitters, Lights, attachments, Models, BaseParts) by name. Effect templates live under `ReplicatedStorage.Assets.VFX` and are cloned into a pool at framework load so repeated spawns avoid re-cloning. A single call (`VFX.newVFX(data)`) handles pooling, positioning, optional replication (local-only, all clients, or a whitelist), and auto-return-to-pool when the burst finishes or a looped effect's timer elapses. Emission always runs on clients; the server's only job, when replication is requested, is to fan the event out to the right audience. Optional per-effect `CustomEffects` module scripts let you layer bespoke behavior (tweens, sounds, etc.) on top of the default emit pass.

## Requiring it

From user code that goes through the framework loader:

```lua
local Framework = require(game.ReplicatedStorage.Framework)
local VFX = Framework.Services.VFX
```

From a ModuleScript that is itself loaded by the framework (avoid circular requires):

```lua
local Shared = require(game.ReplicatedStorage.Shared)
local VFX = Shared.Services.VFX
```

Both the client and the server require and `:Init()` the service. Emission of particles/lights happens on clients only; requiring on the server is valid (and required for `Replicate = true` fan-out).

## API reference

### `VFX.newVFX(data)`

Plays a VFX by name. Works on both client and server.

`data` fields (all optional except `VFXName`):

| Field | Type | Meaning |
|-------|------|---------|
| `VFXName` | `string` | Name of a child of `ReplicatedStorage.Assets.VFX`. Required. |
| `Looped` | `boolean` | If true, enables emitters continuously instead of one-shot. Pair with `DestroyAfter`. |
| `Destroy` | `boolean` | One-shot mode. When not `false`, the instance returns to the pool after the longest particle's `Lifetime.Max`. Pass `false` to keep it alive and manage it yourself. |
| `DestroyAfter` | `number` | Looped mode. Seconds before the looped effect is automatically returned to the pool. Without it, the caller is responsible for stopping it. |
| `Parent` | `Instance` | Parent for the live effect. Defaults to `workspace.VFX` (the `wVFX` folder). |
| `Position` | `Vector3` | Shorthand for `CFrame = CFrame.new(Position)`. |
| `CFrame` | `CFrame` | Placement. Used for Model (`:PivotTo`), BasePart (`.CFrame`), or Attachment (`.WorldCFrame` by default). Falls back to origin if absent. |
| `UseLocalAttachmentCF` | `boolean` | For Attachment templates: if true, assigns to `.CFrame` (parent-relative) instead of `.WorldCFrame` (worldspace). Use when the effect rides along with a parent. |
| `Replicate` | `boolean` | If true on a client, routes through the server and back out to clients. If false/omitted, plays locally only. Ignored if called from the server (server always replicates). |
| `Recipients` | `{Player}` | Optional whitelist. When replicating, only these players see the effect. Omit to broadcast to all. |

Behavior by call site:

- **Client, `Replicate = false` (or omitted):** fires the local-only signal; the same client plays the effect, nothing goes over the wire.
- **Client, `Replicate = true`:** fires the server, which fans out via `FireClient` per recipient (or `FireAllClients`).
- **Server:** always fans out to clients directly. The server never plays particles itself.

Per-asset defaults: if the VFX template contains a `Configuration` child, its `ValueBase` children are read into the data table as defaults. Caller-supplied fields override them.

### Custom effect modules (client-only)

Any ModuleScript under `Services/VFX/CustomEffects` whose name matches a VFX asset name is required on the client at `:Init()` time. The module returns a function called as `module(instance, config)` just before the default emit pass. Return value semantics:

- Return `false` - suppress the default emit pass (your module is fully responsible for playback).
- Return `nil` or nothing - let the service emit as usual.
- Return any other value - emit as usual.

The server does not load custom effect modules (it never emits).

### `Emitter` (internal)

`Services/VFX/Emitter.luau` is the particle-playback helper the service uses internally. You should not require it directly. For completeness:

- `Emitter:Emit(particle)` - one-shot on a single `ParticleEmitter`. Respects `Delay` and `EmitCount` attributes on the emitter. Returns `delay + Lifetime.Max`.
- `Emitter:LoopEmit(particle)` - sets `.Enabled = true` on a single emitter (after optional `Delay`). The pool reset callback is what eventually turns it back off.
- `Emitter:EmitAll(obj, destroy, pool)` - walks `obj` and its descendants, calls `:Emit` on each ParticleEmitter, and (unless `destroy == false`) waits the longest lifetime then returns `obj` to `pool`.
- `Emitter:LoopEmitAll(obj, destroyAfter, pool)` - same walk but via `:LoopEmit`. Returns `(destroy, longest)`: call `destroy()` to stop and return to pool, or pass `destroyAfter` and it will be scheduled automatically.

Per-emitter `Delay` and `EmitCount` attributes on a ParticleEmitter let artists tune timing in Studio without code changes. No attributes = instant single emit.

## Examples

One-shot explosion replicated to everyone:

```lua
local VFX = Framework.Services.VFX

VFX.newVFX({
    VFXName = "Explosion",
    CFrame = hitPart.CFrame,
    Replicate = true,
    Destroy = true,
})
```

Local-only footstep puff (no network traffic):

```lua
VFX.newVFX({
    VFXName = "FootstepPuff",
    Position = foot.Position,
})
```

Looping aura that turns off after 3 seconds:

```lua
VFX.newVFX({
    VFXName = "Aura",
    Looped = true,
    DestroyAfter = 3,
    Parent = character,
    CFrame = character:GetPivot(),
})
```

Scoped replication to teammates only:

```lua
VFX.newVFX({
    VFXName = "HealRing",
    CFrame = caster:GetPivot(),
    Replicate = true,
    Recipients = teammates, -- {Player}
})
```

Attachment that rides along with a weapon:

```lua
VFX.newVFX({
    VFXName = "MuzzleFlash",
    Parent = weapon.Muzzle, -- parent it to the attachment point
    UseLocalAttachmentCF = true,
    CFrame = CFrame.new(), -- parent-relative
})
```

## Gotchas

- **`wVFX` / `rVFX` folder convention.** The service creates two folders at init time:
  - `workspace.VFX` (the `wVFX` folder) - the visible parent for live effects when no `Parent` is specified by the caller. Only the server creates it; clients wait for it to replicate in.
  - `ReplicatedStorage.VFXStorage` (the `rVFX` folder) - hidden parking for pooled instances that are not currently playing. Keeping them here (instead of workspace) means they do not show up in workspace queries but stay alive for reuse.
  If you see effect instances in `ReplicatedStorage.VFXStorage`, they are just idle pool entries, not leaks.

- **Server fans out, clients emit.** The server never calls into `Emitter` or touches ParticleEmitter state. If you invoke `Emitter` yourself from the server, you will toggle state on the server replica and see nothing. Always go through `VFX.newVFX`, and let replication carry the request to clients.

- **`ReplicatedStorage.Assets.VFX` must exist at load time.** The service uses `WaitForChild` on it during require. If the folder is missing, the require yields forever and the whole framework never finishes loading. Make sure your asset pipeline publishes that folder before users require the framework.

- **Custom effect modules are client-only.** Anything under `Services/VFX/CustomEffects` is only required on the client. Do not assume a custom effect runs on the server.

- **`Configuration` child defines per-asset defaults.** Drop a `Configuration` instance inside a VFX template and populate it with `ValueBase` children (e.g. `BoolValue "Looped"`, `NumberValue "DestroyAfter"`). Those become defaults for every spawn; caller data wins on conflict. Handy for letting artists tune effects without touching code.

- **`Enabled` state is remembered.** At init, the service stashes every `ParticleEmitter`/`Light`'s authored `Enabled` value as a `DefaultEnabled` attribute. When an instance returns to the pool, emitters/lights are reset to that authored state (and particles are `:Clear()`ed), so the next caller does not see leftovers or get ambient-always-on emitters mistakenly turned off.

- **Emission on Attachments: local vs world.** By default, CFrame is written to `.WorldCFrame` (worldspace). Set `UseLocalAttachmentCF = true` to write to `.CFrame` instead - use this when the effect should ride along with its parent (e.g. bolted to a weapon's muzzle).

## Related

- `Shared/Components/Util/Pool` - the pool implementation the service uses to recycle instances.
- `Shared/Network` - the RemoteEvent + Signal wrappers behind `GlobalEvents.VFX` and `LocalEvents.VFX` that carry the replication action.
- `Shared/Components/Util/SFXHandler` - sibling service for sound effects; similar asset-folder pattern under `ReplicatedStorage.Assets.SFX`.
- `Services/VFX/Emitter.luau` - the particle helper described above; internal, but documented here for completeness.
