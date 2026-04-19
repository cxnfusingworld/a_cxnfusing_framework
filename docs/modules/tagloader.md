# TagLoader

TagLoader is a tiny client-side dispatcher that wires each child ModuleScript to a CollectionService tag whose name matches the module's filename, then calls the module's returned function on every tagged instance (existing and future). It ships with three handlers out of the box: `Preset` (expands a `Preset` attribute into a bundle of attributes from a template), `UIAnimate` (attribute-driven hover/click tweens for GuiObjects), and `AnimatedTexture` (scrolling tiled image backgrounds via the nested `Imagine` engine).

## Overview

The core contract is simple and strict:

- The child module's **filename is the tag**, verbatim. `UIAnimate.luau` binds the `UIAnimate` CollectionService tag. There is no manifest or registry.
- Each child module must return a single function with the shape `(obj: Instance) -> ()`.
- `Preset` is initialized first on purpose, so it can resolve the `Preset` attribute into real attributes before any other handler reads them off the same instance.

Handlers that ship in this folder:

| Handler | Tag | Target Instance | Summary |
|---|---|---|---|
| `Preset` | `Preset` | any Instance | Copies attributes from a template under `Preset/Presets/<Name>` onto the tagged instance, then clears the `Preset` attribute. |
| `UIAnimate` | `UIAnimate` | `GuiObject` (Click effects need `GuiButton`) | Hover/click tweens for scale, rotation, background color, and a "raise" Y-offset, configured by attributes. |
| `AnimatedTexture` | `AnimatedTexture` | `ImageLabel` / `ImageButton` with `ScaleType = Tile` | Scrolls the tiled image every frame using speed/tile attributes. |

## Requiring it

TagLoader is a framework-loaded client module, so in normal use you do not require it directly; the loader calls `TagLoader:Start()` and each handler auto-binds to its tag. To add a new handler, drop a ModuleScript in `ReplicatedStorage/Modules/Client/TagLoader/` that returns `function(obj) ... end`, then tag any instance in Studio with that file's name.

```lua
-- File: ReplicatedStorage/Modules/Client/TagLoader/MyThing.luau
return function(obj)
    print("tagged:", obj:GetFullName())
end
-- In Studio: add the "MyThing" tag to any instance.
```

If you need to invoke TagLoader manually (for example, in a test harness):

```lua
local TagLoader = require(game.ReplicatedStorage.Modules.Client.TagLoader)
TagLoader:Start()
```

## API reference

### `TagLoader:Start()`

Binds every child ModuleScript under `TagLoader/` to its filename-as-tag. Starts `Preset` first, then iterates the remaining children. A `loaded` dedup guard prevents `Preset` from being bound twice. No teardown is provided; bindings live for the session.

### `Preset` handler

- **Tag:** `Preset`
- **Target:** any `Instance`
- **Reads attribute:** `Preset` (string) - name of a child under `TagLoader/Preset/Presets` whose attributes are the template.
- **Writes attributes:** each attribute present on the matching preset template, only if the target does not already have that attribute set.
- **Clears attribute:** `Preset` (set to `nil` after expansion, so re-tagging cannot re-apply).
- **Behavior:** one-shot per instance. Existing attributes on the target win over the preset's values, letting you override per instance. Missing `Presets` folder or missing preset name is a silent no-op.

### `UIAnimate` handler

- **Tag:** `UIAnimate`
- **Target:** `GuiObject`. Click animations only bind if the instance `:IsA("GuiButton")`.
- **Reads attributes** (all optional; opt in by setting any of them):
  - `HoverSize`, `ClickSize` (number, `1` = no change)
  - `HoverRotation`, `ClickRotation` (degrees)
  - `HoverBGColor`, `ClickBGColor` (`Color3`)
  - `HoverRaise`, `ClickRaise` (studs on Y, positive = up)
  - Per-effect timing via suffixes on any of the above: `_EnterLength` (default `0.2`), `_LeaveLength` (`0.2`), `_ClickLength` (`0.1`), `_ReturnLength` (`0.15`). Example: `HoverSize_EnterLength = 0.3`.
- **Writes attributes** (bookkeeping, written on first init):
  - `OriginalPosition`, `OriginalBGColor`, `OriginalRotation` - resting state snapshots.
  - `_clicked` - transient click-in-progress lock (set during click tween, cleared `0.3s` later).
- **Adds children:** a `UIScale` is parented to the target if one is not already present (scale tweens go through it to avoid fighting `UIListLayout`).
- **Defaults** (used when a `Hover*`/`Click*` attribute is present but not set):
  - `HoverSize 1.1`, `ClickSize 0.9`, `HoverRotation 5`, `ClickRotation 15`, `HoverBGColor (45,45,45)`, `ClickBGColor (0,0,0)`, `HoverRaise 5`, `ClickRaise -2`.

### `AnimatedTexture` handler

- **Tag:** `AnimatedTexture`
- **Target:** `ImageLabel` or `ImageButton`. Expects `ScaleType = Tile` and a sensible `TileSize` to scroll meaningfully.
- **Reads attributes** (all optional; unset falls back to `Imagine.DefaultConfig`):
  - `XSpeed`, `YSpeed` (offset units per second; defaults `100`, `100`).
  - `XTileAmount`, `YTileAmount` (used to size the image via `UDim2.fromScale`; defaults `10`, `10`).
- **Writes properties:** `Size` (set to `UDim2.fromScale(XTileAmount, YTileAmount)`) and `Position` (updated every `RenderStepped`).
- **Connects:** one `RunService.RenderStepped` connection per tagged instance; no teardown wired up from the handler.

