# GameFeel

GameFeel is a bundle of small client-side tweaks that make platformer-style movement feel less punishing: coyote time, jump buffering, apex gravity, and landing VFX. Each feature lives in its own sub-module under `ReplicatedStorage/Modules/Client/GameFeel/`, and the top-level `init.luau` auto-discovers them, hands each one a shared `Config` table, and exposes a flat toggle API (`SetEnabled`, `Set`, `SetAll`, `SetConfig`). Features are independent: turn on only what you need, tune numbers at runtime, and re-toggle to push new config values into a running module.

## Overview

- **Coyote time** - the grace window after you walk off a ledge where you can still jump. Named after the cartoon coyote who hangs in mid-air for a beat before gravity kicks in. Makes timing ledge jumps forgiving.
- **Jump buffer** - remembers that the player pressed jump slightly before landing, and fires the jump as soon as they touch ground. Without it, a press right before landing just gets eaten.
- **Apex gravity** - lowers gravity right at the top of a jump where the player is nearly weightless anyway. Gives more hang-time at the peak without making the rest of the jump feel slow.
- **Landing FX** - spawns a landing VFX (via the `VFX` service with `VFXName = "Landing"`) when the player lands from a fall tall enough to be worth sparkling about.

## Requiring it

GameFeel is client-only. Require it once from a `LocalScript`, call `:Init()` to auto-load the sub-modules, then flip features on.

```lua
local GameFeel = require(game:GetService("ReplicatedStorage")
    :WaitForChild("Modules")
    :WaitForChild("Client")
    :WaitForChild("GameFeel"))

GameFeel:Init()
GameFeel:SetAll(true)
```

`:Init()` walks its own children, requires every `ModuleScript` it finds, and stashes the returned toggle function under the module's name. Drop a new feature file into the folder and it'll be wired up on the next `:Init()` with no code changes.

## API reference

### Top-level (`GameFeel`)

#### `GameFeel:Init()`
Auto-discovers every sibling `ModuleScript` in the `GameFeel` folder and registers its toggle function under the module's filename. Call this once after requiring.

#### `GameFeel:SetEnabled(data: {[string]: boolean})`
Batch toggle. The table you pass is the **full** on/off picture, not a partial patch: any known module not mentioned in `data` gets disabled.

```lua
GameFeel:SetEnabled({
    CoyoteTime = true,
    JumpBuffer = true,
    ApexGravity = true,
    LandingFX = true,
})
```

#### `GameFeel:Set(modName: string, enabled: boolean)`
Toggle a single feature by name. No-op if the name isn't a loaded module.

```lua
GameFeel:Set("CoyoteTime", true)
```

#### `GameFeel:SetAll(enabled: boolean)`
Toggle every loaded feature at once.

#### `GameFeel:SetConfig(newConfig)`
Update tuning values in the shared `Config` table. Only known keys are copied; unknown keys are silently ignored (prevents typos from polluting state). Passing `nil` resets `Config` back to the pristine defaults.

Known keys:

| Key | Default | Meaning |
|---|---|---|
| `CoyoteTime` | `0.15` | Seconds after leaving the ground during which a jump press still fires. |
| `JumpBufferTime` | `0.12` | Seconds before landing during which an airborne press still fires on land. |
| `ApexGravityMultiplier` | `0.5` | Multiplier applied to gravity while at the apex (e.g. `0.5` = half gravity). |
| `ApexGravityMinVelocity` | `12` | Absolute Y-velocity below which the character counts as "at the apex". |
| `LandingFXMinHeight` | `10` | Minimum fall distance (studs) before landing VFX spawns. |

Note: `SetConfig` does **not** live-update sub-modules that are already running. The new values are pushed in when you re-toggle the module (any `Set`/`SetEnabled`/`SetAll` call that flips its enabled state).

---

### CoyoteTime

Config keys: `CoyoteTime`.

When enabled, every frame the player is on the floor the module refreshes a `lastGrounded` timestamp. If the player leaves the ground, presses jump within `CoyoteTime` seconds, and hasn't already spent their airborne jump, GameFeel manually fires `Humanoid:ChangeState(Jumping)`.

Connections: `RunService.Heartbeat` (floor check), `Humanoid.StateChanged` (to track jumps triggered from elsewhere), `UserInputService.JumpRequest` (the grace-window fire).

### JumpBuffer

Config keys: `JumpBufferTime`.

