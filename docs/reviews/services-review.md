# Services Review

## Summary
The DialogueService is a clever, feature-rich subsystem (typewriter with inline effects, word-wrap layout, tag parsing, conditional conversation selection, command streaming, reply trees) but it is not yet open-source ready. The public API leaks internal mutable config (a module-level `Config` table shared across all callers), relies on unenforced side-ordering between `:SetConfig` and `:Type`, has silent-fail branches (no `TypeFrame`, no convo, missing condition), vendors SignalPlus twice verbatim without attribution in the outer `Util/Signal.luau`, contains a stray debug `Test` action (`print("HAHAHAHAHAHA")`), and ships near-empty `ConversationInfo`/`Info` stubs. Docs for the tag/command grammar live only as a block comment in `Conversations/Info.luau`. Noob-friendliness suffers from terse variable names (`serv`, `convos`, `cond`, `suc`, `tw`), lack of type annotations on the public return shapes, and scattered `game.Players.LocalPlayer`/`game.ReplicatedStorage` usage instead of `GetService`.

## Modules

### DialogueGrabber
Path: `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 5 - Short file but names like `serv`, `convos`, `cond`, `suc` and the nested condition loop take work to parse.
- Robustness: 4 - Silently returns nothing when info missing, requires a sibling `Conditions` folder that is not documented or present, ignores nil-npc case.
- API design: 5 - Single `:Grab(prompt)` is fine, but return type `({}, Model)` is wrong when it bails (returns nothing), and coupling to `ProximityPrompt` attributes is undocumented.
- Noob-friendliness: 4 - New users will not know what `ID` attribute, `Conditions` folder, or condition function signature are expected.
- Style consistency: 5 - Mix of `ipairs`-less `for ... in conditions do` and `pairs` elsewhere, inconsistent with framework conventions.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:3` - replace `game.ReplicatedStorage` with `game:GetService("ReplicatedStorage")` or remove the unused `rs` local entirely.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:4` - rename `serv` to `serviceRoot` for clarity.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:6` - rename `convos` to `conversationsFolder`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:11` - replace `game.Players.LocalPlayer` with `game:GetService("Players").LocalPlayer` and guard for server context.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:13` - document/ship a `Conditions` folder or gracefully skip when it does not exist (`FindFirstChild("Conditions")`).
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:17` - fix annotated return type; declare as `(({}?, Model?))` or `(table?, Model?)` since the function can return nothing on line 22.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:22` - warn when `info` is nil so developers know their prompt `ID` did not match any entry in `ConversationInfo`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:29` - rename `suc` to `allConditionsPassed` and `cond` to `conditionName`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueGrabber.luau:44` - warn when `chosen` names a file that does not exist under `Conversations/`.

