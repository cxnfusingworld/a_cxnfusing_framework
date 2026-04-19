# DialogueService

## Overview

DialogueService is an opinionated NPC dialogue system built on top of the framework. It wires a ProximityPrompt-triggered "grabber" to a conversation graph, plays the graph through a letter-at-a-time typewriter (with inline effect tags like `<wave>` and `<rainbow>`), streams `$command:arg$` tokens to listeners for side effects (poses, VFX, arbitrary Actions), and shows reply buttons when a node branches. Conversations are plain Luau data files, so authors can write them without touching runtime code.

## Requiring it

DialogueService lives at `ReplicatedStorage.Services.DialogueService`. It does not currently ship a top-level `init.luau`; you require the submodules you need directly.

```lua
local serv        = game.ReplicatedStorage.Services.DialogueService
local Grabber     = require(serv.DialogueGrabber)
local Handler     = require(serv.DialogueHandler)
local Typewriter  = require(serv.Util.Typewriter)
local Animator    = require(serv.Util.Animator)
```

`Typewriter:Init()` must be called once at startup so the per-letter effect modules under `Util/Typewriter/Effects/` are loaded. If you skip it, effect tags will parse but render nothing.

## API reference

### Grabber

File: `DialogueService/DialogueGrabber.luau`

- `grabber:Grab(prompt: ProximityPrompt) -> (conversation: {}?, npc: Model?)` - reads the prompt's `"ID"` attribute, looks it up in `ConversationInfo`, walks any `Conditioned` variants (AND-ing their named condition modules from `Conditions/`), and returns the first match (falling back to `Default`). Returns nothing if the ID has no entry.

Conditions live at `DialogueService/Conditions/` (you create this folder) and each returns `function(ctx) -> boolean` where `ctx = { player, npc }`.

`ConversationInfo.luau` is the routing table:

```lua
Shopkeeper = {
    Default = "ShopGreeting",
    Conditioned = {
        NightQuest = { "IsNight", "HasQuest" }, -- all must pass
    },
},
```

### DialogueHandler

File: `DialogueService/DialogueHandler/init.luau`

Construct one per UI:

```lua
local handler = Handler.new({
    TextType = "Letter" | "InLabel",
    ReplyTemplate = myReplyButton,
    TypewriterConfig = { ... }, -- see Typewriter below
}, myDialogueUI)
```

The instance exposes:

- `handler.OpenUI` / `handler.CloseUI` - signals. `CloseUI` fires when a convo ends. `OpenUI` is declared but not fired automatically; fire it yourself when you kick off a conversation.
- `handler:StartConversation(data, npc) -> Controller` - begins playing a conversation graph on the configured UI.

The returned Controller:

- `Controller:Skip()` - finishes the current typewriter line instantly.
- `Controller:Continue()` - advances past a `WaitingToContinue` state. Debounced ~0.2s after `Skip` so one keypress doesn't do both.
- `Controller:Restart()` - destroys the active typewriter.
- `Controller:Destroy()` - tears the controller down.
- `Controller.StateChanged` - fires with `"Typing" | "WaitingForReply" | "WaitingToContinue" | "Complete"`.
- `Controller.ReplyAdded` / `Controller.ReplyRemoved` - fire with the reply button as it's spawned / removed.
- `Controller.CommandRun` - fires with `(command: string, arguments: string?)` for every `$cmd:arg$` in the text except `wait` (which is handled internally by pausing the typewriter).
- `Controller.LetterTyped` - forwards each typed letter `{ Letter, Index, Effects, Instance? }`.

Required UI children (discovered by name):

- `"DialogueTextLabel"` - only for `TextType = "InLabel"`.
- `"RepliesContainer"` - always; reply buttons are parented here.

### Typewriter

File: `DialogueService/Util/Typewriter/init.luau`

Two render modes:

- `Typewriter:Type(text) -> Controller` - spawns one `TextLabel` per visible character inside `Config.TypeFrame`. Per-letter effects work.
- `Typewriter:TypeInLabel(label, text) -> Controller` - uses `MaxVisibleGraphemes` on a single label. Fast, no per-letter effects.

Setup:

- `Typewriter:Init()` - loads modules under `Effects/`. Call once.
- `Typewriter:SetConfig(cfg)` - merges into the live module-level Config (see Gotchas).

Config fields:

```
TypeFrame, LetterTemplate, TypeSound, SoundStacks, PitchVariation,
PlaySoundOnSpace, DefaultTypeSpeed, PunctuationPauseLength, CommaPauseLength
```

Controller (both modes):

