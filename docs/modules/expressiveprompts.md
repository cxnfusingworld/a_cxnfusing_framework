# ExpressivePrompts

ExpressivePrompts replaces Roblox's default `ProximityPrompt` UI with a bouncy, animated, themeable card that works across keyboard, gamepad, and touch. Call `:Init()` once on the client and every non-`Default`-style ProximityPrompt in the game is picked up automatically, gets a BillboardGui with a platform-appropriate input glyph, springy appear/hold animations, a progress bar for hold prompts, and configurable colors/sizes via a table of reactive Seam values.

## Overview

What it builds, per prompt:

- A `BillboardGui` adorned to the prompt's parent, sliding in via a Spring on `StudsOffset`.
- A `CanvasGroup` "card" with animated size (squish on hold), subtle rotation, aspect-ratio morph, optional shimmer sweep, and a progress bar for hold prompts.
- A round inner frame hosting the input glyph.
- Two `TextLabel`s: `ActionText` (main line) and `ObjectText` (sub line), which fade out while the prompt is being held.
- Sound effects for appear, click, hold (pitch ramps up while holding), and trigger.
- A tap-target `TextButton` covering the gui when on touch or when `Prompt.ClickablePrompt == true`, calling `Prompt:InputHoldBegin()`/`InputHoldEnd()`.

Supported input kinds, dispatched by `Enum.ProximityPromptInputType`:

- **Keyboard** (also the default fallback for unknown types). Renders a key-cap image plus either a glyph or a `TextLabel` showing the printable character. Uses `UserInputService:GetStringForKeyCode` so the label respects the user's keyboard layout. Labels of 3+ characters (e.g. `Ctrl`, `F12`) drop to `TextSize = 12`.
- **Gamepad**. Looks up an Xbox-style glyph image for `prompt.GamepadKeyCode`. If the keycode isn't mapped, returns no glyph (the prompt still shows, just without an icon).
- **Touch**. Shows Roblox's built-in tap icon (`rbxasset://textures/ui/Controls/TouchTapIcon.png`). The inner frame's "pressed" scale is larger (1.6x vs 1.33x for non-touch) since fingers are bigger than cursors.

Default-style ProximityPrompts are flipped to `Custom` and briefly reparented so Roblox's built-in UI doesn't stack on top of ours.

## Requiring it

```lua
local ExpressivePrompts = require(path.to.ExpressivePrompts)

-- Optional: override config before Init. Config fields are Seam Values,
-- so use :set() (they stay reactive after Init too).
ExpressivePrompts.Config.BackgroundColor:set(Color3.new(0.1, 0.1, 0.2))

ExpressivePrompts:Init()
```

`init.luau` calls `game.Loaded:Wait()` at require time, so it is safe to require from `ReplicatedFirst`. `:Init()` creates the `ScreenGui` under `PlayerGui` and connects to `ProximityPromptService.PromptShown`.

## API reference

### `ExpressivePrompts:Init()`

Creates the backing `ScreenGui` and starts listening for `PromptShown`. Single-shot: calling it a second time warns and bails.

### `ExpressivePrompts.Config`

A table of Seam `Value`s. Read/write via the Seam API (`:set(...)`); bindings update live, so you can re-theme at runtime.

| Field | Type | Default | Purpose |
|---|---|---|---|
| `BackgroundTransparency` | number | `0.5` | Card background transparency. |
| `BackgroundColor` | Color3 | `(0.07, 0.07, 0.07)` | Card background color. |
| `TextColor` | Color3 | `(1, 1, 1)` | Text and glyph color. |
| `SubTextColor` | Color3 | `(0.7, 0.7, 0.7)` | ObjectText (sub line) color. |
| `CornerRadius` | number | `16` | Pixel corner radius on the card. |
| `MainSizeSpringSpeed` | number | `20` | Spring speed for the press-squish. |
| `MainSizeSpringDampening` | number | `0.4` | Spring dampening for the press-squish. |
| `MainRotationSpringSpeed` | number | `30` | Spring speed for the tilt. |
| `MainRotationSpringDampening` | number | `0.1` | Spring dampening for the tilt. |
| `MainRotationStrength` | number | `5` | Degrees of tilt at rest/while held. |
| `AspectRatioSpringSpeed` | number | `20` | Spring speed for the card aspect-ratio morph. |
| `AspectRatioSpringDampening` | number | `0.4` | Spring dampening for the aspect-ratio morph. |
| `ProgressBarYScale` | number | `0.1` | Height (0-1 scale) of the hold progress bar. |
| `ProgressBarColor` | Color3 | `(1, 1, 1)` | Progress bar color. |
| `ProgressBarTransparency` | number | `0.5` | Progress bar transparency. |
| `ShowShimmer` | boolean | `true` | Whether the shimmer sweep plays on appear. |
| `GuiOffsetSpringSpeed` | number | `15` | Spring speed for the slide-in offset. |
| `GuiOffsetSpringDampening` | number | `0.5` | Spring dampening for the slide-in offset. |

### ProximityPrompt properties honored

Set these on the `ProximityPrompt` instance itself; the UI re-renders on `Prompt.Changed`:

- `ActionText`, `ObjectText` - the two text lines. Leaving both empty gives an icon-only card (72x72).
- `UIOffset` - pixel offset applied to the BillboardGui via `SizeOffset`.
- `HoldDuration` - `> 0` enables the hold progress bar, hold sound, and `PromptButtonHoldBegan/Ended` animation. `0` is a tap-style prompt; the UI fakes a one-frame press for feedback.
- `ClickablePrompt` - when true on non-touch, adds a tap-target covering the gui.
- `Style` - must not be `Default` (the module auto-flips `Default` prompts to `Custom`).
- `KeyboardKeyCode`, `GamepadKeyCode` - drive which glyph is shown.
- `AutoLocalize`, `RootLocalizationTable` - forwarded to the text labels.

### Supported KeyCodes / buttons

- **Keyboard**: anything `UserInputService:GetStringForKeyCode` returns a printable character for, plus any KeyCode mapped in the internal `KeyboardButtonImage`, `KeyboardButtonIconMapping`, or `KeyCodeToTextMapping` tables (covers special keys like Return, Shift, spacebar, arrows, Ctrl/Alt/F-keys). A KeyCode with no glyph, no printable character, and no text override raises an error naming the prompt.
- **Gamepad**: only KeyCodes present in the internal `GamepadButtonImage` table. Unmapped codes silently render without a glyph.
- **Touch**: always shows the built-in tap icon regardless of KeyCode.

### Touch behavior

On touch input the module always renders a full-size invisible `TextButton` over the gui and wires `InputBegan`/`InputEnded` (for `UserInputType.Touch` or `MouseButton1`) into `Prompt:InputHoldBegin()` / `Prompt:InputHoldEnd()`. Non-touch input only gets this tap-target when the prompt sets `ClickablePrompt = true`.

## Examples

### Simple tap prompt

Create a ProximityPrompt in Studio with `HoldDuration = 0`, `ActionText = "Pick Up"`, `ObjectText = "Apple"`, and a `KeyboardKeyCode` of `E`. On the client:

```lua
local ExpressivePrompts = require(ReplicatedStorage.Modules.Client.ExpressivePrompts)
ExpressivePrompts:Init()
```

Tapping E (or tapping the card on touch) fires the prompt's `Triggered` event and plays the click/trigger sounds.

### Hold prompt

Set the ProximityPrompt's `HoldDuration = 1.5` and leave everything else as above. ExpressivePrompts automatically:

- Shows the progress bar filling from 0 to 1 over the hold duration.
- Plays the looped hold sound with `PlaybackSpeed` ramping from `0.5` upward.
- Squishes the card, tilts it, fades the text.
- Plays the trigger sound when the hold completes and the prompt fires `Triggered`.

### Per-input variant (retheme live)

```lua
local ExpressivePrompts = require(ReplicatedStorage.Modules.Client.ExpressivePrompts)

ExpressivePrompts.Config.BackgroundColor:set(Color3.fromRGB(30, 20, 50))
ExpressivePrompts.Config.ProgressBarColor:set(Color3.fromRGB(255, 180, 60))
ExpressivePrompts.Config.CornerRadius:set(24)
ExpressivePrompts.Config.ShowShimmer:set(false)

ExpressivePrompts:Init()

-- Later, from anywhere on the client:
ExpressivePrompts.Config.BackgroundColor:set(Color3.fromRGB(10, 60, 10))
-- All currently visible prompts update immediately.
```

## Gotchas

- **Seam dependency.** The entire UI is built on the vendored Seam UI library (a Fusion-style reactive toolkit). `Config` fields are `Seam.Value`s, not raw Lua values; read/write them through the Seam API (`:set`, etc.) so bindings stay reactive. If terms like `Scope`, `Value`, `Computed`, `Spring`, `New`, `Children`, or `OnEvent` are new, see Seam's upstream documentation. Seam ships vendored under `ExpressivePrompts/Packages/` and should be treated as read-only third-party code.
- `:Init()` is single-shot; the second call just warns.
- Default-style prompts are silently converted to `Custom` and momentarily reparented so no overlapping UI shows up.
- Text measurement uses a fixed 1000x1000 bound, so pathologically long `ActionText`/`ObjectText` won't wrap correctly.
- A keyboard prompt whose `KeyboardKeyCode` has no glyph image, no printable text, and no text override raises an error at creation. Change the KeyCode or extend the internal mapping tables.
- Unmapped gamepad KeyCodes render a prompt with no glyph (intentional silent fallback).
- Sound instances are cached on `SoundService` by SoundId-as-name.

## Related

- `src/ReplicatedStorage/Modules/Client/ExpressivePrompts/Packages/` - vendored Seam UI library (read-only).
- `docs/reviews/client-modules-review.md` - review notes and outstanding fix bullets for this module.
- Roblox `ProximityPromptService` / `ProximityPrompt` - the native surface this module decorates.

## Unclear

- The exact set of KeyCodes covered by `KeyboardButtonImage`, `KeyboardButtonIconMapping`, `KeyCodeToTextMapping`, and `GamepadButtonImage` is defined in sibling modules not read for this doc; consult those tables directly for the authoritative glyph list.
- Whether `ExpressivePrompts` exposes any teardown/`:Destroy()` API: none is present in `init.luau`.