### DialogueHandler
Path: `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/` (`init.luau`, `Replace.luau`)

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 4 - 200-line `StartConversation` body does parsing, UI cloning, command dispatch, reply loop, and teardown in one closure; easy to get lost.
- Robustness: 3 - `activeTW` never nilled after Completed, `cmdConn` leaked if node has no commands branch throws, `repeat task.wait() until selected` busy-waits, `Controller:Destroy` walks the Controller while nilling keys (undefined ordering), no pcall around user `CommandRun` handlers.
- API design: 5 - `DialogueMod.new` returns an object but `:StartConversation` returns a new Controller that shadows `self.Controller`; confusing duality. Config uses string-literal union but runtime picks `"Letter"` as fallback without validation.
- Noob-friendliness: 4 - No usage example, required UI child names (`DialogueTextLabel`, `RepliesContainer`) only discoverable by reading source.
- Style consistency: 5 - Mixes `ipairs`, `pairs`, and generic-for; some lines have trailing whitespace per the file; tabs otherwise consistent.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:11` - document required UI children (`DialogueTextLabel`, `RepliesContainer`) in a block comment above `.new`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:17` - add missing `OpenUI`/`CloseUI` signal fields to the typed Config / self shape so users can discover them.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:36` - split `StartConversation` into helper functions (`typeNode`, `runCommands`, `collectReplies`) to cut the 150-line closure.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:41` - use `break` inline on its own line and wrap with `do ... end` or add semicolons; the `for ... do if ... then current = node break end end` one-liner is hostile.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:79` - replace the `for i, v in Controller do Controller[i] = nil end` pattern with explicit field nilling; mutating a table while iterating is undefined.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:95` - replace `game.Players.LocalPlayer` with a cached service reference, and pass the player in instead of hard-coding local-only use.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:118` - guard `currentTW.OnLetterTyped:Connect` with a nil check; `Typewriter:Type` can return nil when `TypeFrame` is unset.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:173` - replace `repeat task.wait() until selected` with a signal/event wait to avoid per-frame polling.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:185` - same: replace `repeat task.wait() until continue` with a bindable or signal wait.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/init.luau:178` - use `Debris:AddItem(child, 0.3)` instead of `task.delay` + `Destroy` closure.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/Replace.luau:3` - expose `replaceList` as a parameter so users can add their own placeholders (e.g. `<userId>`, `<coins>`) without editing framework source.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/Replace.luau:17` - document the command/tag grammar (`$cmd:arg$`, `<tag>`) in the file header; currently it only lives in `Conversations/Info.luau`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/DialogueHandler/Replace.luau:21` - cache the pattern `"%$(.-)%$"` in a local; also handle escaping so a literal `$` is possible.

### Util (Animator, Signal)
Path: `src/ReplicatedStorage/Services/DialogueService/Util/` (`Animator.luau`, `Signal.luau`)

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 5 - Animator is short but uses module-level `animators`/`anims` tables that are shared across all callers, which is not obvious.
- Robustness: 3 - `anims` is a flat name map shared across every character; two characters with same-named animations collide. `ClearInfo` does not clear `anims`. `LoadAnimation` called every `PlayAnim` when `animName` is nil and `anim` passed.
- API design: 4 - `PlayAnim` overloads on (name xor anim) with unclear precedence; `LoadAnims` returns the module-level `anims` table, exposing shared mutable state.
- Noob-friendliness: 4 - No usage docs, and `SetUp` must be called before any other method but that is not stated.
- Style consistency: 6 - Consistent with framework informal style; vendored `Signal.luau` is upstream SignalPlus kept verbatim (fine but unattributed at this path vs `Util/Typewriter/Util/Signal.luau` duplicate).

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:3` - move `animators` and `anims` into a per-character table keyed by the character model, so multiple NPCs do not stomp each other.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:4` - scope `anims` per character to fix the name-collision bug when two rigs share animation names.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:12` - in `ClearInfo`, also clear the character's loaded animation tracks to free memory.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:20` - replace `i, anim in ipairs(animFolder:GetChildren())` with `_, anim` since the index is unused.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:36` - document that passing `anim` without `animName` causes a fresh `LoadAnimation` every call (memory leak) and gate with a cache.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Animator.luau:42` - return both the track and a stop handle, or rename to `:Play` and have it idempotent; today callers must juggle `track:Stop()`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Signal.luau:1` - deduplicate with `Util/Typewriter/Util/Signal.luau`: both are byte-identical vendored SignalPlus; keep one copy and require it from both sites.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Signal.luau:23` - add a one-line note above the ASCII banner pointing at the framework's preferred Signal (`Shared.Util.Signal` or similar) so users know when to use which.

