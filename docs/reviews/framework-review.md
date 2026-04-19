# Framework core Review

## Summary
The Framework core is small, readable, and does what it advertises: aggregates the `Shared` surface, discovers modules by folder and by `LoadModule` tag, runs `:Init()`/`:Start()`, and exposes a loaded signal. For a senior audience the code is easy to follow, but for the mixed-skill open-source audience there are rough edges: no error isolation around `require`/`Init`/`Start` (one bad module brings everything down), a couple of fragile boolean expressions in `ModuleLoader` and `Log`, inconsistent use of `WaitForChild` between client and server paths, undocumented behavior of the `Type` attribute on container instances, and API surface typing that lies about what is accepted. None of the issues are fatal, but most are easy wins that would noticeably raise the floor for new programmers copying this into a project.

## Modules

### Framework
Path: `src/ReplicatedStorage/Framework/init.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 7 - Sectioned with `---<>---` banners and small obvious helpers, but the duplicated client/server log block and commented-out `freeze(Framework)` add noise.
- Robustness: 5 - No pcall around the module loader pipeline, direct `sss.Modules` indexing can error on fresh projects that lack the folder, and `Loaded = true` still runs even if loading partially failed.
- API design: 7 - `:WaitUntilLoaded`, `:IsServer/Client/Studio`, `:Log`, `:SafeCall`, `:Preload` is a sensible surface, but `:Log`'s type signature omits the "print" case and `:Preload`'s callback has no return type documented.
- Noob-friendliness: 7 - Easy to read top-to-bottom, but newcomers will not know why `Framework` vs `Shared` exists or when to pick which, and there is no top-of-file comment explaining it.
- Style consistency: 6 - Mixes `Framework.Services.RunService:IsClient()` (line 123) with `Framework:IsClient()` (line 93) for the same check, and trailing whitespace / blank-line patterns are uneven.

Fixes:
- [ ] `src/ReplicatedStorage/Framework/init.luau:90` - replace the success `Log(..., "warn")` calls (lines 124 and 136) with a non-warn log level so the success banner does not appear as a yellow warning in the Output window.
- [ ] `src/ReplicatedStorage/Framework/init.luau:91` - drop the `local toLoad = {}` initializer and assign directly inside the if/else to avoid the throwaway empty table.
- [ ] `src/ReplicatedStorage/Framework/init.luau:96` - use `sss:WaitForChild("Modules")`, `sss:WaitForChild("Services")`, and `rsMods:WaitForChild("Combined")` for parity with the client branch and to avoid first-run races.
- [ ] `src/ReplicatedStorage/Framework/init.luau:71` - widen the `logType` annotation to `"warn" | "error" | nil` (or `"print"`) so the print path is part of the documented contract.
- [ ] `src/ReplicatedStorage/Framework/init.luau:75` - fix the `SafeCall` annotation to `(func: (...any) -> ...any, ...any) -> ...any` (or whatever `DebugUtils.SafeCall` really is) and forward `...` into the inner call.
- [ ] `src/ReplicatedStorage/Framework/init.luau:79` - forward the `callback` with its real signature `(contentId: string, status: Enum.AssetFetchStatus) -> ()` and document that `IDs` can also be Instances per `ContentProvider:PreloadAsync`.
- [ ] `src/ReplicatedStorage/Framework/init.luau:119` - delete the commented-out `--freeze(Framework)` line; either freeze the outer table or remove the reference.
- [ ] `src/ReplicatedStorage/Framework/init.luau:116` - replace the `t.Fire and t.Connect` signal sniff with an explicit metatable or `__signal` marker check so a future utility table happening to have both keys is not silently skipped.
- [ ] `src/ReplicatedStorage/Framework/init.luau:123` - collapse the two near-identical `Framework:Log(...)` success blocks into one by computing the prefix from `Framework:IsClient()` once.
- [ ] `src/ReplicatedStorage/Framework/init.luau:1` - add a short header comment describing the `Framework` vs `Shared` split and when to require each, since the docs are currently only in the repo README.
- [ ] `src/ReplicatedStorage/Framework/init.luau:104` - wrap `ModuleLoader:LoadAll()` in a `xpcall` or rely on per-module pcall so a single failing `:Init()`/`:Start()` still lets `Loaded` fire.
- [ ] `src/ReplicatedStorage/Framework/init.luau:51` - document (or rename to `OnLoaded` stays, but comment) that `OnLoaded` fires exactly once with the total load time in seconds, so consumers know the payload.

### Log
Path: `src/ReplicatedStorage/Framework/Dependencies/Log.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - The intent is obvious but the gate expression on lines 9-12 mixes `or`/`and` across three conditions without parentheses around each group, which is easy to misread.
- Robustness: 6 - `error(errorFinal, 3)` uses a hardcoded stack level that is wrong depending on whether the caller went through `Framework:Log` or called `log` directly, and there is no guard against non-string `title`/`info`.
- API design: 6 - Returns a single function rather than a module table, which is fine, but the `logType` is stringly-typed with no exported enum/constants and no "info"/"debug" level.
- Noob-friendliness: 7 - A single function with three arguments is easy to grasp, but the big heredoc formatting is surprising output and there is no example in-file.
- Style consistency: 6 - Uses `require(...)` inline at the top plus local `isClient` helper, while `ModuleLoader` caches `isClient` as a boolean; these two neighboring files should agree.

