# Functions

## Overview

`Functions` is the framework's junk drawer of tiny, pure-ish helpers, things that take an input and give you an output without needing a class, state, or lifecycle. Today it holds four entries: `FastTween` (a one-line TweenService wrapper), `FormatNumber` (big numbers into `1.2K`/`3M` shorthand), `AddCommas` (thousands separators), and `GenerateUID` (random dash-free GUID string). If your helper needs connections, instance tracking, or signals it belongs in `Util/` instead; `Functions/` is for the napkin-sized stuff.

## Requiring it

Both surfaces expose the same table, pick based on where you're calling from:

```luau
-- From user Scripts / LocalScripts (full loader)
local Framework = require(game.ReplicatedStorage.Framework)
local FastTween = Framework.Functions.FastTween

-- From a ModuleScript that is itself loaded by the framework
local Shared = require(game.ReplicatedStorage.Shared)
local FormatNumber = Shared.Functions.FormatNumber
```

You can also grab individual files directly at `ReplicatedStorage.Shared.Components.Functions.<Name>`, but going through `Framework.Functions` / `Shared.Functions` is the canonical path.

## API reference

### FastTween

Thin wrapper around `TweenService:Create` + `:Play` that accepts a compact positional array in place of a `TweenInfo`.

**Signature**

```luau
FastTween(obj: Instance, tweenInfo: {any}?, goals: {[string]: any}): Tween?
```

**Args**

- `obj` - the instance to tween. If `nil`, the function silently returns.
- `tweenInfo` - optional positional array matching `TweenInfo.new` order: `{ time, easingStyle, easingDirection, repeatCount, reverses }`. Any nil slot falls back to the defaults `{ 1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, 0, false }`. Easing style and direction may be passed as strings (e.g. `"Quad"`, `"Out"`) and are looked up on the matching Enum.
- `goals` - the tween goals table (same shape you'd pass to `TweenService:Create`). If `nil`, the function silently returns.

**Returns**

- The created `Tween` (already `:Play()`ed), or `nil` if `obj` or `goals` were missing.

**Yields**

- No. Returns immediately after `:Play()`. Connect to `.Completed` on the returned tween if you need to wait.

### FormatNumber

Short, human-readable string with a `K`/`M`/`B`/`T` suffix for big numbers. Negative values keep their sign.

**Signature**

```luau
FormatNumber(value: number): string
```

**Args**

- `value` - any number. Magnitude is compared, so negatives pick up suffixes too.

**Returns**

- A string. Values `< 1000` in magnitude return `tostring(value)` unchanged. Otherwise the largest-fitting suffix wins and the number is formatted to one decimal place; a lone trailing `.0` is stripped.

**Yields**

- No.

### AddCommas

Inserts thousands-separator commas into a number's string form.

**Signature**

```luau
AddCommas(n: number): string
```

**Args**

- `n` - any number. Negative sign is preserved; floats are `tostring`'d and any fractional part rides along unformatted.

**Returns**

- A string with commas inserted every three digits from the right of the integer part.

**Yields**

- No.

### GenerateUID

Wraps `HttpService:GenerateGUID(false)` and strips the dashes so the result is safe to use as an instance name or attribute key.

**Signature**

```luau
GenerateUID(): string
```

**Args**

- None.

**Returns**

- A 32-character hex-ish string with no dashes. Not cryptographically secure; good for throwaway IDs.

**Yields**

- No. `HttpService:GenerateGUID` is local and works on both client and server without any HTTP permissions.

## Examples

### Quick UI fade with FastTween

```luau
local Framework = require(game.ReplicatedStorage.Framework)
local FastTween = Framework.Functions.FastTween

-- Defaults: 1s, Sine, InOut
FastTween(frame, nil, { BackgroundTransparency = 1 })

-- Custom easing via strings
FastTween(part, { 0.5, "Quad", "Out" }, { Position = target })

-- Mixing Enum and string forms, grabbing the tween to listen for completion
local t = FastTween(gui, { 2, Enum.EasingStyle.Back, "InOut" }, {
    Size = UDim2.fromScale(1, 1),
})
if t then
    t.Completed:Connect(function() print("done") end)
end
```

### Formatting big numbers

```luau
local F = require(game.ReplicatedStorage.Shared).Functions

F.FormatNumber(950)       --> "950"
F.FormatNumber(1500)      --> "1.5K"
F.FormatNumber(2000000)   --> "2M"
F.FormatNumber(-3250)     --> "-3.2K"

F.AddCommas(1000)         --> "1,000"
F.AddCommas(1234567)      --> "1,234,567"
F.AddCommas(-9876)        --> "-9,876"
```

### Tagging pooled instances with GenerateUID

```luau
local Shared = require(game.ReplicatedStorage.Shared)
local GenerateUID = Shared.Functions.GenerateUID

local id = GenerateUID()
part:SetAttribute("InstanceId", id)
-- id looks like "5F2BC3A9D0E14B7AA81C4E6F3C12E7A8"
```

## Gotchas

- `FastTween` silently returns when `obj` or `goals` is missing, no warn, no error. Check your arguments yourself if you need to know the call actually ran.
- `FastTween`'s string-to-Enum coercion only runs reliably for `easingStyle` in the current implementation. If you pass the direction as a string and it doesn't take effect, pass `Enum.EasingDirection.Out` explicitly.
- `FastTween`'s positional-array form is handy for one-liners but is not the same shape as `TweenInfo.new`'s argument list at the call site; anything non-trivial (chained delays, complex reverses) is clearer with real `TweenInfo`. There's nothing stopping you from calling `TweenService` directly when this helper gets in the way.
- `FormatNumber` formats to one decimal and only strips a single trailing `.0`. Edge cases like `1.10K` currently stay `1.10K` rather than collapsing to `1.1K`.
- `FormatNumber` passes anything under 1000 through `tostring` unchanged, which includes whatever Lua prints for unusual floats (scientific notation, long decimals). Pre-round if that matters.
- `AddCommas` is integer-oriented. Floats are stringified first, so the fractional part rides along but is not comma-separated.
- `GenerateUID` takes no arguments. If you want the dashed GUID form, call `HttpService:GenerateGUID(false)` yourself. Treat it as unique-enough for runtime tagging, not as a security token.
- All four entries live behind `Framework.Functions` / `Shared.Functions`; the table is frozen upstream, so don't try to monkey-patch new entries at runtime. Add new helpers by dropping a file in `Shared/Components/Functions/` and wiring it into `init.luau` (see the repo CLAUDE.md "How to Add X" section).

## Related

- `Shared.Utilities.TimeFormatter` - similar small-helper spirit, but for seconds-to-string formatting (`MM:SS`, `HH:MM:SS`, smart format).
- `Shared.Utilities.ExtendedLibraries` - extended `math`/`table`/`Random`/`Vector3` helpers for math-ish tasks bigger than `Functions/` handles.
- `Shared.Utilities.Trove` - lifecycle/cleanup container; pair with `FastTween` when you want to auto-cancel tweens on teardown.
- `TweenService` (Roblox) - drop down to this directly when `FastTween`'s shorthand isn't expressive enough (repeat counts, delay chains, callbacks composed with other tweens).
- `HttpService:GenerateGUID` - the thing `GenerateUID` wraps; use it directly if you need the dashed form.
