# Shared Review

## Summary
The Shared surface is feature-rich but uneven: core aggregation (`init.luau`, `Network/`) is tidy, and several small utilities (`Bezier`, `Trove`, `Spring`, `TimeFormatter`, `ExtendedLibraries`) read well. However, many utilities ship with broken boilerplate (stray `::` type blocks in `Info/GameState.luau` and `Util/FlashController.luau`, a `LODManager` init loop with a wrong math expression, early returns that return non-table sentinels like `true`, duplicate vendored SignalPlus copies in two locations, left-in `print("DEBUG: ...")` calls in `Pool/Utils.luau` and `Pool/Validation.luau`). Several services (`VFX`, `DataService`, `SFXHandler`) assume assets and folders that are not documented or defaulted, which will trap new users. Naming conventions are inconsistent (mixed `PascalCase`/`camelCase`/`snake-ish`), type annotations are partial, and error handling in user-facing methods silently returns early instead of warning. With targeted fixes this surface is usable, but today it would confuse new programmers badly.

## Modules

### Shared entry (`init.luau`)
Path: `src/ReplicatedStorage/Shared/init.luau`

Scores (1-10):
- Clarity: 7 - layout with `---<>---` section headers is readable but non-standard.
- Robustness: 5 - assumes `ReplicatedStorage.Assets` and a `Data` child exist; will yield forever if missing.
- API design: 6 - flat aggregate table is fine, but exposes `Debris` as a service-like field and re-exports `TweenService` only indirectly.
- Noob-friendliness: 6 - no comments explaining what to edit when adding events/utilities (docs live elsewhere).
- Style consistency: 7 - matches overall casual voice.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/init.luau:30` - use `rs:FindFirstChild("Assets")` with a clear warning if absent, instead of indefinitely yielding `WaitForChild`.
- [ ] `src/ReplicatedStorage/Shared/init.luau:31` - drop `Data = script:WaitForChild("Data")` or document/create the expected `Data` child; currently yields forever when absent.
- [ ] `src/ReplicatedStorage/Shared/init.luau:36` - remove `Debris` from the top-level table or move it under `Utilities`, so service handles are grouped.
- [ ] `src/ReplicatedStorage/Shared/init.luau:50` - read the version string from a single source instead of hard-coding it in the shared aggregate.
- [ ] `src/ReplicatedStorage/Shared/init.luau:54` - rename `freeze` helper to `tryFreeze`; add a skip for tables that look like signals (`Fire`+`Connect`) to match the documented invariant in `CLAUDE.md`.

### Config (`Config.luau`)
Path: `src/ReplicatedStorage/Shared/Config.luau`

Scores (1-10):
- Clarity: 8 - trivially clear.
- Robustness: 7 - nothing to fail, but no defaults protection against mistyped keys.
- API design: 5 - empty "Game Config" section is dead weight; four near-identical log flags should be an enum/level.
- Noob-friendliness: 7 - easy to read but gives no type hints.
- Style consistency: 7 - matches voice.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Config.luau:3` - delete the empty Game Config section or fill with a commented example.
- [ ] `src/ReplicatedStorage/Shared/Config.luau:9` - collapse `LogsEnabled`, `WarningLogsEnabled`, `ErrorLogsEnabled` into `LogLevel = "info" | "warn" | "error" | "silent"`.
- [ ] `src/ReplicatedStorage/Shared/Config.luau:1` - add an `export type Config = typeof(...)` for downstream autocomplete.

### Network (`Network/init.luau`, `Network/SignalWrapper/*`)
Path: `src/ReplicatedStorage/Shared/Network`