### Typewriter
Path: `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/` (`init.luau`, `Util/Layout.luau`, `Util/Parse.luau`, `Util/Signal.luau`)

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 5 - Good section headers (`---<>--- Helpers ---<>---`), but the mutable module-level `Config` plus `DefaultConfig` plus two different `Config` type declarations on `:Type` and `:SetConfig` is confusing.
- Robustness: 4 - `:Type` bails with `warn("No type frame.")` and returns nil, breaking any caller chaining. `_activeConnections` never cleared on normal completion, so effect RenderStepped connections leak after the message finishes. `:Destroy` walks Controller while nilling. `:TypeInLabel` uses `contentText:sub(i, i)` which is wrong for multi-byte graphemes even though it iterates `utf8.graphemes`.
- API design: 4 - Singleton module with mutable `Config` means two callers cannot type simultaneously with different settings; `:SetConfig` affects everything. Effects are auto-loaded from a folder but there is no list/get API.
- Noob-friendliness: 5 - Effect tags (`<wave>`, `<rainbow>`, `<b>`) discoverable only by reading `Effects/` folder; no documented list.
- Style consistency: 6 - Matches framework casual style, consistent indent.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:19` - do not store `Config` at module scope; pass it into each `:Type` call or store per-controller so concurrent typewriters can differ.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:49` - `Typewriter.DefaultConfig = {} :: Config` lies to consumers (actually empty); expose the real `DefaultConfig` table declared on line 5.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:73` - inline the giant duplicated config shape on `:SetConfig`; reuse the named `Config` type instead of copy-pasting the fields.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:105` - instead of silent `warn("No type frame.") return`, error with guidance: `error("Typewriter: TypeFrame not set. Call :SetConfig({TypeFrame = frame})")`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:135` - replace the `for i, v in Controller do Controller[i] = nil end` pattern with explicit nilling of known fields.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:149` - `:Wait` mixes present-tense + future-state race: a second call before the first delay finishes resets the timer silently; document or queue waits.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:191` - disconnect effect connections on `Completed` too, not only `Destroy`, otherwise RenderStepped effects keep ticking after the message finishes.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:257` - `contentText:sub(i, i)` breaks on multi-byte glyphs; use `utf8.offset` + `utf8.codepoint` or `string.sub(contentText, utf8.offset(contentText, i), utf8.offset(contentText, i+1)-1)`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/init.luau:283` - `Typewriter:Init` is never called by any in-area loader; rely on lazy-loading on first `:Type` or document that the framework's ModuleLoader calls it.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Layout.luau:9` - `BOLD_MAP` only supports 6 fonts; add a fallback that tries `Font.fromEnum(baseFont)` with `Bold` weight for the modern `Font` API, or document the limitation.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Layout.luau:25` - `baseFont: Font` is annotated `Font` but receives `Enum.Font` at call site (`Config.LetterTemplate.Font`); reconcile the type.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Layout.luau:59` - cache `Vector2.new(10000, 10000)` as a module-level constant; it is reallocated on every measure.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Parse.luau:18` - document that tag names must match `%w+`; underscores / numbers-only will silently fail.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Parse.luau:39` - unbalanced close tags (`</wave>` without opening) silently do nothing; warn instead.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Util/Signal.luau:1` - deduplicate with outer `Util/Signal.luau` (identical vendored file).

### Typewriter Effects
Path: `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/` (`b.luau`, `i.luau`, `wobble.luau`, `ghost.luau`, `glitch.luau`, `pulse.luau`, `rainbow.luau`, `wave.luau`)

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 7 - Each effect is tiny and its math is discoverable; good pattern.
- Robustness: 4 - `b.luau` / `i.luau` return no connection so Typewriter skips tracking them (fine today but inconsistent with the "return RBXScriptConnection" contract). `glitch.luau` clones labels and never destroys them on controller `:Destroy`; `pulse.luau` creates a `UIScale` never cleaned up; `wobble.luau` requires `Shared` but never uses it.
- API design: 6 - Good convention: `(label, data) -> RBXScriptConnection?`. But cleanup of side-effect Instances (clones, UIScales) not part of the contract.
- Noob-friendliness: 6 - Small and copy-pasteable; new effects easy to add. Lack of a "how to add an effect" doc file.
- Style consistency: 6 - Consistent tab style; `glitch.luau` has dead commented `--label.ZIndex+=1` and reuses `was1`/`last1` naming that is hard to follow.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/wobble.luau:2` - remove the unused `require(ReplicatedStorage:WaitForChild("Shared"))` import.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/wobble.luau:8` - `originalPos` is captured but the offset is added to `originalPos`, which is fine, but the `div = 500` constant is magic; rename to `WOBBLE_DIVISOR` and comment.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/glitch.luau:7` - return a cleanup object (table with `Disconnect`) that disconnects and destroys `new1`/`new2`, instead of a bare RBXScriptConnection that leaks the clones.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/glitch.luau:14` - remove the dead `--label.ZIndex+=1` comment or enable it with rationale.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/glitch.luau:16` - rename `last1`, `was1`, `last2`, `was2` to `ghostAShownAt`, `ghostAFlipped`, etc.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/pulse.luau:5` - store the `UIScale` on the returned cleanup table and destroy it in `:Disconnect`; current code leaks it when Typewriter destroys.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/b.luau:1` - add a comment that `b` is short for "bold" so new users know why the tag is `<b>`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/i.luau:1` - same; add comment documenting the `<i>` italic tag.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/rainbow.luau:3` - `length = 10` is a module-level magic; rename to `CYCLE_SECONDS` and comment.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/ghost.luau:4` - document that `0.4` per-index offset desyncs letters; extract as named constant.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Util/Typewriter/Effects/wave.luau:5` - extract amplitude (`4`) and frequency (`5`) as named locals so noobs can tweak.

