# Modules/Client Review

## Summary
The client-side modules ship useful, demo-worthy features (tag-driven UI animation, animated textures, platformer game-feel helpers, and a polished custom ProximityPrompt UI built on Seam), but quality varies sharply between areas. TagLoader is minimal and brittle, quietly coupling every child module to the `ConnectToTag` contract and using filenames as tag identifiers with no docs. GameFeel has a clean top-level API but its per-feature modules all duplicate character/humanoid boilerplate, leak connections on respawn, and mutate `workspace.Gravity` globally. ExpressivePrompts is the most senior-developer code here but is dense, under-commented, uses `any` types everywhere, and has a noticeable reliability bug (sound-cache recreates sounds on collision). None of the modules are ready for new programmers to copy from without a pass for naming, comments, and robustness.

## Modules

### TagLoader
Path: `src/ReplicatedStorage/Modules/Client/TagLoader/`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 5 - The loop in `init.luau` is tiny, but the contract (every child module must return `(obj)->()` and its filename must equal a CollectionService tag) is undocumented.
- Robustness: 4 - No disconnect, no attribute-change response, no error wrapping, `Preset` silently no-ops if `Presets` folder is empty, and `Imagine.apply` can return nil but callers assume a table.
- API design: 4 - `UIAnimate` stores throwaway state as attributes prefixed `_clicked` and `Original*` directly on user instances, polluting them; `Imagine:Unapply` wipes `self` but is never wired up.
- Noob-friendliness: 3 - No README or header comment explains which tags exist or what attributes they read; a new user has to read every file to learn the attribute schema.
- Style consistency: 5 - Mixes `rs`/`ts` short locals with PascalCase, `game["Run Service"]` string indexing, and inconsistent spacing; drifts from the rest of the framework's style.

Fixes:
- [ ] `TagLoader/init.luau:8` - rename parameter `mod` to `moduleScript` and add a top-of-file comment explaining the "filename == tag name" contract.
- [ ] `TagLoader/init.luau:13` - wrap `require(mod)` in `pcall` and log via `Shared.Log` so a bad child module does not kill `:Start()`.
- [ ] `TagLoader/init.luau:19` - `:Start()` iterates children including `Preset` twice because `initTag(script:WaitForChild("Preset"))` runs first then the loop re-hits it; rely on the `loaded` guard or just remove the pre-call and document ordering.
- [ ] `TagLoader/init.luau:21` - filter to `mod:IsA("ModuleScript")` before calling `initTag`.
- [ ] `TagLoader/Preset.luau:15` - annotation `{[string]:any}` is wrong for a Folder/Configuration Instance; retype as `Instance?` and rename `f` to `presetInstance`.
- [ ] `TagLoader/Preset.luau:23` - document that this module deletes the `Preset` attribute after applying and explain it is one-shot.
- [ ] `TagLoader/UIAnimate.luau:46` - stop using `Instance.new("UIScale", obj)` (deprecated parent-in-constructor); assign `Parent` after construction.
- [ ] `TagLoader/UIAnimate.luau:62` - `getResetValue` returns `0` for unknown types when `typeName == nil`; the `nil` branch is unreachable and misleading, delete it.
- [ ] `TagLoader/UIAnimate.luau:68` - replace the `_clicked` attribute pseudo-lock with a weak table keyed by the instance so user attribute names are not polluted.
- [ ] `TagLoader/UIAnimate.luau:103` - `task.wait(0.3)` hard-codes the click cooldown instead of deriving from `config.Click + config.Return`; compute it.
- [ ] `TagLoader/UIAnimate.luau:80` - connections are never disconnected when the GuiObject is destroyed, causing memory leaks; use a Trove or `obj.Destroying` cleanup.
- [ ] `TagLoader/AnimatedTexture/init.luau:3` - validate the instance is an `ImageLabel`/`ImageButton` with `ResampleMode = Tile` and warn otherwise.
- [ ] `TagLoader/AnimatedTexture/Imagine/init.luau:19` - `Imagine.apply` returns `nil` when `base` is nil, but the return type annotation is `Image`; either error or change the type.
- [ ] `TagLoader/AnimatedTexture/Imagine/init.luau:44` - replace `game["Run Service"].RenderStepped` with `game:GetService("RunService").RenderStepped`.
- [ ] `TagLoader/AnimatedTexture/Imagine/init.luau:4` - shadowing the global `table` with `local table = require(...)` is hostile; rename the local to `TableExt`.
- [ ] `TagLoader/AnimatedTexture/Imagine/init.luau:62` - `Unapply` clears `self` then sets metatable nil; wipe connections first then clear the table to avoid iterating a mutating table.
- [ ] `TagLoader/AnimatedTexture/Imagine/TableExtended.luau:1` - the `lib = table.clone(table)` + `__index = lib` construction is unused because `extended` only defines `merge`; drop the cloned library and just return `{ merge = ... }`.