### `AnimatedTexture.Imagine` (internal)

Usually not called directly, but exposed for power users:

```lua
local Imagine = require(path.to.AnimatedTexture.Imagine)
local img = Imagine.apply(someImageLabel) -- returns nil if base is nil
img:Animate({ XSpeed = 60, YSpeed = 0, XTileAmount = 8, YTileAmount = 8 })
-- later
img:Unapply()
```

- `Imagine.apply(base)` - wraps an `ImageLabel`/`ImageButton`. Returns `nil` silently if `base` is `nil`; guard at the call site.
- `Imagine:Animate(config)` - clones the config, merges over `DefaultConfig`, sizes the image, and starts a `RenderStepped` loop that slides `Position` by `(speed * dt) % tileSize` each frame.
- `Imagine:Unapply()` - disconnects any `RBXScriptConnection` fields on `self`, clears the table, and drops its metatable.

### `AnimatedTexture.Imagine.TableExtended` (internal)

A one-function shim that returns a `table`-like library with an extra `merge(t1, t2)` (shallow, `t2` wins). Falls back to the real stdlib `table` via `__index`, so `clone`, `insert`, etc. still work. Used inside `Imagine` via a deliberate local-name shadow (`local table = require(...)`).

## Examples

### Tag a Frame and watch `Preset` apply

```
-- In Studio:
-- 1. Under ReplicatedStorage/Modules/Client/TagLoader/Preset/Presets,
--    add a Folder named "BigButton" with attributes:
--      HoverSize = 1.2
--      HoverRotation = 10
-- 2. On a Frame in your UI:
--      Tags: "Preset", "UIAnimate"
--      Attribute Preset = "BigButton"
-- On play: Preset runs first, copies HoverSize/HoverRotation onto the Frame,
-- clears the Preset attribute. UIAnimate then reads HoverSize/HoverRotation
-- and starts hover tweens.
```

### Tag an ImageLabel with `AnimatedTexture`

```
-- ImageLabel properties:
--   Image = "rbxassetid://<your tiled texture>"
--   ScaleType = Tile
--   TileSize = UDim2.fromOffset(64, 64)
-- Tags: "AnimatedTexture"
-- Attributes:
--   XSpeed = 80        -- scrolls to the right
--   YSpeed = 0
--   XTileAmount = 12
--   YTileAmount = 6
-- Parent the ImageLabel inside a Frame with ClipsDescendants = true so the
-- oversized tiled surface is masked.
```

### Custom handler

```lua
-- File: ReplicatedStorage/Modules/Client/TagLoader/Glow.luau
return function(obj)
    if not obj:IsA("GuiObject") then return end
    local stroke = Instance.new("UIStroke")
    stroke.Color = obj:GetAttribute("GlowColor") or Color3.new(1, 1, 1)
    stroke.Thickness = obj:GetAttribute("GlowThickness") or 2
    stroke.Parent = obj
end
-- Tag any GuiObject with "Glow" and optionally set GlowColor / GlowThickness.
```

## Gotchas

- **Filename == tag name.** Rename a handler file and every tag on every instance in the game has to be renamed too. No alias layer.
- **Dedup by name, not identity.** `:Start()` pre-calls `initTag` on `Preset` then iterates all children, so the `loaded` table is what stops `Preset` from binding twice. If you add a second `Preset.luau`-named ModuleScript elsewhere and funnel it through, expect double-bind.
- **Attribute pollution on tagged instances.** `Preset` writes the template's attributes directly onto the target, then clears `Preset`. `UIAnimate` writes `OriginalPosition`, `OriginalBGColor`, `OriginalRotation`, and a `_clicked` lock onto the GuiObject. If you inspect attributes in Studio at runtime you will see these; the repo review calls this out as something to move to a weak table later.
- **Existing attributes win in `Preset`.** The expansion skips any attribute the designer has already set on the target. Use this for per-instance overrides; do not expect the preset to force values.
- **`Preset` is one-shot.** The `Preset` attribute is cleared after expansion, so you cannot swap presets at runtime by rewriting the attribute.
- **Client-only.** TagLoader lives under `Modules/Client/` and is loaded via the client-side module loader path; server-side tagging does not run these handlers.
- **No teardown.** Tag-bound callbacks, `UIAnimate` mouse connections, and `AnimatedTexture`'s `RenderStepped` connection live until the instance is destroyed. Hot-reloading in Studio will stack connections. `Imagine:Unapply` exists but nothing wires it up from the handler.
- **`UIAnimate` click cooldown is hard-coded** at `task.wait(0.3)`, not derived from `Click + Return` lengths. Tweening longer than that lets a second click interrupt.
- **`AnimatedTexture` expects `ScaleType = Tile`.** Other `ScaleType`s will not scroll usefully, and the handler does not validate this.
- **`Imagine.apply(nil)` returns `nil`.** The shipped `AnimatedTexture` wrapper early-returns when `obj` is falsy, so this is safe through the tag path, but watch for it if you call `Imagine` directly.
- **Folder must stay flat.** `TagLoader:Start()` iterates every child and calls `require` on it. Non-ModuleScript children or nested folders (other than `Preset/Presets` which is owned by the `Preset` module) will error on require.

## Related

- `Shared.Utilities.ConnectToTag` - the underlying CollectionService subscription TagLoader builds on.
- `docs/reviews/client-modules-review.md` - outstanding review bullets for TagLoader and its handlers.
- `Modules/Client/GameFeel/` - sibling client module for platformer-flavored helpers.
- `Modules/Client/ExpressivePrompts/` - sibling client module for prompt UI.
- `CLAUDE.md` - top-level directory layout and tag-loaded module conventions.
