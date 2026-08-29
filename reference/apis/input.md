# Player Input

The full input stack, spanning three layers: asset types in `/Verse.org/Assets` (`input_action(t)`, `input_mapping`), the current API in `/Verse.org/Input` `[4000+]`, its engine-level predecessor `/UnrealEngine.com/ControlInput` `[3630+]`, and built-in action/mapping catalogs (`/Fortnite.com/Input/Character`, `/Verse.org/Input/UI` and `/Gameplay`). Widget-side input consumption (`ui_input_mode`) is in `ui.md`.

> **Digest = source of truth.** Exact signatures live in your build's generated digests; `grep` them. See `../uefn/toolchain.md`. Verified against build 42.00.

## The asset types (`/Verse.org/Assets`)

- **`input_action(t)`** — `class<allocates><unique><computes><epic_internal>`: **asset-only**. You author Input Action assets in UEFN (Content Browser) and reference them via `@editable MyAction:input_action(logic)` or use the built-in module constants below — you can never construct one in code. `t` is the value type configured on the asset (digital = `logic`; pointer actions use `vector3`). `<unique>` means actions compare by identity.
- **`input_mapping`** — `class<computes><epic_internal>`: an asset bundling actions with their key/button/touch bindings. Same deal: asset-authored or built-in constant, `@editable` to reference.

## Acquisition & `player_input` (`/Verse.org/Input` `[4000+]`)

```verse
if (Input := GetPlayerInput[Player]):        # (player)<transacts><decides>:player_input
    Input.AddInputMapping(MyMapping)         # :void — enables the mapping's actions for THIS player
    Events := Input.GetInputEvents(MyAction) # (input_action(t)):input_events(t)
```

- `AddInputMapping`/`RemoveInputMapping(input_mapping):void` — per-player, no_rollback (plain `:void` — hoist out of `if`/`for` contexts). **An `input_action` only fires for a player while ≥1 active mapping on that player references it** — forgetting `AddInputMapping` is the #1 "why no events" cause.
- `var PreferredInputMethod:input_method` — `enum<open>`: `KeyboardAndMouse`/`Gamepad`/`Touch`; updates when the player switches. **There is no changed-event** — read it when you need it (e.g. on menu open), don't expect a `listenable`.
- `var AvailableInputDevices:available_input_devices` — struct of `logic` flags `Gamepad`/`Keyboard`/`Mouse`/`Touch`, live-updated on connect/disconnect. `Keyboard` and `Mouse` always carry the same value (engine groups them as one capability).

## `input_events(t)` — the detection→activation state machine

```
BeginDetectEvent -> DetectionOngoingEvent -> TriggerActivationEvent -> EndDetectEvent
        \________________(early release)___> CancelActivationEvent __________/
```

| Event | Payload | Fires when |
|---|---|---|
| `TriggerActivationEvent` | `(player, t)` | all conditions met — **bind this one** in the common case |
| `CancelActivationEvent` | `(player, t, float elapsed)` | canceled before activation (e.g. released before a press-and-hold threshold) |
| `BeginDetectEvent` | `(player, t)` | detection starts (required key now down) |
| `DetectionOngoingEvent` | `(player, t, float elapsed)` | still processing (threshold not yet met) |
| `EndDetectEvent` | `(player, float elapsed)` | detection finished (no required keys down) |

All are `listenable(tuple(...))`. Ordering guarantees (from the digest): `BeginDetectEvent` and `TriggerActivationEvent` may fire on the **same frame**, but Begin always fires first; **Begin/End always fire as a pair** regardless of success or cancel. The `.Await()` on these events needs a primer `Subscribe` first — workaround in `../patterns.md` §8.

## `/UnrealEngine.com/ControlInput` `[3630+]` — the engine-level twin

Same `GetPlayerInput[Player]`/`player_input`/`input_events(t)` shape and the same state machine, but: **different event names**, and its `player_input` has only the three core methods — no `PreferredInputMethod`/`AvailableInputDevices`. The types are distinct — don't mix the modules; prefer `/Verse.org/Input`.

| `/Verse.org/Input` | `/UnrealEngine.com/ControlInput` |
|---|---|
| `BeginDetectEvent` | `DetectionBeginEvent` |
| `DetectionOngoingEvent` | `DetectionOngoingEvent` |
| `TriggerActivationEvent` | `ActivationTriggeredEvent` |
| `CancelActivationEvent` | `ActivationCanceledEvent` |
| `EndDetectEvent` | `DetectionEndEvent` |

## Built-in mappings & actions (bind to these before authoring your own)

All actions are `input_action(logic)` unless noted; default bindings from the digest comments (KBM/gamepad).

- **`/Fortnite.com/Input/Character`** `[3720+]`: `RangedWeaponMapping` — `Reload`, `WeaponPrimary`, `WeaponSecondary`; `TraversalMapping` — `Crouch`, `Sprint`, `Jump`. (Listen to the player's existing combat/movement input without defining assets.)
- **`/Verse.org/Input/UI`** `[4000+]` — active in UI mode when bound to an interactive UI element:
  - `MenuNavigationMapping`: `NextTab` (E / RShoulder), `PreviousTab` (Q / LShoulder), `NextPage` (C / RTrigger), `PreviousPage` (Z / LTrigger), `Back` `[4030+]` (Esc / Back).
  - `InventoryMenuMapping`: `Use` (F / FaceTop), `Inspect` (I / FaceLeft), `Sort` (V / RThumb), `Drop` (X / LThumb).
  - `CraftingMenuMapping`: `Craft` (F / FaceTop), `Inspect`, `Favorite` (V / RThumb), `Scrap` (X / LThumb).
  - `MapMenuMapping`: `Track` (F), `PlaceMarker` (P), `Reset` (R), `ToggleView` (V), `ZoomIn` (Z), `ZoomOut` (X).
  - `TouchMapping` `[4100+]` + `PointerSelect:input_action(vector3)` `[4100+]`, `PointerZoom:input_action(vector3)` `[4120+ experimental]` — pointer/touch positions as viewport-space vectors (pair with deprojection below).
- **`/Verse.org/Input/Gameplay`** `[4110+]`: `HotbarMapping` (mapping constant only — no individual actions exposed).

## Viewport ↔ world bridging `[4100+]` (`/Verse.org/Input`)

- `(Player).ProjectWorldToViewport[WorldPos]<decides><reads>:vector3` — viewport coords in **centimeters**: `Left` = horizontal (positive rightward from the top-left corner), `Up` = vertical (positive upward, so on-screen points below the top have negative `Up`), `Forward` always 0. **Fails when the position is behind the camera.**
- `(Player).DeprojectViewportToWorld(ViewportPos)<reads>:deproject_results` — only `Left`/`Up` are read (`Forward` ignored; depth is unknown). Returns `Origin` (**the camera eye**, not the near plane — a trace from here can hit geometry that isn't visible on screen; filter out the player pawn or advance past the near clip) + `Direction` (unit ray) — trace the ray yourself to find the world hit.

Both use `/Verse.org/SpatialMath` vectors. Typical touch-select flow: `PointerSelect` activation → `DeprojectViewportToWorld` → `FindSweepHits` along the ray (`scene-graph.md`).

## Related

- Whether a widget consumes input at all: `player_ui_slot{InputMode:ui_input_mode}` (`None`/`All`) — `ui.md`.
- Await-primer workaround for input events: `../patterns.md` §8.