### GameFeel
Path: `src/ReplicatedStorage/Modules/Client/GameFeel/`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - The outer `init.luau` API is readable, but each child module repeats the same Character/Humanoid bootstrap pattern inline with no helper.
- Robustness: 4 - `ApexGravity` mutates global `workspace.Gravity` every heartbeat and captures `origGravity` once at require time so later authoritative changes get overwritten; `conns` is not cleared on respawn so connections leak; JumpRequest-based buffering will not catch gamepad A if UIS is sunk.
- API design: 6 - `SetEnabled`, `Set`, `SetAll`, `SetConfig` are fine for a flat feature toggle system, but `SetConfig(nil)` resetting the global config is an unsignposted footgun.
- Noob-friendliness: 5 - Variable names `chr`, `hum`, `hrp`, `uis`, `conns`, `upd` are opaque to beginners and there are no comments describing the actual platformer mechanic each module implements.
- Style consistency: 6 - Each module follows the same shape, which is good, but the shape itself duplicates ~15 lines of boilerplate per file.

Fixes:
- [ ] `GameFeel/init.luau:24` - rename parameter `data` to `enabledByName` and document the key names.
- [ ] `GameFeel/init.luau:64` - make the `nil` reset explicit via a separate `GameFeel:ResetConfig()` method so `SetConfig(nil)` is not a hidden side door.
- [ ] `GameFeel/init.luau:66` - silently ignoring unknown config keys is bug-prone; warn on unknown keys instead.
- [ ] `GameFeel/init.luau:78` - ensure `:Init` skips folders or non-ModuleScript siblings (already does via `IsA`, but also log which modules were registered).
- [ ] `GameFeel/CoyoteTime.luau:5` - extract the shared Character/Humanoid/CharacterAdded bootstrap to a local `GameFeel/_CharacterContext.luau` helper and require it from each feature.
- [ ] `GameFeel/CoyoteTime.luau:31` - unused `inp, proc` parameters on `keyCheck`; use `_, _` or remove them (they come from JumpRequest which passes nothing, so remove entirely).
- [ ] `GameFeel/CoyoteTime.luau:65` - on respawn the `hum.StateChanged` connection is bound to the old humanoid and never refreshed; rebind on `CharacterAdded`.
- [ ] `GameFeel/ApexGravity.luau:13` - cache `workspace.Gravity` only when `enabled` toggles true, and restore it on toggle-off; currently `origGravity` is captured once at require time.
- [ ] `GameFeel/ApexGravity.luau:22` - write to `workspace.Gravity` only when the value actually changes to avoid every-frame replication noise.
- [ ] `GameFeel/ApexGravity.luau:31` - after respawn `hrp` is taken immediately; on new characters `PrimaryPart` may be nil, wait for `chr:WaitForChild("HumanoidRootPart")`.
- [ ] `GameFeel/JumpBuffer.luau:34` - buffered jump only fires on Heartbeat after grounding; on the first frame of landing you may still be `Enum.Material.Air`, so also listen to `GetPropertyChangedSignal("FloorMaterial")`.
- [ ] `GameFeel/JumpBuffer.luau:40` - `CharacterAdded` reassigns `hum` but `conns` are still bound to the previous humanoid; rebind or at least clear `lastJumpRequest`.
- [ ] `GameFeel/LandingFX.luau:8` - `lastGrounded = hrp.Position` at module-require time assumes the character exists; this will error if required before `CharacterAdded`.
- [ ] `GameFeel/LandingFX.luau:35` - magic numbers `-3`, `7`, `0.1` for ray offsets; pull into named locals.
- [ ] `GameFeel/LandingFX.luau:49` - on respawn reset `lastGrounded = newChr.PrimaryPart.Position` and `fxShown = true` to avoid a phantom landing VFX on spawn.