Scores (1-10):
- Clarity: 6 - two-constructor entry point is clean, but `Remote.luau` mixes class body and instance body in one closure.
- Robustness: 5 - `remote.wait()` declared without `self`; player validation exists client-side but is silent; no Destroy on signal connections tracked.
- API design: 6 - Fire/Connect/Once/Wait/OnInvoke cover basics, but mixes method styles (`:Connect` with dot `.OnInvoke`) and silently no-ops on bad args.
- Noob-friendliness: 5 - `Remotes` folder must exist under `Network` for `SignalWrapper/init.luau` to work, but nothing creates it.
- Style consistency: 6 - `conn.new` / `conn.once` / `conn.wait` inconsistent with rest of codebase's `PascalCase` method style.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Network/SignalWrapper/init.luau:11` - auto-create the `Remotes` child folder if missing instead of yielding `WaitForChild`.
- [ ] `src/ReplicatedStorage/Shared/Network/SignalWrapper/Remote.luau:62` - rename `conn.wait` to `conn:Wait` and make it a method for consistency.
- [ ] `src/ReplicatedStorage/Shared/Network/SignalWrapper/Remote.luau:96` - warn (don't silently return) when `FireClient`/`InvokeClient` gets a non-player argument.
- [ ] `src/ReplicatedStorage/Shared/Network/SignalWrapper/Remote.luau:126` - change `function new.OnInvoke` to `function new:SetInvokeCallback` so method-vs-function usage is consistent.
- [ ] `src/ReplicatedStorage/Shared/Network/SignalWrapper/Remote.luau:141` - have `:Destroy` also null out `remote`/`invokeRemote` upvalues to prevent post-destroy use.
- [ ] `src/ReplicatedStorage/Shared/Network/init.luau:11` - drop the `rem :: typeof(rem)` no-op cast; use the inferred type directly.

### Functions index (`Components/Functions/init.luau`)
Path: `src/ReplicatedStorage/Shared/Components/Functions/init.luau`

Scores (1-10):
- Clarity: 8 - one-line registry.
- Robustness: 9 - nothing to go wrong.
- API design: 7 - flat namespace, acceptable.
- Noob-friendliness: 7 - no section docstring.
- Style consistency: 8 - matches pattern.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/init.luau:1` - add a one-line comment describing what "Functions" means vs `Util` so contributors know which folder new helpers belong in.

### AddCommas
Path: `src/ReplicatedStorage/Shared/Components/Functions/AddCommas.luau`

Scores (1-10):
- Clarity: 9 - short, named, typed.
- Robustness: 8 - handles negatives, stops on no-replace.
- API design: 9 - pure function.
- Noob-friendliness: 9 - easy.
- Style consistency: 8 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/AddCommas.luau:3` - use a single `string.gsub` with `"(%d)(%d%d%d)$"` reverse walk, or at least comment why the repeat-loop is needed.

### FastTween
Path: `src/ReplicatedStorage/Shared/Components/Functions/FastTween.luau`

Scores (1-10):
- Clarity: 5 - positional-array TweenInfo is unusual.
- Robustness: 4 - `tweenInfo = table.clone(tweenInfo) or defaultTI` mis-orders the fallback; if `tweenInfo` is nil it errors before the `or`.
- API design: 4 - positional array for easing/direction is harder to use than `TweenInfo.new` directly.
- Noob-friendliness: 4 - two type checks on index `[2]` (lines 22 and 25) means `[3]` can never become an `EasingDirection` from a string.
- Style consistency: 6 - matches casual style.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FastTween.luau:14` - guard with `tweenInfo = tweenInfo and table.clone(tweenInfo) or table.clone(defaultTI)` so nil input does not crash.
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FastTween.luau:25` - change the second coercion to `Enum.EasingDirection[tweenInfo[3]]` - current code maps it to `EasingStyle` again.
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FastTween.luau:11` - accept `TweenInfo` directly and only fall back to the array form if a plain table is passed.
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FastTween.luau:12` - warn on `not obj or not goals` rather than silently returning.

### FormatNumber
Path: `src/ReplicatedStorage/Shared/Components/Functions/FormatNumber.luau`

Scores (1-10):
- Clarity: 8 - straightforward.
- Robustness: 7 - no negative-sign handling for ultra-small edge cases; trailing `.0` strip regex `%.0$` only catches a single trailing zero.
- API design: 8 - fine.
- Noob-friendliness: 8 - readable.
- Style consistency: 8 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FormatNumber.luau:16` - update strip to `"%.?0+$"` so `1.10K` and `1.00K` both collapse to `1.1K`/`1K`.
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/FormatNumber.luau:2` - hoist the `suffixes` array outside the returned function; rebuilding it on every call is wasteful.

### GenerateUID
Path: `src/ReplicatedStorage/Shared/Components/Functions/GenerateUID.luau`

Scores (1-10):
- Clarity: 9 - tiny.
- Robustness: 9 - wraps `HttpService:GenerateGUID`.
- API design: 8 - fine.
- Noob-friendliness: 9 - obvious.
- Style consistency: 7 - mixed case (`GUID`, `final`).

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Functions/GenerateUID.luau:4` - use `string.gsub(GUID, "-", "")` directly in the return; inline drops the redundant `final` local.

### Info index (`Components/Info/init.luau`)
Path: `src/ReplicatedStorage/Shared/Components/Info/init.luau`

Scores (1-10):
- Clarity: 8 - small registry.
- Robustness: 5 - `PlayerState` has hard client-only return, `GameState` is empty/broken.
- API design: 6 - exposes state tables that mutate; the outer `table.freeze` in Shared will try to freeze them (saved by the signal-detect rule, but `GameState` has no such marker).
- Noob-friendliness: 5 - not obvious that `Info.PlayerState` is `true` on the server.
- Style consistency: 7 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Info/init.luau:1` - document at the top that these tables are per-context (client vs server) and may not exist in both.

### GameState
Path: `src/ReplicatedStorage/Shared/Components/Info/GameState.luau`

Scores (1-10):
- Clarity: 2 - the file is syntactically weird - a second `{}` literal after `::` that looks like a failed type annotation.
- Robustness: 3 - returns an empty table but has a dangling `::{}` block; likely breaks strict Luau parsing.
- API design: 3 - empty.
- Noob-friendliness: 2 - a new contributor cannot tell what this is supposed to hold.
- Style consistency: 3 - breaks the rest of the codebase's shape.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Info/GameState.luau:1` - delete the dangling `::\n{ }` and write the file as `return {} :: { [string]: any }` with a comment describing intended contents.
- [ ] `src/ReplicatedStorage/Shared/Components/Info/GameState.luau:1` - either populate with real default fields or remove GameState from `Info/init.luau` until it is a real module.

### PlayerState
Path: `src/ReplicatedStorage/Shared/Components/Info/PlayerState.luau`

Scores (1-10):
- Clarity: 6 - mixes type annotation hack `true :: data` with real implementation.
- Robustness: 4 - on server returns `true`, which downstream code using `Info.PlayerState.Character` will index-on-boolean and crash.
- API design: 5 - exposes a mutable shared table keyed off `LocalPlayer`.
- Noob-friendliness: 4 - the client-only branch is invisible until the server crashes.
- Style consistency: 6 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Info/PlayerState.luau:4` - return an empty frozen stub `{}` on the server instead of `true`, so server-side accidental use produces a clearer missing-field error.
- [ ] `src/ReplicatedStorage/Shared/Components/Info/PlayerState.luau:32` - disconnect the `plr.CharacterAdded` on player leaving to avoid a leak if this module is ever re-required (Luau caches modules, but safe-guard anyway).
- [ ] `src/ReplicatedStorage/Shared/Components/Info/PlayerState.luau:19` - use a `Trove` or explicit `Connections` cleanup helper instead of an empty `Connections` table that nothing populates.

### Util index (`Components/Util/init.luau`)
Path: `src/ReplicatedStorage/Shared/Components/Util/init.luau`

Scores (1-10):
- Clarity: 8 - list of requires.
- Robustness: 9 - nothing runtime-risky.
- API design: 6 - namespace is big and flat; noobs won't know which of 20 utilities to use.
- Noob-friendliness: 5 - no grouping or description.
- Style consistency: 8 - matches pattern.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/init.luau:1` - add a block comment grouping utilities by purpose (signal/async, math, instance lifecycle, camera/fx).

### ActionBuffer
Path: `src/ReplicatedStorage/Shared/Components/Util/ActionBuffer.luau`

Scores (1-10):
- Clarity: 7 - small.
- Robustness: 5 - `IsActive` sets timestamp to `0` (still truthy, still a number) rather than `nil`, so a stale `0` will make all future calls return `0 - os.clock()` truth-test-wise; but because `(os.clock() - 0) <= window` is false for large os.clock, behavior is still okay - the magic value is confusing though.
- API design: 6 - no way to peek without consuming.
- Noob-friendliness: 5 - name vs behavior: `IsActive` is really "ConsumeIfActive".
- Style consistency: 7 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ActionBuffer.luau:18` - set `storedActions[actionName] = nil` instead of `0` after consume, and rename `IsActive` to `Consume` (or add a separate `Peek`).
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ActionBuffer.luau:4` - make `EXPIRE_TIME` overridable on a per-module basis via `ActionBuffer.DefaultWindow`.

### Bezier
Path: `src/ReplicatedStorage/Shared/Components/Util/Bezier.luau`

Scores (1-10):
- Clarity: 9 - two functions, commented.
- Robustness: 8 - handles `n < 2` and `n == 2`; `GetSmoothSpline` clamps endpoints.
- API design: 8 - accepts a points array and `t`.
- Noob-friendliness: 8 - docstrings help.
- Style consistency: 8 - good.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Bezier.luau:18` - replace `unpack` with `table.unpack` (or `table.clone(points)`) to avoid relying on the global `unpack` alias.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Bezier.luau:22` - drop the trailing `table.remove` on the deCasteljau loop; the loop already shrinks logically - just maintain a `count` instead of shrinking the array each pass.

### CameraShaker (`CameraShaker/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/CameraShaker`

Scores (1-10):
- Clarity: 7 - Sleitnick code, documented inline.
- Robustness: 7 - proven; uses `#instances` loops that can miss mid-iteration removals but handles it via a delete list.
- API design: 7 - solid class API.
- Noob-friendliness: 7 - top-of-file usage example.
- Style consistency: 5 - doesn't match rest of the codebase (uses `self._foo` and `;` line endings).

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/CameraShaker/init.luau:1` - add an author-attribution comment noting this is vendored (per repo CLAUDE.md the author didn't write it), so contributors don't restyle it.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/CameraShaker/init.luau:32` - replace the legacy `wait(1)` in the docstring example with `task.wait(1)`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/CameraShaker/CameraShakePresets.luau:97` - the `__index` metamethod runs `f()` every access, producing a *new* preset each time; document this or cache, since callers expect `Presets.Explosion` to be a preset and calling `:Shake()` later with the same value mutates surprisingly.

### ConnectToAdded
Path: `src/ReplicatedStorage/Shared/Components/Util/ConnectToAdded.luau`

Scores (1-10):
- Clarity: 7 - small.
- Robustness: 5 - unused `cs` local; silently returns nil when args bad; doesn't return a disconnector for the "call on existing" pass.
- API design: 6 - passing `useDescendants: boolean` as a positional flag is less readable than a config.
- Noob-friendliness: 6 - fine.
- Style consistency: 7 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ConnectToAdded.luau:1` - remove the unused `cs = game:GetService("CollectionService")` import.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ConnectToAdded.luau:3` - swap the positional boolean for `opts: { useDescendants: boolean? }` so callers don't misread truthy values.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ConnectToAdded.luau:4` - warn (don't silently `return`) when `obj` or `callback` is nil.

### ConnectToTag
Path: `src/ReplicatedStorage/Shared/Components/Util/ConnectToTag.luau`

Scores (1-10):
- Clarity: 8 - straightforward.
- Robustness: 7 - no removal hook; no cleanup helper.
- API design: 7 - returns the added-signal connection only; user has to manage the initial-pass work.
- Noob-friendliness: 7 - fine.
- Style consistency: 8 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ConnectToTag.luau:3` - also accept a `removedCallback` and wire up `GetInstanceRemovedSignal` so the helper is symmetric.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ConnectToTag.luau:4` - warn on nil args instead of silent return.

### DebugUtils
Path: `src/ReplicatedStorage/Shared/Components/Util/DebugUtils.luau`

Scores (1-10):
- Clarity: 6 - mixes unrelated helpers (SafeCall, draw shapes, profile).
- Robustness: 4 - `NewBox` uses `p.Shape = Enum.PartType.Ball`, which is a Ball not a Box; config.lifetime is coerced even when caller passes a valid value (fine) but `thickness` default is only applied to width/height, not depth.
- API design: 5 - module is a grab-bag; draw helpers belong in their own `Draw` util.
- Noob-friendliness: 5 - `tick()` used instead of `os.clock()`.
- Style consistency: 6 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DebugUtils.luau:92` - `NewBox` incorrectly sets `p.Shape = Enum.PartType.Ball`; remove that line so it remains a cuboid block.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DebugUtils.luau:16` - swap `tick()` for `os.clock()`; tick is deprecated.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DebugUtils.luau:7` - move `SafeCall`/`TimeBlock`/`ProfileInstance` into a separate `DiagnosticsUtil.luau` so `DebugUtils` is just the draw helpers.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DebugUtils.luau:3` - guard `debugParent` creation against the server (draw helpers are typically client-side only) or document that it creates a workspace folder on both sides.

### DecalProjector
Path: `src/ReplicatedStorage/Shared/Components/Util/DecalProjector.luau`

Scores (1-10):
- Clarity: 7 - documented sections.
- Robustness: 6 - no cleanup if `set.Lifetime` is 0 and `set.FadeTime` is 0 the decal lingers forever; silently returns when Texture missing.
- API design: 7 - config table is clear.
- Noob-friendliness: 7 - readable.
- Style consistency: 8 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DecalProjector.luau:46` - warn when `set.Texture` is nil instead of silently returning.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DecalProjector.luau:59` - use `Debris:AddItem(part, set.Lifetime + set.FadeTime)` as a safety net in case the task.delay fails.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/DecalProjector.luau:7` - the `table` import via `ExtendedLibraries.table` shadows the global `table`; rename to `tableX` or `tblLib` for clarity.

### ExtendedLibraries (`ExtendedLibraries/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries`

Scores (1-10):
- Clarity: 8 - each file is small with comments.
- Robustness: 6 - `chooseFromDictionary` does an O(n) count then O(n) pick (could be done in one pass); `assertNumber`/`assertTable` throw on bad input instead of returning the library's name from `error("prefix: ...")`.
- API design: 6 - uses `camelCase` while rest of repo leans `PascalCase`; exposes a `lib`-cloned inner table via metatable, which confuses autocomplete for noobs.
- Noob-friendliness: 6 - fine once you know to do `ExtendedLibraries.math.lerp`.
- Style consistency: 5 - breaks PascalCase convention.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries/init.luau:1` - add a one-line comment saying these extend (not replace) Luau's `math`/`table`/`Random`/`Vector3` via metatable.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries/MathExtended.luau:9` - use `error(..., 2)` so the stack points at the caller.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries/RandomExtended.luau:42` - the error-after-return on line 42/50 is unreachable after `error`; delete the trailing `return` statements.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries/RandomExtended.luau:91` - replace the two-pass `chooseFromDictionary` with a single pass using `math.random` on an accumulated count.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ExtendedLibraries/TableExtended.luau:16` - rename `merge` to `mergeInto` or document that it does not deep-merge; new users will assume deep.

### FlashController
Path: `src/ReplicatedStorage/Shared/Components/Util/FlashController.luau`

Scores (1-10):
- Clarity: 4 - top-of-file has a stray `::` type-cast block outside any expression.
- Robustness: 3 - `config.Object` is accessed on line 39 but `config` may be nil; `table.merge({Lighting=true}, config)` is re-used without `set.HighlightConfig` ever existing, crashes on first call with `Object`.
- API design: 5 - `presets` table is empty and the `Preset` argument never resolves.
- Noob-friendliness: 3 - the stray `::` block will surprise any reader.
- Style consistency: 4 - diverges from the rest.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/FlashController.luau:4` - delete the stray `local presets = {}\n::\n{ [string]: {} }` block; either drop presets entirely or give it a real type alias via `type Presets = { [string]: {} }`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/FlashController.luau:39` - guard `set.HighlightConfig.Object = set.Object` with a `set.HighlightConfig = set.HighlightConfig or {}` check to prevent nil-index.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/FlashController.luau:13` - create the `ColorCorrectionEffect` lazily (on first `:Flash` call) instead of at require time, so the module is safe to require on the server.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/FlashController.luau:101` - document that `Object`, `Lighting`, and `Preset` are all optional and at least one must be truthy.

### IllusionIAS (`IllusionIAS/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS`

Scores (1-10):
- Clarity: 6 - large single file with many self:* methods defined via field assignment.
- Robustness: 7 - defensive guards, handles destroy path.
- API design: 7 - rich surface but inconsistent (dot `IIAS.new` vs colon `self:SetHold`).
- Noob-friendliness: 5 - will not work on server (early `return {}::IIAS`); 50+ methods without grouping is overwhelming.
- Style consistency: 6 - uses `@native` attributes that other modules do not.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/init.luau:11` - the early server return `return {}::IIAS` makes `Shared.Utilities.IllusionIAS.new` nil on the server; add a no-op `new` that warns or document clearly.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/init.luau:13` - the `require("@self/...")` syntax is non-standard; change to `require(script:WaitForChild("..."))` to match the rest of the codebase.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/BindUtils.luau:11` - same `@self` / relative-path require issue.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/init.luau:79` - `if not InputContext then return {} :: IAS.Object end` is unreachable since `Instance.new("InputContext")` never returns nil; delete the dead branch.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/init.luau:186` - the large stub-table of placeholder functions on lines 130-183 should be removed once the real methods below redefine them - keep or delete them, don't do both.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/IllusionIAS/IllusionSignal.luau:83` - `Wait` uses `args = table.pack(...)` then `table.unpack(args or {})` - the `or {}` is dead since `args` is always set in the callback.

### Imagine
Path: `src/ReplicatedStorage/Shared/Components/Util/Imagine.luau`

Scores (1-10):
- Clarity: 6 - math for tiling is dense and uncommented.
- Robustness: 5 - `game["Run Service"]` stringly access rather than `RunService`; no guard for server.
- API design: 6 - `apply`/`Animate`/`Unapply` tri-step is fine but `Unapply` iterates `self` and nukes connections, which breaks metatable.
- Noob-friendliness: 5 - config fields are undocumented.
- Style consistency: 6 - method naming mixed.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Imagine.luau:44` - replace `game["Run Service"]` with `game:GetService("RunService")`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Imagine.luau:18` - early-return a warn if called on the server; `RenderStepped` is client-only.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Imagine.luau:6` - document each DefaultConfig field; XTileAmount/YTileAmount are not intuitive.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Imagine.luau:75` - track the connection explicitly in `self.Connection` and disconnect it before the generic iteration, so the intent is clear.

### LODManager
Path: `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau`

Scores (1-10):
- Clarity: 5 - unit of "batch" is under-documented.
- Robustness: 3 - `batch = math.ceil(registeredCount+1/Config.BatchSize)` has operator-precedence bug (`1/BatchSize` first) so every registration goes in batch 1; `Deregister` calls `table.clear(Batches[batchNum][ID])` then sets it to nil - `table.clear` on a data table mutates shared refs; `table.clear(found)` on an entry that was also in `Batches` breaks iteration.
- API design: 5 - `UpdateModels` reparents models to `debrisFolder` without restoring the previous model.
- Noob-friendliness: 4 - no example usage.
- Style consistency: 6 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau:54` - fix the operator precedence: `math.ceil((registeredCount + 1) / Config.BatchSize)`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau:81` - remove the `table.clear(Batches[batchNum][ID])` call; the entry is a shared ref and clearing it also wipes `Registered[ID]` mid-operation.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau:86` - remove the unconditional `table.clear(found)` after dereg; the entry is already nil'd and clearing corrupts any holder references.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau:118` - `:Init` starts an infinite `while true do` without any yield if `BatchUpdateDelay == 0` and `Batches` is empty; add a `task.wait(Config.UpdateSpeed)` outside the inner loops unconditionally.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/LODManager.luau:1` - document the format of the `Models` parameter at the top with an example.

### Observer
Path: `src/ReplicatedStorage/Shared/Components/Util/Observer.luau`

Scores (1-10):
- Clarity: 7 - small but `newproxy`-based.
- Robustness: 6 - no disconnect; if `callback` errors the set still applied; no `__index` forwarding for method calls.
- API design: 5 - returns a proxy that hides the underlying table, but users can still mutate `tabl` directly and skip notification.
- Noob-friendliness: 4 - newproxy + userdata metatables are advanced; no example.
- Style consistency: 6 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Observer.luau:6` - document that the returned proxy must be assigned to (not the original table) for `callback` to fire, and add a usage example in a comment.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Observer.luau:12` - pcall the callback so a throwing observer doesn't prevent later writes.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Observer.luau:6` - add a type export `export type Observed<T> = ...` so downstream code can annotate.

### Pool (`Pool/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/Pool`

Scores (1-10):
- Clarity: 5 - split into 7 files with lots of features, but individual files are well-commented.
- Robustness: 4 - `Utils.getMemoryPressure` and `Validation.checkDataIntegrity` have leftover `print("DEBUG: ...")` calls; `Utils.smartGarbageCollection` calls `gcinfo()` twice hoping for GC (misuse - `gcinfo` is read-only); Heartbeat-driven `autoOptimize` runs every frame and walks all pools (wasteful).
- API design: 5 - surface is huge (60+ methods) for what should be a simple pool, with confusing parallel names (`OptimizeAllPools` vs `OptimizeAllPoolsWithGC`).
- Noob-friendliness: 4 - a beginner asked to pool a bullet will be overwhelmed.
- Style consistency: 7 - internally consistent; differs from rest of framework (underscore-prefixed fields).

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/Utils.luau:137` - delete the leftover `print(("DEBUG: Current memory usage: %.1f MB")...)` call.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/Validation.luau:145` - delete the leftover `print(("DEBUG: Pool %s - ..."))` call.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/Utils.luau:168` - `smartGarbageCollection` calls `gcinfo()` twice; Luau has no programmatic GC trigger, so either `collectgarbage("collect")` or remove the function.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/init.luau:218` - `RunService.Heartbeat:Connect(autoOptimize)` fires every frame; gate it behind a `Constants.AUTO_OPTIMIZE_ENABLED` flag so users can opt out.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/Utils.luau:17` - the server-side `UIPoolStorage = StarterGui:FindFirstChild(...)` pattern creates a `ScreenGui` inside `StarterGui` at require time; make that lazy (only on first UI pool).
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Pool/init.luau:398` - `pool._maxSize` is accessed as a direct field outside the impl type; either expose via `pool:GetMaxSize()` or move `EmergencyCleanup` to use the public API.

### Promise (`Promise/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/Promise`

Scores (1-10):
- Clarity: 6 - `init.luau` is all types, `Main.luau` is 1401 lines of vendored roblox-lua-promise.
- Robustness: 9 - well-tested upstream library.
- API design: 9 - follows standard Promise pattern.
- Noob-friendliness: 6 - type aliases `Promise1`...`Promise8` are hard to understand.
- Style consistency: 6 - vendored code - differs from the project style.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Promise/init.luau:1` - add a top-of-file comment: "Vendored from fewkz/typed-luau-promise + evaera/roblox-lua-promise; do not modify".
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Promise/init.luau:248` - the `require(script:WaitForChild("Main"))` is fine; but casting via `:: PromiseLib` hides the typed chain - document that chained return types are fake-typed via `Promise1...8`.

### Ragdoll
Path: `src/ReplicatedStorage/Shared/Components/Util/Ragdoll.luau`

Scores (1-10):
- Clarity: 7 - small and linear.
- Robustness: 5 - Motor6D.Enabled = false then created BallSocket; does not clean up its attachments on `Stop` (leaks two attachments per joint); `:SetStateEnabled(Ragdoll, false)` last line looks backward.
- API design: 6 - just Start/Stop, fine.
- Noob-friendliness: 6 - uses `pairs` on descendants which is slow for big characters.
- Style consistency: 6 - mixes PascalCase and lowercase.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Ragdoll.luau:56` - on `Stop`, also destroy the created `Attachment` instances (currently they leak).
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Ragdoll.luau:73` - `SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)` at end of `Stop` seems wrong - you just forced state to `GettingUp`; remove the last line or document why.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Ragdoll.luau:11` - drop `pairs(...)` - iterate `character:GetDescendants()` directly.

### SFXHandler
Path: `src/ReplicatedStorage/Shared/Components/Util/SFXHandler.luau`

Scores (1-10):
- Clarity: 6 - short but mixes lookup and playback.
- Robustness: 4 - `sfx:GetDescendants()` linear search on every call; `local new = pitchShift(...)` assigned but unused; `PlayByCategory` can crash when the first found `Folder` is empty (`math.random(1,0)`).
- API design: 5 - mandatory ReplicatedStorage layout (`Assets/SFX/Categories`) is undocumented.
- Noob-friendliness: 4 - failure mode is a silent return.
- Style consistency: 6 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SFXHandler.luau:73` - guard `#c:GetChildren() == 0` before calling `math.random(1, n)`; current code errors on empty category.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SFXHandler.luau:56` - remove the unused `local new =` assignment.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SFXHandler.luau:31` - build a name→Sound cache in module init rather than walking `GetDescendants` on every `:Play`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SFXHandler.luau:4` - document the required `ReplicatedStorage/Assets/SFX/Categories/<CategoryName>/<Sound>` folder layout.

### Signal (vendored SignalPlus at `Util/Signal.luau`)
Path: `src/ReplicatedStorage/Shared/Components/Util/Signal.luau`

Scores (1-10):
- Clarity: 7 - well-commented.
- Robustness: 9 - linked-list signal, reused threads.
- API design: 8 - standard.
- Noob-friendliness: 6 - ok.
- Style consistency: 7 - vendored, differs.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Signal.luau:1` - mark with a header comment `-- Vendored: SignalPlus v3.6.1 (Alexander Lindholt). Do not modify.`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Signal.luau:1` - this file is byte-identical to `Pool/SignalPlus.luau`; pick one location and `require` it from the other to avoid drift.

### SpectateService (`SpectateService/*`)
Path: `src/ReplicatedStorage/Shared/Components/Util/SpectateService`

Scores (1-10):
- Clarity: 6 - two files; CameraManager has a big ASCII banner and useful docstring.
- Robustness: 5 - returns `true :: main` on server (same bug shape as `PlayerState`); `CameraManager` connects a `RenderStepped` handler *at require time*, meaning any require of `SpectateService` starts a render loop.
- API design: 6 - `:Watch`/`:StopWatching`/`:GetWatched` is fine.
- Noob-friendliness: 5 - server-side users will get crashes.
- Style consistency: 6 - mixed.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SpectateService/init.luau:5` - replace the server-side `return true :: main` with an empty stub table that warns on use.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SpectateService/CameraManager.luau:226` - the top-level `RenderStepped:Connect` runs whenever `CameraManager` is required; move it into `Manager.new`'s first call or guard it so requiring on the server doesn't error.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SpectateService/CameraManager.luau:102` - `plr:GetMouse()` is deprecated-ish; use `UserInputService:GetMouseLocation` / `Camera:WorldToViewportPoint` instead.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/SpectateService/init.luau:73` - the sequence `curCam:Destroy(); curCam = nil; camManager.close()` is on one line with no comma - hard to read; split across lines.

### Spring
Path: `src/ReplicatedStorage/Shared/Components/Util/Spring.luau`

Scores (1-10):
- Clarity: 7 - `--!strict` with detailed moonwave docstrings.
- Robustness: 8 - lazily computes position/velocity.
- API design: 8 - good; aliases (`p`, `v`, `t`) are nice.
- Noob-friendliness: 7 - documented; physics naming may intimidate.
- Style consistency: 7 - vendored - differs from repo.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Spring.luau:8` - add a vendored-from header (`probablytukars/LuaQuaternion`) with a "do not restyle" note.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Spring.luau:117` - `Spring.new` signature has `damping: number?, speed: number?, clock: (() -> number)?` but the `t_Spring` type lists them non-optional; tighten types to match.

### StateMachine
Path: `src/ReplicatedStorage/Shared/Components/Util/StateMachine.luau`

Scores (1-10):
- Clarity: 6 - methods compact and commented.
- Robustness: 5 - `states` is a module-level global; multiple StateMachine instances share the same state dictionary; `:Init()` discovers from `script.States` which does not exist in the repo by default.
- API design: 6 - `TryEnter`/`Exit`/`Update` fine; no explicit transition table.
- Noob-friendliness: 5 - requires a `States` sibling folder that is not present.
- Style consistency: 6 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/StateMachine.luau:15` - move `states = {}` into each instance (e.g., `self._states = {}`) or document loudly that all StateMachine instances share one registry.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/StateMachine.luau:127` - `:Init()` assumes a `States` child; create it (or warn) if missing so first-time users don't get a crash.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/StateMachine.luau:49` - `:TryEnter` exits self with nil switchTo then re-enters - the recursion path via `Exit(nil, ...)` -> no-op return is subtle; simplify.

### TimeFormatter
Path: `src/ReplicatedStorage/Shared/Components/Util/TimeFormatter.luau`

Scores (1-10):
- Clarity: 9 - four named format helpers.
- Robustness: 7 - `FormatMS` handles `seconds <= 0` fine; non-integer seconds pass through to `%02d` which truncates silently.
- API design: 8 - clear.
- Noob-friendliness: 9 - readable.
- Style consistency: 8 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/TimeFormatter.luau:3` - `pad(n)` should `math.floor(n)` first so fractional seconds don't produce garbage like `01:05.75`.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/TimeFormatter.luau:67` - `SmartFormat` with 30 seconds returns `"00:30"` via its final fallback, but the branch uses `tostring(seconds)` which would print `30` (not zero-padded) despite the `"00:"` prefix; normalize by always using `pad(seconds)`.

### Trove
Path: `src/ReplicatedStorage/Shared/Components/Util/Trove.luau`

Scores (1-10):
- Clarity: 9 - moonwave-documented, named markers.
- Robustness: 9 - proven code.
- API design: 9 - full surface.
- Noob-friendliness: 8 - rich examples.
- Style consistency: 6 - vendored - differs from repo.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/Trove.luau:1` - explicit vendored-from header (`Sleitnick/Trove`) for consistency with other vendored utilities.

### ViewCheck
Path: `src/ReplicatedStorage/Shared/Components/Util/ViewCheck.luau`

Scores (1-10):
- Clarity: 8 - tight.
- Robustness: 6 - module-level `params` is shared - concurrent callers clobber `FilterDescendantsInstances`.
- API design: 7 - three functions, good.
- Noob-friendliness: 7 - readable.
- Style consistency: 8 - fine.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ViewCheck.luau:6` - create a fresh `RaycastParams` inside `isBlocked` each call, or clone before mutating, to avoid cross-thread clobbering.
- [ ] `src/ReplicatedStorage/Shared/Components/Util/ViewCheck.luau:4` - guard `cam = workspace.CurrentCamera` - it can be nil on the server; warn and early-out there.

### Services index (`Components/Services/init.luau`)
Path: `src/ReplicatedStorage/Shared/Components/Services/init.luau`

Scores (1-10):
- Clarity: 6 - mixes Roblox services with framework services.
- Robustness: 7 - fine.
- API design: 4 - exposing `ReplicatedStorage`, `Players`, etc. here shadows the convention; noobs won't know which `Services` to use.
- Noob-friendliness: 5 - the table blurs "game services" and "your services".
- Style consistency: 7 - matches.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Services/init.luau:3` - split into two tables: `RobloxServices = { ... }` and `FrameworkServices = { DataService, VFX }`, or move the Roblox service handles into `Shared.Utilities.Services`.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/init.luau:21` - `StarterPlayerScripts = StarterPlayer:WaitForChild("StarterPlayerScripts")` will yield forever if require happens before the hierarchy builds; use `:FindFirstChild` with a warning.

### VFX (`Services/VFX/*`)
Path: `src/ReplicatedStorage/Shared/Components/Services/VFX`

Scores (1-10):
- Clarity: 6 - `VFX/init.luau` is dense; `Emitter.luau` is clear.
- Robustness: 4 - `getVFX` silently returns when pool is missing; `LoopEmit` defines `destroyed` but never uses it; assumes `ReplicatedStorage.Assets.VFX` exists (WaitForChild yields forever).
- API design: 6 - `newVFX(data)` config is documented inline but noobs need to read the source.
- Noob-friendliness: 5 - the `emitAnyways` coercion pattern is hard to reason about.
- Style consistency: 6 - matches framework style.

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Services/VFX/init.luau:17` - fail gracefully if `ReplicatedStorage.Assets.VFX` is missing (warn + return the partial module) rather than hanging on `WaitForChild`.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/VFX/init.luau:41` - warn when `getVFX(particleName)` cannot find a pool, instead of silently returning.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/VFX/init.luau:123` - simplify the triple-coerced `emitAnyways` - compute a plain boolean `local shouldEmit = not custom or customResult ~= false`.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/VFX/Emitter.luau:36` - delete unused `local destroyed = false` in `LoopEmit`.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/VFX/Emitter.luau:44` - `toEmit = {obj, table.unpack(obj:GetDescendants())}` collapses to just `obj:GetDescendants()` plus `obj`; prefer `for _, desc in { obj, table.unpack(obj:GetDescendants()) } do` on one line or a clearer loop.

### DataService (`Services/DataService/*`)
Path: `src/ReplicatedStorage/Shared/Components/Services/DataService`

Scores (1-10):
- Clarity: 8 - moonwave-documented, clean split server/client/Data/Networker.
- Robustness: 6 - `Signal.luau` vendored (skipped per rules), but server relies on `ProfileStore`; early-returns on opposite run context return `{}::DataServiceServer`, so typed callers get `nil` errors.
- API design: 8 - typed, documented.
- Noob-friendliness: 6 - `path` as `string | {string}` is great; but required `:init(...)` before use is only mentioned in docstrings.
- Style consistency: 7 - internally consistent; differs from rest (snake-less, method-style with dot syntax).

Fixes:
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/DataServiceServer.luau:6` - the `return {} :: DataServiceServer` stub on the client silently hands back an empty table; add a metatable `__index` that warns on field access to surface misuse fast.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/DataServiceClient.luau:6` - same issue on the server side; warn on misuse.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/Networker/NetworkerServer.luau:75` - the `assert(instance:GetAttribute(INSTANCE_ATTRIBUTE) == nil, ...)` blocks re-registering the same instance; consider reusing the existing tag instead of erroring.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/Networker/NetworkerServer.luau:140` - `metatable.__index` access without nil-check will error if the metatable is an empty table; guard with `type(metatable.__index) == "table"`.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/Data.luau:164` - `_firePathChangedSignals` walks the path but uses `prev` from the last table - only the final leaf gets a meaningful `prev`; document or fix so intermediate `changed` signals get the correct prev.
- [ ] `src/ReplicatedStorage/Shared/Components/Services/DataService/init.luau:1` - add a top-of-file comment stating ProfileStore is vendored and DataService wraps it, so users know where the data persists.

## Cross-cutting Issues
- **Stray `::` type-literal blocks.** `Components/Info/GameState.luau` and `Components/Util/FlashController.luau` both contain orphan `::` + `{ ... }` blocks outside any declaration. These are remnants of a type-annotation experiment and likely fail `--!strict` parsing. Search the rest of the repo for the pattern and remove.
- **Duplicated vendored SignalPlus.** `Components/Util/Signal.luau`, `Components/Util/Pool/SignalPlus.luau`, and `Components/Services/DataService/Signal/init.luau` are three signal implementations with overlapping surfaces. Pick one canonical location (probably `Components/Util/Signal.luau`) and have the others `require` it - reduces drift and binary size.
- **Silent early-return anti-pattern.** Many modules (`FastTween`, `ConnectToAdded`, `ConnectToTag`, `VFX.getVFX`, `SFXHandler:Play`, `DecalProjector:Project`, `Imagine.apply`) return silently on bad args. New users will not know their call failed. Replace with `warn` + return sentinel.
- **Indefinite `WaitForChild` on missing expected assets.** `Shared/init.luau:30` (`Assets`), `Shared/init.luau:31` (`Data`), `Services/init.luau:21` (`StarterPlayerScripts`), `Network/SignalWrapper:11` (`Remotes`), `VFX/init.luau:17` (`Assets.VFX`). If the expected child doesn't exist, require yields forever and the framework never finishes loading. Use `FindFirstChild` + create-or-warn.
- **Context-wrong early returns.** `PlayerState.luau`, `SpectateService/init.luau`, `IllusionIAS/init.luau`, and both halves of `DataService` early-return `true::type` or `{}::type` on the wrong run-context. Downstream code that trusts the type annotation indexes a boolean or empty table and crashes. Replace with stub tables whose fields warn.
- **Module-require side effects.** `FlashController` creates a `ColorCorrectionEffect` in Lighting at require time. `Pool/Utils.luau` creates `workspace.PoolStorage` and `StarterGui.UIPoolStorage`. `CameraManager` connects `RenderStepped` at require time. `LODManager.luau:118` spawns an infinite loop when `:Init` is called. Pull these into explicit `init`/first-call paths so requiring is side-effect-free.
- **Naming inconsistency.** Public methods use a mix of `PascalCase` (`ActionBuffer:Record`, `StateMachine:TryEnter`), `camelCase` (`DataService.set`, `Networker.fire`), and `lowercase` (`Signal.new`, `ragdoll.Start`). Pick one (the framework CLAUDE.md leans casual/PascalCase) and document it for contributors.
- **Left-in debug prints.** `Pool/Utils.luau:137`, `Pool/Validation.luau:145`, and `Pool/Debug.luau:290` print `DEBUG:` lines every frame or every validation. Strip before open-sourcing.
- **Inconsistent type annotations.** Some files are `--!strict`, some have partial types, many have none. For an open-source release, either mark Shared as "type-partial by design" in the README or do a pass to add at minimum parameter/return types to public functions.
- **Utility grab-bag.** `Components/Util/` has 20+ entries with no sub-grouping (signal/async, math, instance-lifecycle, fx/camera, input). Consider sub-folders (`Util/Async/`, `Util/Fx/`, `Util/Math/`) or at least comment-grouping in `Util/init.luau` so newcomers can navigate.
- **`tick()` vs `os.clock()`.** `DebugUtils`, `Pool/*`, and some analytics code use `tick()`, which is deprecated on Roblox. Sweep to `os.clock()`.
