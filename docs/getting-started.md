# Getting Started

Zero to `framework ready` printed in Studio in about five minutes.

## 1. Install Aftman

The repo pins its toolchain (Rojo, StyLua, Selene) via [Aftman](https://github.com/LPGhatguy/aftman). Install Aftman, then from the repo root:

```bash
aftman install
```

This installs the exact versions listed in `aftman.toml`.

## 2. Clone the repo

```bash
git clone https://github.com/cxnfusing/a_cxnfusing_framework.git
cd a_cxnfusing_framework
```

(Substitute your own fork URL if you forked it.)

## 3. Serve with Rojo

```bash
rojo serve default.project.json
```

Rojo listens on port 34872 by default. Leave this running.

## 4. Connect from Studio

- Open a new or existing place in Roblox Studio.
- Install the Rojo plugin if you don't have it.
- In the Rojo panel, click **Connect**.

Studio now mirrors the `src/` tree into the DataModel. You should see `Framework`, `Shared`, `Modules`, `Services` under `ReplicatedStorage`.

If you'd rather skip Rojo entirely, open `Framework.rbxlx` in Studio directly — every module is already placed.

## 5. Boot the framework

Insert a `Script` into `ServerScriptService` with:

```lua
local rs = game:GetService("ReplicatedStorage")
local Framework = require(rs:WaitForChild("Framework"))
Framework:WaitUntilLoaded()

print("framework ready")
print("version: " .. Framework.Version)
```

Press **Play**. The output window should show the framework's own banner (client and server each log a `✅ framework loaded` line) followed by your `framework ready` + version print.

## 6. Add a LocalScript too

Framework works on the client as well. Insert a `LocalScript` into `StarterPlayer/StarterPlayerScripts`:

```lua
local rs = game:GetService("ReplicatedStorage")
local Framework = require(rs:WaitForChild("Framework"))
Framework:WaitUntilLoaded()

local Trove = Framework.Utilities.Trove
local trove = Trove.new()

trove:Add(function()
    print("bye from the client!")
end)

task.wait(3)
trove:Destroy()
```

## Next

- [Architecture](architecture.md) — learn how Framework, Shared, and tag-loaded modules fit together
- Pick a module page from the [docs index](index.md) and try one of its examples
- Read `src/ReplicatedStorage/Read Me.server.luau` for the author's original tour

## If something is off

- `Framework` is `nil` → Rojo isn't connected or the Rojo mapping is wrong. Check `default.project.json`.
- `framework ready` never prints → one of the auto-loaded modules is erroring during `:Init`/`:Start`. Check the Output window for the first error.
- "Infinite yield on WaitForChild 'Assets'" → the framework expects a folder named `Assets` under `ReplicatedStorage`. Create an empty Folder named `Assets` if your project doesn't have one yet.
- "Infinite yield on WaitForChild 'Data'" → same story for a folder named `Data` under `Shared`.
