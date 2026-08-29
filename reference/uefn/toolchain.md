# UEFN Toolchain & Project Reference

Verse-in-UEFN specifics: the build/verify loop, API digests, project/module layout, editor integration, and toolchain-version caveats. Language semantics live in `../language/`; API surfaces in `../apis/`.

## The three API layers

- `/Verse.org/*` — core language libs (native to Verse; documented in `../language/verse-org.md`, plus `/Verse.org/SceneGraph` in `../apis/scene-graph.md`).
- `/UnrealEngine.com/*` — engine: UI, itemization, diagnostics (`../apis/ui.md`, `../apis/itemization.md`, `../apis/engine.md`).
- `/Fortnite.com/*` — gameplay: characters, damage, devices, weapons (`../apis/characters-combat.md`, `../apis/devices.md`, `../apis/itemization.md`).

## Digest = source of truth, and it's versioned

The authoritative signatures live in the generated `*.digest.verse` files for your project's build (`Verse.digest.verse`, `UnrealEngine.digest.verse`, `Fortnite.digest.verse`, plus your project's `Assets.digest.verse`). The digests referenced while writing this skill were build `++Fortnite+Release-42.00-CL-57316517`. **If a symbol/overload isn't in your digest, regenerate the digests from the current build** before assuming it's missing — the engine adds APIs frequently, and many carry `@available{MinUploadedAtFNVersion := N}` gates and `@experimental`. To find a symbol fast, `grep` the digest rather than reading it whole (the Fortnite digest is ~11.5k lines). Bundled digests often lag the build a project targets.

## Build & verify loop

Never report Verse as compiling without building it. With the **`ue-editor` MCP server** connected: `VerseToolset.BuildAll` → fix `Error` diagnostics → repeat; `SessionToolset.PushChanges` + `GetClientLogEntries` for runtime truth. The full protocol is in `SKILL.md` (top section) and the **unreal-mcp** skill. Without the MCP, say the code is unverified.

**Build/LSP state goes stale in BOTH directions** — the LSP misses errors the in-editor build catches, and the build can re-report errors whose fix is already on disk (byte-identical error text to a previous wave is the tell). Before debugging a "surviving" error, rebuild fresh; if it persists, restart the editor. Only then treat it as real. (Parametric-type code can even hide errors and crash the preview session on start — see `../gotchas.md`.)

## Toolchain effect default (current build)

Functions currently default to `<no_rollback>` — every `<decides>` function needs an explicit `<computes>` or `<transacts>`, and so does every helper called from a rollback context. This is the single most build-breaking toolchain fact; the full rules are in `../language/effects-failure-concurrency.md`. It will flip to `<transacts>`-as-default in a future build — update this skill when it does.

## Project & module layout

- **A folder is a module**; declare the nested tree in a `modules.verse` (`Gameplay := module: Stats := module: ...`). Full module semantics: `../language/language.md` → Modules.
- **`Content/Collections` is not usable as a Verse module folder** — it's UE's reserved asset-collections directory. Same caution for other editor-reserved roots (`Developers/`, `__ExternalActors__/`, ...).
- **UEFN Content Browser asset folders are modules too** (package `<Project>/Assets`) — an asset folder named `UI` or `Towers` collides with any code identifier of the same name (3532/3588). See `../gotchas.md` → Mutability & bindings.
- Declare every folder module whose types appear in public APIs as `<public>` in `modules.verse`, or you'll hit 3593 (see `../gotchas.md` → Modules & evolution).
- Packages publish under `/yourname@fortnite.com/ProjectName/...` — cross-package imports use that full path.
- `.versefuture` files are a handy convention for experimental (live-variable) rewrites kept alongside the active `.verse` — not compiled as-is.

## Editor integration

- **`@editable`** fields surface in the UEFN details panel; the attribute family lives in `/Verse.org/Simulation` (`../language/verse-org.md`). Required editable references on components stay uninitialized; `<concrete>` inline-instantiated classes need defaults on every field (`../gotchas.md`).
- `session.Environment()` distinguishes `Edit`/`Private`/`Live`.

## Publishing & persistence constraints

- **Publishing is a permanent contract** — public definitions can't be removed/narrowed, `<castable>`/`<final_super>` are irreversible, persistable schemas only grow. Details: `../gotchas.md` → Modules & evolution and `../language/language.md` → specifier evolution.
- **Hard limit of 4 persistent `weak_map`s per island**; write costs and the per-transaction dedup model: `../gotchas.md` → Scene Graph / engine specifics; serialization format: `../language/language.md` → Persistence.

## The two VMs

`[BetaVerse]` = shippable (BPVM + VerseVM). The VMs differ on integer overflow, class-init order, nested functions, STM completeness, cancellation correctness — prefer the cross-VM-safe form; per-feature details are flagged `[VVM]`/`[BPVM]` throughout `../language/`.