### Actions/Test
Path: `src/ReplicatedStorage/Services/DialogueService/Actions/Test.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 4 - Pure debug stub with `print("HAHAHAHAHAHA")`; purpose unclear.
- Robustness: 5 - Harmless but gets loaded into the public artifact.
- API design: 3 - No contract for what an Action is or how commands map to actions.
- Noob-friendliness: 2 - New users copy this file as a template; "HAHAHAHAHAHA" is not a template.
- Style consistency: 6 - Consistent but trivial.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Actions/Test.luau:1` - replace `print("HAHAHAHAHAHA")` with a meaningful example action (e.g. play an emote) and rename file to `Example.luau`.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Actions/Test.luau:1` - add a header comment describing how an Action is discovered and invoked (currently no loader in this area references `Actions/`).

### Conversations/Greeting
Path: `src/ReplicatedStorage/Services/DialogueService/Conversations/Greeting.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 7 - Reads well as a data file; fields are self-describing.
- Robustness: 5 - `Position` fields are present but unused by the current handler (dead data). `Connections` referencing labels at runtime only; a typo silently ends the convo.
- API design: 6 - Good declarative shape; `Start = true` convention is fine but not documented.
- Noob-friendliness: 6 - Example is decent; the mixed tag/command syntax in strings is visible here for the first time but unexplained.
- Style consistency: 7 - Consistent.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Conversations/Greeting.luau:1` - add a header comment describing the node schema (`Label`, `Text`, `Connections`, `Type`, `Start`).
- [ ] `src/ReplicatedStorage/Services/DialogueService/Conversations/Greeting.luau:8` - either document what `Position` is for (editor-only metadata?) or remove it from the data file.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Conversations/Greeting.luau:31` - add validation in `DialogueHandler` that warns when `Connections` reference a `Label` not present in `lookup`.

### Conversations/Info
Path: `src/ReplicatedStorage/Services/DialogueService/Conversations/Info.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - Pure comment file documenting tag/command grammar.
- Robustness: 10 - Nothing to break.
- API design: 3 - Naming it `Info.luau` inside `Conversations/` is confusing (looks like a conversation named `Info`); it will be loaded by `DialogueGrabber` if referenced and return nil.
- Noob-friendliness: 4 - Documentation hidden inside a `return`less ModuleScript; not discoverable.
- Style consistency: 6 - Consistent block-comment style.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/Conversations/Info.luau:1` - move this grammar doc to a top-level `README.md` or into `DialogueHandler/Replace.luau` header; remove the file from `Conversations/` so grabber can't accidentally require it.
- [ ] `src/ReplicatedStorage/Services/DialogueService/Conversations/Info.luau:1` - if kept, rename to `SYNTAX.md` and place outside `Conversations/` so it is not mistaken for a conversation asset.

