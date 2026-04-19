# a_cxnfusing_framework

A batteries-included Roblox Luau framework for beginners and senior devs alike.

Drop it into a Rojo project, require `Framework`, and get a loader plus utilities, functions, info tables, services, network events, and tag-discovered modules wired up on both client and server.

Version `1.2.0 - Open Source, Added Utilities`. MIT licensed. Originally by [cxnfusion](https://github.com/cxnfusingworld).

---

## What's in the box

- **Framework loader** with `:WaitUntilLoaded()`, `:IsServer`/`:IsClient`/`:IsStudio`, `:Log`, `:SafeCall`, `:Preload`.
- **Network events** — Global (RemoteEvent wrapper with invoke) and Local (Signal).
- **Utilities** — Bezier, Spring, StateMachine, Observer, Trove, Signal, Promise, Pool, CameraShaker, FlashController, DecalProjector, SFXHandler, Imagine, ViewCheck, LODManager, Ragdoll, ConnectToAdded, ConnectToTag, ActionBuffer, DebugUtils, TimeFormatter, IllusionIAS, ExtendedLibraries (math/Random/Vector3/table), SpectateService.
- **Functions** — FastTween, FormatNumber, AddCommas, GenerateUID.
- **Info** — PlayerState, GameState.
- **Services** — VFX, DataService (ProfileStore-backed), DialogueService (typewriter + effects + reply trees).
- **Client modules** — TagLoader (filename-as-tag presets, animated textures, UI animate), ExpressivePrompts (keyboard/gamepad/touch input prompts, Seam-powered), GameFeel (coyote time, jump buffer, apex gravity, landing FX).
- **Tag-loaded modules** — attach the `LoadModule` CollectionService tag to any ModuleScript and it gets required automatically.

---

## Install

### Rojo (recommended)

1. Install [Aftman](https://github.com/LPGhatguy/aftman) and run `aftman install` in the repo root. This pulls Rojo, StyLua, and Selene at the versions pinned in `aftman.toml`.
2. Open Studio, install the Rojo plugin, and point it at `default.project.json`.
3. Run `rojo serve default.project.json` and click Connect in Studio.

### Studio-only

Open `Framework.rbxlx` directly in Roblox Studio. Every module is already placed.

---

## Quickstart

From any `Script` or `LocalScript`:

```lua
local rs = game:GetService("ReplicatedStorage")
local Framework = require(rs:WaitForChild("Framework"))
Framework:WaitUntilLoaded()

print("framework ready, version " .. Framework.Version)

Framework:Log("Boot", "Hello from the framework!")
```

From a ModuleScript that is itself loaded by the framework, require `Shared` instead (no dependencies, no load wait):

```lua
local Shared = require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"))
local Trove = Shared.Utilities.Trove
```

---

## Module index

| Module | What it does | Docs |
| --- | --- | --- |
| Framework | Loader, helpers, require surface for user code | [docs/modules/framework.md](docs/modules/framework.md) |
| Shared | Leaf require surface for framework-loaded modules | [docs/modules/shared.md](docs/modules/shared.md) |
| Network | Global + Local events, SignalWrapper | [docs/modules/network.md](docs/modules/network.md) |
| Info | PlayerState, GameState | [docs/modules/info.md](docs/modules/info.md) |
| Functions | FastTween, FormatNumber, AddCommas, GenerateUID | [docs/modules/functions.md](docs/modules/functions.md) |
| VFX | Framework's VFX service | [docs/modules/vfx.md](docs/modules/vfx.md) |
| DataService | ProfileStore-backed player data | [docs/modules/dataservice.md](docs/modules/dataservice.md) |
| Util - Motion | Bezier, Spring, StateMachine, Observer | [docs/modules/util-motion.md](docs/modules/util-motion.md) |
| Util - Collections | Trove, Signal, Promise, Pool | [docs/modules/util-collections.md](docs/modules/util-collections.md) |
| Util - Input | ConnectToAdded, ConnectToTag, ActionBuffer | [docs/modules/util-input.md](docs/modules/util-input.md) |
| Util - Visual | CameraShaker, FlashController, DecalProjector, SFXHandler, Imagine, ViewCheck, LODManager, Ragdoll | [docs/modules/util-visual.md](docs/modules/util-visual.md) |
| Util - Misc | DebugUtils, TimeFormatter, IllusionIAS, ExtendedLibraries, SpectateService | [docs/modules/util-misc.md](docs/modules/util-misc.md) |
| DialogueService | Typewriter + effects + reply trees | [docs/modules/dialogueservice.md](docs/modules/dialogueservice.md) |
| TagLoader | Filename-as-tag client handlers (Preset, UIAnimate, AnimatedTexture) | [docs/modules/tagloader.md](docs/modules/tagloader.md) |
| ExpressivePrompts | Animated keyboard/gamepad/touch input prompts | [docs/modules/expressiveprompts.md](docs/modules/expressiveprompts.md) |
| GameFeel | CoyoteTime, JumpBuffer, ApexGravity, LandingFX | [docs/modules/gamefeel.md](docs/modules/gamefeel.md) |

---

## Documentation

- [docs/index.md](docs/index.md) — documentation home / navigation
- [docs/getting-started.md](docs/getting-started.md) — zero-to-booting walkthrough
- [docs/architecture.md](docs/architecture.md) — how the loader, Shared, and tag-loaded modules fit together
- [docs/modules/](docs/modules/) — per-module reference pages
- [docs/reviews/](docs/reviews/) — internal code-quality notes and follow-up fix bullets (not yet applied)

---

## Credits

Docs written by [SirBepy](https://github.com/SirBepy)

Originally by [cxnfusion](https://github.com/cxnfusingworld). The repo vendors a few third-party libraries — see headers in each vendored file for original authors (ProfileStore by loleris, CameraShaker by Sleitnick, Seam UI library by miagobble, among others).