### ExpressivePrompts
Path: `src/ReplicatedStorage/Modules/Client/ExpressivePrompts/`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - The Seam-based declarative style is concise once you know Seam, but there are almost no inline comments explaining what each spring/state represents; a noob reading this cold will be lost.
- Robustness: 5 - `PlaySound` caches by `SoundId` in SoundService but never awaits the previous instance, so rapid `PlaySound(APPEAR_SOUND)` calls play mid-play sounds correctly; however the cache uses `Scope:New(Scope:New(...))` (nested), which parents the sound twice and is almost certainly wrong. `NewInputConnections` reads `SoundService:FindFirstChild(HOLD_SOUND).PlaybackSpeed` each RenderStepped without nil-checking even though the sound is only created on `PromptButtonHoldBegan`, crashing if Heartbeat fires before the hold handler.
- API design: 7 - `ExpressivePrompts.Config` as a table of Seam values is a nice reactive surface; `:Init` self-guards against double-init; cleanup function returned from `CreatePrompt` is good.
- Noob-friendliness: 4 - Heavy Seam idioms (`Scope:Value`, `:Spring`, `:Computed`, `Use(...)`) with zero pointers to Seam docs; `any` type aliases everywhere; mixed camelCase and PascalCase parameter names within the same file.
- Style consistency: 5 - `BuildTextLabels.luau` and `UpdateUIFromPrompt.luau` use 4-space indent while the rest of the repo uses tabs; parameters toggle between `prompt`/`Prompt` inconsistently.

Fixes:
- [ ] `ExpressivePrompts/init.luau:40` - `game.Loaded:Wait()` inside a `ModuleScript` at require time yields the loader; move this into `:Init` so it does not block other client modules.
- [ ] `ExpressivePrompts/init.luau:54` - `Scope = Seam.Scope(Seam)` is never destroyed; if `:Init` is conceptually re-entrant (it warns on double init but still), document the lifetime.
- [ ] `ExpressivePrompts/init.luau:80` - replace `GetUpdateCallback(...)` + `unpack` indirection with a direct closure; `unpack` with varargs hides intent.
- [ ] `ExpressivePrompts/init.luau:118` - destructuring via `unpack(BuildFrames(...))` is fragile; return a named table `{Frame=..., ResizeableInputFrame=...}` from `BuildFrames`.
- [ ] `ExpressivePrompts/init.luau:190` - mutating `InputConnections` after it was returned from `NewInputConnections` breaks the module's contract; have `NewInputConnections` accept the Prompt and own this connection internally.
- [ ] `ExpressivePrompts/init.luau:202` - hard-coded 2-second cleanup delay should be derived from the spring settle time or named as a constant `PROMPT_FADE_OUT_TIME`.
- [ ] `ExpressivePrompts/init.luau:222` - no `Player:WaitForChild("PlayerGui")` timeout; pass a timeout and warn on failure.
- [ ] `ExpressivePrompts/BuildFrames.luau:1` - parameter list has 9 positional args; convert to a single `options` table and add a type alias.
- [ ] `ExpressivePrompts/BuildFrames.luau:20` - `Scope:Spring(PromptTransparency, 30, 1)` uses magic numbers that should use the same config pattern as the rest of the file.
- [ ] `ExpressivePrompts/BuildTextLabels.luau:2` - re-indent file with tabs to match repo style.
- [ ] `ExpressivePrompts/BuildTextLabels.luau:5` - rename `ActionText`/`ObjectText` return table to a named table `{Action=..., Object=...}` to avoid index-0-vs-1 confusion in callers.
- [ ] `ExpressivePrompts/NewInputConnections.luau:25` - `Sound = Scope:New(Scope:New("Sound", {...}), Properties or {})` creates and then re-wraps the same instance, likely a typo; simplify to a single `Scope:New("Sound", merged)`.
- [ ] `ExpressivePrompts/NewInputConnections.luau:57` - nil-guard `SoundService:FindFirstChild(HOLD_SOUND)` before indexing `.PlaybackSpeed`.
- [ ] `ExpressivePrompts/NewInputConnections.luau:46` - `HoldStart = os.clock()` is captured before any hold has started, so the first `RenderStepped` reads a stale value until `PromptButtonHoldBegan`; initialize to `math.huge` or guard on `ButtonHeldDown.Value`.
- [ ] `ExpressivePrompts/NewInputConnections.luau:88` - `task.delay(1/60, ...)` is a single-frame hack for the tap flash; use `RunService.RenderStepped:Once`.
- [ ] `ExpressivePrompts/UpdateUIFromPrompt.luau:2` - re-indent file with tabs to match repo style.
- [ ] `ExpressivePrompts/UpdateUIFromPrompt.luau:20` - magic constants `72`, `72`, `72`, `24`, `9`, `-10` all describe the prompt layout; hoist to named locals at file top.
- [ ] `ExpressivePrompts/UpdateUIFromPrompt.luau:51` - dividing `Prompt.UIOffset.X / PromptUI.Size.Width.Offset` will NaN if `Offset` is 0; guard.
- [ ] `ExpressivePrompts/NewInputLabel/init.luau:8` - parameter `inputType` is lowercase while the file's other parameters are PascalCase; pick one.
- [ ] `ExpressivePrompts/NewInputLabel/Gamepad/init.luau:5` - silently returning `nil` when the gamepad keycode is unknown means the prompt renders nothing; return a fallback label or warn.
- [ ] `ExpressivePrompts/NewInputLabel/Keyboard/init.luau:58` - `error(...)` on unsupported keycode hard-crashes the prompt pipeline; warn + render a placeholder instead.
- [ ] `ExpressivePrompts/NewInputLabel/Keyboard/init.luau:51` - `string.len(buttonTextString)` is ASCII-length; use `utf8.len` for CJK keycaps.
- [ ] `ExpressivePrompts/NewInputLabel/Touch.luau:1` - file lives flat while Keyboard/Gamepad are folders; either move it into `Touch/init.luau` for consistency or document the asymmetry.