### ConversationInfo
Path: `src/ReplicatedStorage/Services/DialogueService/ConversationInfo.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - Small typed table; the `t` type alias is terse.
- Robustness: 5 - Only one rig (`DefaultRig`) is registered; the `Conditioned` table shape is documented by type but never exercised.
- API design: 6 - Key-by-prompt-attribute-ID is reasonable; lack of a registration helper means users edit this file directly.
- Noob-friendliness: 4 - No comment explaining that keys correspond to the `ID` attribute on `ProximityPrompt`s.
- Style consistency: 6 - Consistent.

Fixes:
- [ ] `src/ReplicatedStorage/Services/DialogueService/ConversationInfo.luau:1` - rename the type alias `t` to `ConversationInfoMap` for readability.
- [ ] `src/ReplicatedStorage/Services/DialogueService/ConversationInfo.luau:1` - add a header comment: "Keys correspond to the `ID` attribute set on ProximityPrompts. Values describe which `Conversations/` file to run."
- [ ] `src/ReplicatedStorage/Services/DialogueService/ConversationInfo.luau:10` - provide a commented-out example with `Conditioned` populated so users see how multi-variant selection works.

## Cross-cutting Issues
- SignalPlus is vendored byte-for-byte in two places (`Util/Signal.luau` and `Util/Typewriter/Util/Signal.luau`); pick one and require it from the other, or require the framework's canonical signal.
- `game.Players.LocalPlayer` and `game.ReplicatedStorage` are used directly throughout; switch to `game:GetService(...)` and guard LocalPlayer usage against server contexts.
- Pervasive pattern `for i, v in t do t[i] = nil end` appears in Controller `:Destroy` methods (DialogueHandler, Typewriter, Layout); mutating a table while iterating is undefined behavior in Luau generalized iteration - replace with explicit nilling or `table.clear`.
- No `:Init()`/`:Start()` exists on the top-level DialogueService entry (there is no `init.luau` at `DialogueService/`); the ModuleLoader will require `DialogueGrabber`, `DialogueHandler`, `ConversationInfo`, `Actions/Test`, `Conversations/*` as siblings and print / execute side effects (`Test.luau` is a bare function that returns a callable, but Greeting returns data, etc.). The loader contract for this subtree is undefined; add a `DialogueService/init.luau` that exposes a curated API and sets `Priority`.
- Tag/command grammar (`$cmd:arg$`, `<effect>`, `<plrName>`, `<npcName>`) is documented only as a block comment in `Conversations/Info.luau`. For open-source, promote to a `docs/modules/DialogueService.md` or a header in `Replace.luau` / `Typewriter/init.luau`.
- Effects that spawn side-effect Instances (`glitch` clones labels, `pulse` creates `UIScale`) only return an `RBXScriptConnection`; when Typewriter disconnects on `:Destroy`, the extra Instances persist. Standardize the effect return contract to a cleanup object (`{Disconnect = function() end}`) or pass Typewriter a trove.
- No unit tests or example usage script is shipped alongside DialogueService in this path; new users cannot run the system without reading `StarterGui/Dialogue/` which lives outside this area.
- Variable naming is inconsistent across the area: `serv`, `convos`, `cond`, `tw`, `suc` (DialogueGrabber/Handler) vs. descriptive `characterData`, `cursorX`, `lineHeight` (Layout). Normalize for the open-source audience.
- Silent failures everywhere: missing TypeFrame warns and returns, missing convo returns nothing, missing condition is ignored, missing label in `Connections` ends the dialogue. Add consistent warnings tagged with `[DialogueService]` prefix.
- `Actions/` folder contains `Test.luau` but no discovery mechanism in this subtree wires it to the `CommandRun` signal; either delete the folder or add the wiring and document it.
