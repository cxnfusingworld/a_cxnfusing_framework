# Documentation

Welcome to the `a_cxnfusing_framework` docs. This is a Roblox Luau framework you can drop into a Rojo project to get a loader, utilities, services, and network events wired up on both client and server.

## Start here

- [Getting started](getting-started.md) — install and boot the framework from scratch
- [Architecture](architecture.md) — how Framework, Shared, and tag-loaded Modules compose at runtime

## Modules

| Module | Summary |
| --- | --- |
| [Framework](modules/framework.md) | Loader + helpers + require surface for user code |
| [Shared](modules/shared.md) | Leaf require surface for framework-loaded ModuleScripts |
| [Network](modules/network.md) | Global + Local events |
| [Info](modules/info.md) | PlayerState, GameState |
| [Functions](modules/functions.md) | FastTween, FormatNumber, AddCommas, GenerateUID |
| [VFX](modules/vfx.md) | Framework VFX service |
| [DataService](modules/dataservice.md) | ProfileStore-backed player data |
| [Util - Motion](modules/util-motion.md) | Bezier, Spring, StateMachine, Observer |
| [Util - Collections](modules/util-collections.md) | Trove, Signal, Promise, Pool |
| [Util - Input](modules/util-input.md) | ConnectToAdded, ConnectToTag, ActionBuffer |
| [Util - Visual](modules/util-visual.md) | CameraShaker, FlashController, DecalProjector, SFXHandler, Imagine, ViewCheck, LODManager, Ragdoll |
| [Util - Misc](modules/util-misc.md) | DebugUtils, TimeFormatter, IllusionIAS, ExtendedLibraries, SpectateService |
| [DialogueService](modules/dialogueservice.md) | Typewriter + effects + reply trees |
| [TagLoader](modules/tagloader.md) | Filename-as-tag client handlers |
| [ExpressivePrompts](modules/expressiveprompts.md) | Animated input prompts (keyboard/gamepad/touch) |
| [GameFeel](modules/gamefeel.md) | CoyoteTime, JumpBuffer, ApexGravity, LandingFX |

## Reviews

`docs/reviews/` contains internal quality notes generated during the open-sourcing pass. Each file scores modules on clarity, robustness, API design, noob-friendliness, and style consistency, and lists concrete fixes that have NOT yet been applied:

- [Framework core review](reviews/framework-review.md)
- [Shared review](reviews/shared-review.md)
- [Services review](reviews/services-review.md)
- [Modules/Client review](reviews/client-modules-review.md)

Use these as a follow-up backlog. If you're a contributor, consider picking a fix bullet and sending a PR.