- `OnLetterTyped` - fires per visible character. `:Type` payload is `{Letter, Index, Effects, Instance}`; `:TypeInLabel` omits `Effects`/`Instance`.
- `Completed` - fires when typing reaches the end.
- `:Skip()` - drops remaining per-letter waits.
- `:Cancel()` - stops and destroys.
- `:Destroy()` - clears letters and disconnects effects.
- `:Wait(seconds)` - pauses the loop (used by `$wait:0.5$`).

Punctuation adds `CommaPauseLength` for `, ;` and `PunctuationPauseLength` for `. ? !`.

### Effects list

Effect tags are discovered from `Util/Typewriter/Effects/` and wrap text like `<wave>hi</wave>`. Available:

| Tag | Description |
|-----|-------------|
| `<b>` | Bold font variant (via a small `BOLD_MAP` of common fonts). |
| `<i>` | Italic font variant. |
| `<wobble>` | Small Lissajous-style positional jitter per letter. |
| `<ghost>` | Transparency fade with per-index offset. |
| `<glitch>` | Chromatic-aberration flicker using cloned ghost labels. |
| `<pulse>` | Scale pulse via a `UIScale`. |
| `<rainbow>` | HSV colour cycle across time. |
| `<wave>` | Vertical sine-wave bob. |

Effect tags are valid only in `TextType = "Letter"` mode. Tag names must match `%w+` (word chars only).

### Actions

File: `DialogueService/Actions/Test.luau`

`Actions/` is a folder of optional ModuleScripts that you wire up yourself in response to `Controller.CommandRun`. No loader in DialogueService currently auto-dispatches them; the convention is:

```lua
local Actions = {} -- require each child of Actions/ yourself
controller.CommandRun:Connect(function(cmd, arg)
    local fn = Actions[cmd]
    if fn then fn(arg) end
end)
```

Each action module returns a function that takes whatever args you decide to pass.

### Conversations folder conventions

Each `Conversations/<Name>.luau` returns an array of **nodes**:

```lua
{
    Label = "OnStart",         -- unique id within this convo
    Start = true,              -- optional; marks the entry node
    Type = "Dialogue"|"Reply", -- Dialogue types out, Reply spawns a button
    Text = "...",              -- supports placeholders, tags, commands
    Connections = { "Label1", "Label2" }, -- outgoing edges
    Position = UDim2.fromOffset(0, 0),    -- editor-only hint; runtime ignores
}
```

Rules enforced by the handler:

- Exactly one node should have `Start = true`. If none do, the first array entry is used.
- A node's `Connections` are split by target `Type`: **Reply** targets become buttons for the player; a **Dialogue** target becomes an auto-advance next node. If both exist, replies win.
- A reply node routes to its first `Connections` entry when clicked.
- Text is pre-processed by `DialogueHandler/Replace.luau` which swaps `<plrName>` / `<npcName>` and peels off `$cmd:arg$` commands before the typewriter sees the string.

## Grammar

(Pulled from `Conversations/Info.luau`, which is a grammar cheat sheet. **Do not register `Info` in `ConversationInfo.luau`** - it has no `return` and will crash the handler if routed to.)

Variables (resolved by `Replace.luau`):

- `<plrName>` - the local player's `DisplayName`.
- `<npcName>` - the NPC model's `Name`.

Commands (stripped from text and fired through `Controller.CommandRun`, except `wait` which is handled internally):

- `$pose:PoseName$` - convention for swapping the NPC's pose; your listener implements it (see the Animator example).
- `$wait:Seconds$` - pauses the typewriter for that many seconds. Handled inside the Typewriter itself.
- `$anyCommand:anyArg$` - any other name is forwarded as `(command, arguments)` for you to handle.

A command placed before the first visible character fires up-front (index 0). Commands anchored past the last letter are flushed after `Completed`.

Effect tags (see Effects list above): `<b>`, `<i>`, `<wave>`, `<rainbow>`, `<wobble>`, `<pulse>`, `<ghost>`, `<glitch>`. All use the `<name>...</name>` form.

Literal `$`: a lone `$` without a matching closing `$` is rendered as a literal dollar sign.

## Examples

### Minimal conversation with a reply tree

From `Conversations/Greeting.luau`:

```lua
return {
    {
        Label = "OnStart", Start = true, Type = "Dialogue",
        Text = [[Hey$pose:Greet$ I'm <b><npcName></b>!
$pose:Idle$What's your name?]],
        Connections = { "Name", "Decline" },
    },
    {
        Label = "Name", Type = "Reply",
        Text = [[I'm <b><plrName></b>!]],
        Connections = { "FollowUp" },
    },
    {
        Label = "Decline", Type = "Reply",
        Text = [[Umm... bye!]],
        Connections = { },
    },
    {
        Label = "FollowUp", Type = "Dialogue",
        Text = [[Nice to meet you <wave><rainbow><b><plrName></b>!</rainbow></wave>
I'll see you around.]],
        Connections = { },
    },
}
```