## Cross-cutting Issues
- Character/Humanoid bootstrap (`plr`, `chr`, `hum`, `hrp`, `CharacterAdded`) is copy-pasted across all four GameFeel features; a shared `CharacterContext` helper would remove ~60 lines and fix the respawn-rebinding bugs once.
- Every GameFeel module and `TagLoader/UIAnimate.luau` leak `RBXScriptConnection`s: `conns` is not cleared when the player respawns, only when the feature is toggled off. Standardize on a Trove or janitor per feature.
- The repo's `stylua.toml` specifies tabs, but `BuildTextLabels.luau`, `UpdateUIFromPrompt.luau`, and parts of `NewInputLabel/*` use 4-space indents. Run StyLua across `Modules/Client/ExpressivePrompts/**`.
- Type annotations are applied inconsistently: `: any` is used liberally in ExpressivePrompts to paper over Seam types, `{[string]:any}` is mis-applied in `TagLoader/Preset.luau`, and GameFeel uses `:{}`. Either commit to typed Seam wrappers or strip type annotations entirely per the repo's stated "not uniformly typed" stance.
- Attribute-based config (TagLoader) and table-based config (GameFeel, ExpressivePrompts) live side-by-side with no documented schema; new users have no discoverable list of supported attributes for tag-loaded instances. Add a per-module attribute table in the file header.
- Naming style varies within single files: `plr/chr/hum/hrp/uis/ts/rs` short locals in GameFeel vs PascalCase everywhere in ExpressivePrompts vs mixed camel+Pascal parameters in `NewInputLabel/init.luau` (`inputType`, `prompt`, `Config`). Pick one and apply repo-wide.
- Sound/VFX assets are hardcoded by numeric asset ID (`ExpressivePrompts/NewInputConnections.luau`, `LandingFX` VFX name) without a central registry, making them un-swappable without editing source.
- No module in this area disconnects cleanly on teardown, so hot-reloading in Studio will stack connections. Every top-level module should expose a `:Destroy()` or Trove.
