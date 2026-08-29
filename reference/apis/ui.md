# Screen UI & HUD

Widgets (`/UnrealEngine.com/Temporary/UI`), Fortnite widgets + HUD control (`/Fortnite.com/UI`), and the player-input event layers.

> **Digest = source of truth.** Exact signatures live in your build's generated digests; `grep` them. See `../uefn/toolchain.md`.

## `/UnrealEngine.com/Temporary/UI` — widgets (build screen UI)

`GetPlayerUI[Player]<transacts><decides>→player_ui`:

- **`player_ui`**: `AddWidget(W)` / `AddWidget(W, Slot:player_ui_slot)`, `RemoveWidget(W)`, `SetFocus(W)` (target must be focusable; a `SetFocus` before `AddWidget` applies when the widget is added, unless another widget got focus in between).
- **`widget`** (abstract base): `SetVisibility(widget_visibility)`/`GetVisibility()`, `SetEnabled(logic)`/`IsEnabled()`, `GetParentWidget[]`/`GetRootWidget[]` (both `<transacts><decides>`, fail when not in a `player_ui`).

Concrete widgets:

- Containers: `canvas` (runtime `AddWidget(canvas_slot)`/`RemoveWidget(widget)`; `canvas_slot{Anchors, Offsets:margin, SizeToContent:logic, Alignment:vector2, ZOrder, Widget}`; `MakeCanvasSlot(W, Position, ?Size, ?ZOrder, ?Alignment)<computes>` for fixed-position slots — no `Size` ⇒ `SizeToContent`), `stack_box{Orientation}` (+`stack_box_slot{Widget, HorizontalAlignment, VerticalAlignment, Padding, Distribution:?float}` — set `Distribution` to share space proportionally instead of desired size), `overlay` (stacked on top of each other, +`overlay_slot`), `button` (single-child `button_slot`; `OnClick():listenable(widget_message)`, `HighlightEvent()`/`UnhighlightEvent()` (hover), `var TriggeringInputAction:?input_action(logic)` to trigger Click from an input action).
- Leaves: `text_base` (**abstract** — the concrete `text_block` lives in `/Fortnite.com/UI`, below): `SetText(message)`/`GetText():string`, `SetTextColor/Size/Opacity`, `SetJustification(text_justification)`, `SetOverflowPolicy`, `var AutoWrap:logic`/`var WrapWidth:float` (≤0 = no wrap)/`var WrappingPolicy:text_wrapping_policy`; `texture_block` (`SetImage(texture)`, `SetTint(color)`, `SetDesiredSize(vector2)`, `SetTiling(image_tiling, image_tiling)` `Stretch`/`Repeat`); `material_block` (same shape over `material`); `color_block` (`SetColor`/`SetOpacity`/`SetDesiredSize`).
- Layout types: `anchors{Minimum,Maximum:vector2}` ((0,0)=top-left … (1,1)=bottom-right), `margin{Left,Top,Right,Bottom}` (**units: 1.0 = one pixel at 1080p**), `horizontal_alignment`/`vertical_alignment` (`Center`/`Left|Top`/`Right|Bottom`/`Fill`), `orientation`, `widget_visibility` (`Visible`/`Collapsed` (no layout space)/`Hidden` (keeps layout space)), `text_justification` (`Left`/`Center`/`Right`/`InvariantLeft`/`InvariantRight` — Left/Right flip under RTL cultures), `text_overflow_policy` (`Clip`/`Ellipsis`), `text_wrapping_policy` (`LineBreak`/`PerCharacter`), `player_ui_slot{ZOrder, InputMode:ui_input_mode}` (`None`/`All` — whether the widget consumes input), `widget_message{Player, Source}` (the payload of every widget `listenable`).
- **`vector2`/`vector2i` live in `/UnrealEngine.com/Temporary/SpatialMath`** (X/Y), not the Verse.org SpatialMath.

UI is usually built declaratively in a `<constructor>` with a `let:` block (see `../patterns.md` §6). Widget-facing APIs take `message`, not `string` — every static player-facing string is a dedicated `<localizes>` field (see `../language/language.md` → Localization).

## `/Fortnite.com/UI`

- **`text_block`** — the concrete text widget (`class<final>(text_base)`); adds drop-shadow: `SetShadowOffset(?vector2)`/`SetShadowColor(color)`/`SetShadowOpacity`.
- **Styled buttons**: `text_button_base` (abstract: `DefaultText<localizes>`, `OnClick():listenable(widget_message)`, `SetText(message)`/`GetText()`, `var TriggeringInputAction:?input_action(logic)`) → `button_loud` / `button_regular` / `button_quiet`.
- **`slider_regular`**: `DefaultValue/MinValue/MaxValue/StepSize` + `Set*/Get*` for each (setters clamp/enforce min ≤ max), `OnValueChanged():listenable(widget_message)`.
- **HUD control**: `fort_playspace.GetHUDController()` → `fort_hud_controller` with all-player `ShowElements`/`HideElements`/`ResetElementVisibility` and per-player `ShowElementsForPlayer`/`HideElementsForPlayer`/`ResetElementsForPlayer` (all take `[]hud_element_identifier`; **player-specific rules override general ones** and only `ResetElementsForPlayer` clears them). The old `player_ui.ShowHUDElements`/`HideHUDElements`/`ResetHUDElementVisibility` are `@deprecated` (they affected all players).
- **`hud_element_identifier` catalog**: `creative_hud_identifier_*` (`all`, `build_menu`, `crafting_resources`, `elimination_counter`, `equipped_item`, `experience_level`/`_supercharged`/`_ui`, `health`, `health_numbers`, `hud_info`, `interaction_prompts`, `map_prompts`, `minimap`, `pickup_stream`, `player_count`, `player_inventory`, `round_info`, `round_timer`, `shield_numbers`, `shields`, `storm_notifications`, `storm_timer`, `team_info`), `player_hud_identifier_all`, `hud_identifier_world_resource_*` (`wood`/`stone`/`metal`/`permanite`/`gold_currency`/`ingredient`), `hud_identifier_visual_sound_effect_*` (`weapons`/`loot`/`movement`/`vehicle`/`healing`/`all`). (The digest also ships misspelled duplicates `mimimap`/`shileds` — use the correctly-spelled ones.)

## Player input

The full input stack — `/Verse.org/Input`, `/UnrealEngine.com/ControlInput`, asset types, built-in action/mapping catalogs (incl. the UI-mode mappings with default bindings), and viewport↔world bridging — lives in **`input.md`**. Widget-relevant knobs here: `player_ui_slot{InputMode:ui_input_mode}` (`None`/`All`) controls whether a widget consumes input, and `button`/`text_button_base` can auto-trigger from an action via `var TriggeringInputAction:?input_action(logic)`.