### Wiring it up

```lua
local Players = game:GetService("Players")
local UIS     = game:GetService("UserInputService")

Typewriter:Init()

local handler = Handler.new({
    TextType = "Letter",
    ReplyTemplate = myReplyButton,
    TypewriterConfig = {
        TypeFrame = dialogueFrame,
        LetterTemplate = letterTemplate,
        TypeSound = typeSound,
        SoundStacks = true,
        PitchVariation = { 0.95, 1.05 },
        PlaySoundOnSpace = false,
        DefaultTypeSpeed = 0.03,
        CommaPauseLength = 0.2,
        PunctuationPauseLength = 0.5,
    },
}, myDialogueUI)

handler.CloseUI:Connect(function() myDialogueUI.Visible = false end)

prompt.Triggered:Connect(function()
    local convo, npc = Grabber:Grab(prompt)
    if not convo then return end

    myDialogueUI.Visible = true
    Animator:SetUp(npc)
    Animator:LoadAnims(npc, npc:WaitForChild("Anims"))

    local controller = handler:StartConversation(convo, npc)

    controller.CommandRun:Connect(function(cmd, arg)
        if cmd == "pose" then
            Animator:StopAllAnims(npc)
            Animator:PlayAnim(npc, arg)
        end
    end)

    UIS.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == Enum.KeyCode.E then
            controller:Skip()
            controller:Continue()
        end
    end)
end)
```

## Gotchas

- **Module-level shared Typewriter Config.** `Typewriter` stores its Config as a module-level table, and `:SetConfig` merges into it. Two UIs cannot type simultaneously with different settings; the last `:SetConfig` wins for everyone. DialogueHandler calls `:SetConfig` each time `StartConversation` runs.
- **Busy-wait on replies and continue.** Both the reply selection and the continue-gate use `repeat task.wait() until ...`. It works, but idle dialogue polls every frame rather than waiting on a signal.
- **Effect leaks on Completed.** Typewriter disconnects effect `RenderStepped` connections on `:Destroy`, not on normal `Completed`. Effects that spawn side-effect Instances (`glitch` clones labels, `pulse` creates a `UIScale`) keep those Instances around unless the controller is destroyed.
- **`Test` action prints a joke.** `Actions/Test.luau` is a stub that prints `"HAHAHAHAHAHA"`. Replace or remove it before shipping; the framework does not auto-wire it, but it is trivial to invoke accidentally.
- **`Animator` state is module-level.** `animators` and `anims` are shared across every NPC. Two NPCs with identically-named animations collide - the second `:LoadAnims` overwrites the first's track.
- **`Conversations/Info.luau` returns nothing.** It's documentation, not a conversation. Do not add `Info` to `ConversationInfo.luau`.
- **Silent failures.** `Grab` returns nothing when the prompt's `ID` is missing or unmapped; `Typewriter:Type` warns and returns nil if `TypeFrame` is unset; typos in a node's `Connections` silently end the conversation.
- **`OpenUI` is never auto-fired.** Only `CloseUI` fires at the end of a convo. Show your UI yourself when you call `StartConversation`.
- **Multi-byte graphemes in `:TypeInLabel`.** Per-letter `OnLetterTyped` indexes use `contentText:sub(i, i)`, which is byte-wise, not grapheme-wise. Fine for ASCII, breaks on some Unicode.
- **Non-deterministic `Conditioned` ordering.** `Grabber` iterates `info.Conditioned` with `pairs`; when two variants could both pass, the winner is not deterministic.

## Unclear

- There is no top-level `DialogueService/init.luau`, so how the framework's `ModuleLoader` treats this subtree (priority, init ordering, whether `Typewriter:Init` is called automatically) is not defined in the files reviewed.
- The `Conditions/` folder is referenced by `DialogueGrabber` via `WaitForChild` but is not present in the reviewed sources; its expected module shape is inferred from usage (`function(ctx) -> boolean`).

## Related

- `StarterGui/Dialogue/` - example UI handler that consumes DialogueService (outside this module tree).
- `Shared.Util.Signal` - the framework's canonical signal. DialogueService vendors its own copy locally.
- `docs/reviews/services-review.md` - open-source readiness review with per-file fix bullets.