When enabled, a `JumpRequest` made while airborne stores `os.clock()` as `lastJumpRequest`. Every `Heartbeat` the module checks whether there's a buffered press that hasn't expired and the player is now grounded; if so, it fires `Humanoid:ChangeState(Jumping)` and clears the buffer. Grounded presses fall through to Roblox's normal jump handler.

### ApexGravity

Config keys: `ApexGravityMultiplier`, `ApexGravityMinVelocity`.

Snapshots `workspace.Gravity` at module-require time as `origGravity`. While enabled, every `Heartbeat` checks whether the character is airborne **and** `math.abs(AssemblyLinearVelocity.Y) < ApexGravityMinVelocity`. If yes, sets `workspace.Gravity = origGravity * ApexGravityMultiplier`; otherwise restores `origGravity`.

### LandingFX

Config keys: `LandingFXMinHeight`.

Tracks the character's last grounded `Vector3` position. On the frame the character touches ground, if the vertical drop (`lastGrounded.Y - hrp.Position.Y`) exceeds `LandingFXMinHeight`, raycasts down from the HumanoidRootPart up to 7 studs to pin the effect to the actual surface, then calls `VFX.newVFX({ VFXName = "Landing", Position = pos })`. Requires a `Landing` entry registered in the shared `VFX` service.

## Examples

Enable all four features with a single config pass:

```lua
local GameFeel = require(game:GetService("ReplicatedStorage")
    :WaitForChild("Modules")
    :WaitForChild("Client")
    :WaitForChild("GameFeel"))

GameFeel:Init()

GameFeel:SetConfig({
    CoyoteTime = 0.2,
    JumpBufferTime = 0.15,
    ApexGravityMultiplier = 0.45,
    ApexGravityMinVelocity = 14,
    LandingFXMinHeight = 12,
})

GameFeel:SetEnabled({
    CoyoteTime  = true,
    JumpBuffer  = true,
    ApexGravity = true,
    LandingFX   = true,
})
```

Flip a single feature off at runtime:

```lua
GameFeel:Set("ApexGravity", false)
```

Reset everything back to defaults:

```lua
GameFeel:SetConfig(nil)
GameFeel:SetAll(false)
```

## Gotchas

- **`ApexGravity` mutates `workspace.Gravity` globally.** It writes to `workspace.Gravity` every `Heartbeat` while enabled. If anything else in your game reads or writes `workspace.Gravity`, it will fight with this module. Additionally, `origGravity` is captured once at module-require time, so the "normal" value is whatever `workspace.Gravity` was at that moment, not Roblox's default `196.2`. The effect runs on the client's `Heartbeat`, so only the `LocalPlayer` sees the altered gravity.
- **`SetConfig` doesn't live-update running modules.** The shared `Config` is re-read by a sub-module only when its toggle function is called. If a feature is already enabled and you change its config, re-toggle it (`Set` it off then on) to push the new values in.
- **Respawn handling is partial.** Each sub-module hooks `Players.LocalPlayer.CharacterAdded` to swap its `chr`/`hum`/`hrp` references, but existing connections (e.g. `Humanoid.StateChanged` in CoyoteTime) remain bound to the old humanoid across respawns and are only replaced when the feature is toggled off and back on. If your game relies on these features post-respawn, consider calling `GameFeel:SetAll(false); GameFeel:SetAll(true)` on `CharacterAdded`.
- **`LandingFX` requires a registered `Landing` VFX.** It calls `Shared.Services.VFX.newVFX({ VFXName = "Landing", ... })`. If no VFX is registered under that name, nothing spawns.
- **Jumps fire via `Humanoid:ChangeState(Jumping)`.** Both CoyoteTime and JumpBuffer bypass the Humanoid's normal `JumpCooldown`.
- **`SetEnabled` is authoritative, not additive.** Any known module you omit from the table gets disabled. Use `Set` for single-feature changes.
- **`SetConfig(nil)` is a silent full reset.** It replaces `Config` with a fresh clone of the defaults. Prefer being explicit.

## Related

- `ReplicatedStorage/Shared/Components/Services/VFX/` - the VFX service `LandingFX` calls into via `Shared.Services.VFX`.
- `docs/reviews/client-modules-review.md` - open review items for this module (respawn rebinding, shared character-context helper, gravity restore-on-disable).
- `CLAUDE.md` - framework conventions for client modules and the `Modules/Client/` tree.