Fixes:
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:9` - wrap each condition fully, e.g. `if (logType == "error" and not Config.ErrorLogsEnabled) or (logType == "warn" and not Config.WarningLogsEnabled) or (logType == nil and not Config.LogsEnabled) then` so operator precedence is explicit and the print-gate fires on `nil` specifically instead of `not logType`.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:14` - coerce `title` and `info` via `tostring(title or "Untitled")` so callers passing tables/numbers do not blow up inside `string.format`.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:46` - take the stack `level` as an optional fourth argument (default 3) so `Framework:Log` can pass `4` and point at user code instead of the wrapper.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:4` - replace the per-call `isClient()` helper with a single cached `local isClient = game:GetService("RunService"):IsClient()` at module scope (matching `ModuleLoader.luau`).
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:17` - dedent the `printFormat`/`errorFormat` heredocs so the Output window is not full of leading tabs; or build the string with `table.concat` for readability.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:8` - export a small `LogType` table (`{ Warn = "warn", Error = "error", Print = nil }`) or a string literal type alias so downstream callers get autocomplete instead of guessing literals.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/Log.luau:1` - remove the `require(Shared)` dependency and pass `Config` in from `Framework/init.luau`, so `Log` stays usable from contexts where `Shared` has not finished loading yet.

### ModuleLoader
Path: `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau`

Scores (1-10, 1 = bad, 10 = great):
- Clarity: 6 - Small and linear, but `collectModules`'s Type-attribute check on line 20 is hard to read and the module-level `loaded`/`requiredModules` tables are hidden globals relative to the `Loader` table they back.
- Robustness: 4 - Nothing is wrapped in pcall: a single failing `require`, `:Init`, or `:Start` aborts the whole framework load, and there is no dedupe for modules required twice across `RequireAll` calls on overlapping roots (beyond the `GetFullName` map, which is string-keyed and expensive).
- API design: 6 - `:RequireAll`, `:RequireTagged`, `:LoadAll`, `:GetLoaded` is a reasonable split, but `:LoadAll` can only be called once meaningfully (sort runs over all accumulated modules every call) and `:GetLoaded` returns the live internal table by reference.
- Noob-friendliness: 5 - New users will not know that tagging a module after `LoadAll` requires it but never runs its `:Init`/`:Start`, nor that putting the `Type="Server"` attribute on a *folder* silently skips every descendant.
- Style consistency: 7 - Consistent `ipairs` use and naming, but mixes `if ... then return end` one-liners with multiline ifs in adjacent blocks.

Fixes:
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:20` - rewrite as `if rootType == "Client" and not isClient then return end` / `elseif rootType == "Server" and isClient then return end` so precedence is obvious and the redundant `rootType and ...` guard disappears.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:43` - wrap `require(module)` in `xpcall` and log via `Log` on failure, so one broken ModuleScript does not break the whole framework.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:69` - wrap `module:Init()` and `module:Start()` (lines 69 and 74) in `pcall` (or a `SafeCall` helper) and report the offending module name through `Log`.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:41` - key `loaded` by the `module` instance itself rather than `module:GetFullName()` so rename/reparent during play testing does not produce duplicate requires.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:93` - in `RequireTagged`, after `LoadAll` has already run, also call `:Init`/`:Start` on newly-tagged modules (guarded by a `self.Started` flag) so post-load tagging is not silently half-loaded.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:54` - split `LoadAll` into `Init` and `Start` stages, or assert on second call, so accidental double-invocation does not re-run every module's lifecycle.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:99` - return a shallow copy from `GetLoaded` (`return table.clone(loaded)`) so callers cannot mutate loader internals.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:17` - document on `collectModules` that putting `Type` on a non-ModuleScript container skips the whole subtree, since that is the single most surprising behavior for new users.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:3` - move `loaded` and `requiredModules` onto the `Loader` table (or into a closure-scoped state object) so tests can reset them between runs; right now they persist for the life of the VM.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:94` - rename the `mod` parameter in the `ConnectToTag` callback to `tagged` or `taggedInstance`, since a tagged instance may be a folder that is merely a container and the current name implies it is itself the ModuleScript.
- [ ] `src/ReplicatedStorage/Framework/Dependencies/ModuleLoader.luau:64` - define `Priority` as "higher runs earlier" in a comment next to the `sort`, since the doc says so but the file does not.

## Cross-cutting Issues
- No error isolation anywhere in the load pipeline: `require`, `:Init`, and `:Start` all run bare. For an open-source audience, a single typo in a user module kills the whole framework silently from the user's perspective. A shared `SafeCall`-wrapped helper used in both `ModuleLoader` and `init.luau` would solve this in one place.
- `WaitForChild` usage is inconsistent between the client and server branches of `Framework/init.luau`, and `Log.luau` reaches across and requires `Shared` on its own instead of being injected. Pick one convention (inject dependencies at the top of `Framework/init.luau`) and apply it to both `Log` and `ModuleLoader`.
- The `Type` attribute and `NonRecursive` / `NoLoad` tag semantics are only documented in the repo README and project CLAUDE.md - the module that implements them has no inline explanation. New contributors looking at `ModuleLoader.luau` alone will not discover that `Type` on a folder skips the entire subtree.
- `Framework.Loaded` is set to `true` unconditionally at the end of `init.luau`, and `OnLoaded` is fired once. There is no signal for load failure or partial load. Either document this explicitly or add a `LoadErrors` table that `:WaitUntilLoaded` consumers can inspect.
- Typed annotations are partially applied (`Framework:Log`, `Framework:SafeCall`, `collectModules`) but inaccurate (`logType` union misses `nil`, `SafeCall` drops varargs). Either commit to `--!strict` types across these three files or strip the partial ones to avoid misleading IntelliSense for new programmers.
- Log output uses emoji and heavy indentation. That is on-brand with the casual voice the project documents, but it will render awkwardly in non-Studio consoles (CI, rbx-cli). Consider gating the fancy format behind `Config.PrettyLogs` so headless consumers can get plain lines.
