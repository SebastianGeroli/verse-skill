# Asset Reflection — how UEFN assets become Verse API

Every UEFN project generates an `Assets.digest.verse` reflecting Content Browser assets into Verse classes and constants. This is how you reference materials, meshes, particle systems, prefabs, WBP widgets, textures, sounds and animations from code — and the digest is the only reliable way to learn the generated names and shapes.

> **Digest = source of truth.** Open your project's `Assets.digest.verse` (see `toolchain.md`); it regenerates per build, and a published island keeps a separate `<Project>-Published-Assets` snapshot digest alongside the working one.

## Overall shape

- **Content Browser folders are modules**, mirrored as nested `module:` blocks; assets at the content root land at digest file-root. This is why an asset folder named `UI` or `Towers` collides with any code identifier of the same name (`../gotchas.md` → Mutability & bindings).
- **Everything is `<scoped {/youraccount@fortnite.com/Project}>`** — package-private, usable from your Verse code but never publishable in a public API signature. Projects not claimed by an account/team show the placeholder `<scoped {/invaliddomain/Project}>`.
- Two artifact kinds per asset, often **paired**: a plain constant `X_asset:some_type = external {}` (pass to APIs taking the asset *value* type) and/or a generated `X := class(...)` (instantiate/configure in code). `external {}` means "value provided by the engine".
- **`@import_as("/Project/Path/BP_X.BP_X_C")`** binds a generated class to its Blueprint class — informational; you use the Verse name.
- **Auto-qualified names**: when reflection would collide with a sibling (e.g. a material named `MatID_1` next to mesh slots also named `MatID_1`), the digest declares it qualified — `var (/Project/Assets/Pickaxe/SomeMesh:)MatID_1<public>:material`. Copy the qualified form at *every* use, exactly as printed (same rule as `../gotchas.md` qualification).

## Reflection shapes by asset kind

| Asset | Generates | Members |
|---|---|---|
| **Material (instance or parent)** | `M_X := class<scoped{...}>(material)` | each overridable parameter → `@editable var Param<public>:t = external {}`; scalar→`float`, vector→`color`, texture→`texture`. Parameterless materials get an empty body. |
| **Static mesh** | `SM_X_asset:mesh = external {}` **and** `SM_X := class<final>(mesh_component)` | material slots → `@editable var SlotName<public>:material` (often auto-qualified, see above) |
| **Niagara system** | `NS_X_asset:particle_system = external {}` **and** `NS_X := class<public>(particle_system_component)` | user-exposed Niagara parameters → `@editable var Param<public>:t` |
| **Entity prefab** | `EP_X := class<final><concrete>(entity)` (empty body) **and** `EP_X_asset:entity_prefab = external {}` | none — instantiate `EP_X{}` and `Parent.AddEntities(array{...})` |
| **Widget Blueprint (WBP/UMG)** | `UW_X := class(widget)` | exposed Blueprint variables → `var X<public>:t` (Text→`message`, bool→`logic`, LinearColor→`color`, float/texture/material as-is). No events/functions reflect — interactivity stays in the BP graph; read state back via `logic` vars (`WasClicked`-style). |
| **Sound (audio asset setup)** | `X := class<final><public>(sound_component)` | `@editable var PitchBase/PitchRandomSpread:float`, `var Sounds:[]sound_wave` |
| **Texture** | `T_X:texture = external {}` constant | — |
| **Animation sequence** | `X:animation_sequence = external {}` constant | — |

## Using reflected assets

- **Materials**: instantiate the generated class with an archetype and mutate params with `set` — `Mat := M_Highlight{}` … `set Mat.ValidColor = NamedColors.Green` — then apply: on a Scene Graph mesh, assign the reflected slot var (`set MyMesh.MatID_1 = Mat`, or bind at construction); on a `creative_prop`, `Prop.SetMaterial(Mat, ?Index)`. The base `material`/`particle_system`/`animation_sequence` classes are `<epic_internal>` — only reflected subclasses/constants give you values.
- **Meshes**: the `_asset:mesh` constant feeds `creative_prop.SetMesh` and `mesh`-typed `@editable`s; the `mesh_component` subclass is what you `AddComponents` to an entity.
- **Particles**: `SpawnParticleSystem(NS_X_asset, Position, ...)` (`/UnrealEngine.com/Assets`) for fire-and-forget; add the `particle_system_component` subclass to an entity for a configured, controllable emitter (`Play`/`Stop`).
- **WBP widgets**: construct via archetype (set the exposed vars) and add like any widget (`player_ui.AddWidget`, container slots — `../apis/ui.md`).
- **Textures**: `texture_block.SetImage(T_X)`, `has_icon`, `@editable` texture fields. **Animations**: feed `PlaySkeletalAnimation` / `PlayAnimation` (`../apis/scene-graph.md`, `../apis/characters-combat.md`).

## Prefer UMG over Verse-built UI

Default to authoring UI as a Widget Blueprint (UMG) and reflecting it into Verse, rather than building the layout in Verse with `canvas`/`stack_box`/`button_regular` etc. This is a standing preference, not a per-task call — apply it on every new UI screen or widget unless the exception below applies.

- **Static structure → UMG.** A screen with a fixed set of children known at design time (a title, a row of buttons, a background panel) belongs in a WBP: better visual control, and non-programmers can iterate on it without touching Verse.
- **Dynamic/runtime-composed structure → Verse.** Only build in Verse when the children are added, removed, or varied at runtime in a way UMG's static tree can't express — a paginated list of rows whose count depends on game state (quest list, inventory grid), a container that gets `AddWidget`/`RemoveWidget` calls in a loop. The individual row/slot can itself still be a WBP (e.g. a `WBP_ItemSlot` instantiated once per row); it's the *container's* dynamic composition that stays in Verse.
- **Interactivity needs Verse Fields, not raw reflection.** A plain WBP reflects with no events (`asset-reflection.md` → Reflection shapes: "No events/functions reflect"). To make a button drive Verse code, add a Verse Field on the WBP via the **VerseFieldsToolset** (an `event(tuple())` for a no-payload click, `event(tuple(int))` for one carrying an index, etc.) and bind it to the button's OnClicked in the Blueprint graph via MVVM. A good shape for a clickable WBP slot is a `ClickedEvent<public>:event(tuple(...))` field, plus plain `var X<public>:message/logic/texture/int` fields for the row's display data. Confirm the exact reflected types in `Assets.digest.verse`. On the Verse side, instantiate `WBP_X{}`, `set` the display vars, then consume the event. **A reflected event field is a plain `event(t)`, not a `listenable(t)`, so it has no `Subscribe`.** Await it in a loop inside a task whose lifetime matches the widget's:
  ```verse
  # Run with spawn/branch while the widget is shown; cancelling the task ends the listener.
  ListenForClicks(Slot:WBP_X)<suspends>:void=
      loop:
          Payload := Slot.ClickedEvent.Await()
          OnSlotClicked(Payload)
  ```
  If many call sites need callback-style subscription (so the result can sit in a `[]cancelable` alongside real `Subscribe` handles), add a small reusable helper to the project. This version compiles, doesn't poll, and leaves nothing running once it's disposed:
  ```verse
  # Lets a disposer end a listener immediately, from a non-transactional context.
  stoppable<public> := interface<castable>:
      Stop<public>():void

  # Cancel is <transacts>, where neither Signal nor spawn is allowed, so it can only flip a flag
  # (the listener then exits on the next signal). Stop signals, ending the listener right away.
  event_subscription := class(cancelable, stoppable):
      var <private>Canceled<public> : logic = false
      StopEvent<private> : event() = event(){}

      Cancel<override>()<transacts>:void=
          set Canceled = true

      Stop<override>():void=
          set Canceled = true
          StopEvent.Signal()

      AwaitStop()<suspends>:void=
          StopEvent.Await()

  (Event:event(t) where t:type).SubscribeEvent<public>(Callback(:t):void):cancelable=
      Subscription := event_subscription{}
      spawn{ RunSubscription(Event, Callback, Subscription) }
      Subscription

  # The task sleeps until the event or Stop fires; no per-frame work.
  RunSubscription(Event:event(t), Callback(:t):void, Subscription:event_subscription where t:type)<suspends>:void=
      race:
          loop:
              Payload := Event.Await()
              if(Subscription.Canceled?):
                  break
              Callback(Payload)
          Subscription.AwaitStop()

  # The owner's disposer prefers Stop, falling back to Cancel for ordinary cancelables.
  subscriptions := class(disposable):
      var Items<private> : []cancelable = array{}
      Add<public>(Item:cancelable)<transacts>:void=
          set Items += array{ Item }
      Dispose<override>():void=
          for(Item : Items):
              if(Stoppable := stoppable[Item]):
                  Stoppable.Stop()
              else:
                  Item.Cancel()
          set Items = array{}
  ```
  - **Why `Stop` in addition to `Cancel`:** `cancelable.Cancel` is `<transacts>`. Both `Event.Signal` and the `spawn` macro are no-rollback there (error 3512), and `task(t)` has no `Cancel` method, so `Cancel` alone can only set a flag. The listener would then sit parked until the event fired again, which may be never once the widget is gone. `Dispose()` has no effect specifiers (no-rollback), so it can signal, and `Stop` ends the task immediately. `Cancel` still works for callers that only hold a `cancelable`: the callback never runs after it.
  - **Don't "fix" this by racing the loop against a `Sleep(0.0)` flag check.** It cleans up within a frame, but it wakes every active subscription every frame for as long as the widget exists.
  - Name the helper so it doesn't collide with any existing `Subscribe` in scope; Verse forbids shadowing.
- **Nothing a player's UI spawns may outlive the player.** Live servers see constant join/leave churn, so anything left running per departed player piles up for the whole session. The lifecycle rules:
  - Keep every subscription handle; never discard the `cancelable` a `Subscribe`/`SubscribeEvent` returns.
  - Store handles in the owning widget's disposer.
  - A parent widget's `Dispose` must also dispose its child and slot wrappers.
  - The per-player UI manager calls `Dispose` from its player-removed handler.
  - Build widgets and slot pools once per player and reuse them across open/close (hide by removing from `player_ui`). Rebuilding on every open multiplies the handles you have to manage.
- **Passive display-only elements** (a HUD hint, a static label) need no Verse Fields at all — just instantiate the reflected `WBP_X` class and `set` its exposed vars. Remember that a value set through the archetype (`WBP_X{ Field := V }`) may not push through its MVVM binding on creation; `set` it again after construction if it doesn't show.
- **Toolchain**: create/duplicate/edit the WBP itself with the `UMGToolSet` MCP toolset (list_properties → get_properties → set_properties on any widget/slot it returns); author the Verse Fields and their MVVM bindings with `VerseFieldsToolset`. After either, the new/changed members only show up in Verse once the digest regenerates — rebuild (`VerseToolset.BuildAll`) before trusting a "missing member" error.
- **`UMGToolSet.AddWidget` can't construct Blueprint-generated widget classes** (`UEFN_TextBlock_C`, `UEFN_Button_Regular_C`, any project `WBP_X`) — it errors `Can't construct a widget using the passed class because it is unsupported`, even though `ListWidgetClasses`/`GetWidgetClassInfo` both report the class as valid. It works fine for native engine panel/leaf classes (`/Script/UMG.CanvasPanel`, `/Script/UMG.StackBox`, `/Script/UMG.Image`, `/Script/FNE_UILibrary.ActionWidget`, …). **Workaround**: `AddWidget` a native placeholder (e.g. `/Script/UMG.Image`) into the target slot, then `ReplaceWidgetWithTemplate` to swap it for the Blueprint class — that path succeeds (the `unmatchedProperties`/`unmatchedFunctions` it reports back are expected noise from the type swap, not failures). Then `RenameWidget` it to a sensible name.
- **A `VerseFieldsToolset.AddVerseField` field only reflects into `Assets.digest.verse` once it's actually bound to a widget property.** An added-but-unbound field (checked via `ListVerseFields`, which lists it fine) will NOT appear in the digest no matter how many times you `BuildAll` — it isn't a staleness problem, the generator genuinely skips unused fields. The fix is the full sequence in order: `BindWidgetPropertyToVerseField` for every field → `UMGToolSet.CompileWidgetBlueprint` → `AssetTools.save_assets` → `VerseToolset.BuildAll`. Do all four before concluding a field "isn't reflecting" — checking the digest between compile and the final `BuildAll`, or before binding, gives a false negative. **Avoid `AssetTools.reload_asset`** on a widget blueprint with MVVM bindings — it produced `ReloadPackage failed to find a replacement object for '..._C:MVVMViewClass_N'` in testing (references nulled, though `MVVMToolset.FixupMVVMData` + recompile recovered it).
- **A brand-new interactive click event on a fresh button is currently NOT achievable through these MCP tools.** `VerseFieldsToolset.AddVerseField`'s `fieldType` explicitly rejects `"event"` ("`'event' fields exist but can't be created or retyped through this API`"). `ClickedEvent<public>:event(tuple(...))`-style fields have to be authored by hand in the UMG Blueprint graph (an Event Dispatcher wired to the button's OnClicked). There is no exposed MCP toolset for Blueprint event-graph editing (checked: no toolset covers `EdGraph`/event dispatchers, and `ValkyriePythonToolset` only exposes an enable/is-enabled check, not script execution). Practical options when a new WBP needs a button that drives Verse: (a) keep that specific interactive control as Verse-built (`button_regular` from `/Fortnite.com/UI`, `.OnClick().Subscribe(...)` — works today, no WBP needed) while the rest of the screen is UMG: (b) ask the user to add the Event Dispatcher / OnClicked wiring by hand in the WBP designer (a couple of minutes of manual work), then pick up the Verse-side `ClickedEvent.Await()` listener (above) once the field exists in the digest. Don't silently fall back to (a) without surfacing the tradeoff — it's a real capability gap, not a style choice.

## Device custom-UI widgets (device ViewModels)

Some Fortnite devices can swap their built-in UI for a designer-made WBP, and the device keeps running its logic. The **Skilled Interaction** device does this through its `Custom Widget` property. The device pushes live state into a device-owned MVVM **ViewModel**, and the WBP binds to it. Verse is not involved: the device's Verse events (`InteractionSucceededEvent`, and so on) fire as usual. **Before proposing to reimplement a device minigame or HUD in Verse to reskin it, check for a custom-widget option.** The Verse API may expose only a sliver of the state (for example, the scrubber position only inside input events), while the ViewModel exposes all of it every frame. Epic's walkthrough is *Creating Custom Skilled Interactions in UEFN* (dev.epicgames.com/documentation/fortnite/creating-custom-skilled-interactions-in-unreal-editor-for-fortnite).

**Setup:** create a plain `UserWidget` WBP, add the device's ViewModel to it (for Skilled Interaction: `/CRD_SkilledInteractionDevice/UI/UEFN_SkilledInteraction_ViewModel.UEFN_SkilledInteraction_ViewModel_C`, shown in the editor as "Device - Skilled Interaction View Model"), build and bind, then assign the WBP in the device's `Custom Widget` property. The widget is added on screen and bound once per interacting player.

**Skilled Interaction ViewModel properties.** Meter and zone values are normalized 0–1.

| Property | Type | Use |
|---|---|---|
| `CurrentMeterValue` | double | scrubber position, updates continuously |
| `GoodZoneMin` / `GoodZoneMax`, `PerfectZoneMin` / `PerfectZoneMax` | double | zone bounds; they change at runtime if *Position Zone Randomly* is on |
| `SuccessCount` / `SuccessTarget`, `FailureCount` / `FailureTotalAllowed` | int | progress |
| `InteractTimer` (double), `InteractTimerVisibility` (`ESlateVisibility`) | | time-limit readout |
| `MeterColor`, `GoodZoneColor`, `PerfectZoneColor`, `ScrubberColor`, `BackgroundColor`, `TimerColor` | LinearColor | the colors configured on the device, which bind straight to `Image.ColorAndOpacity` |
| `IsMeterLocked` | bool | |
| `Header`, `Description`, `InteractTextLabel` | string | device label text |

### Positioning by layout, not transforms

UEFN's MVVM is heavily restricted. There are **no Slider or ProgressBar widgets** (natives available: Canvas, Overlay, StackBox, SizeBox, ScaleBox, Image, Grid/UniformGrid, WrapBox, WidgetSwitcher, ScrollBox, plus `UEFN_TextBlock` and `UEFN_Button_Regular`). Conversion functions come from a short allowlist: `Add*`/`Multiply*` (Int/Double), `MakeTransform`, `Conv_*ToText`, `Conv_BoolToSlateVisibility`, `Conv_DoubleToBoolInterval`, `InvertBool`, material `Conv_Set*Parameter`, brush makers, and a few others. Binding `Render Transform` through `MakeTransform` needs split struct pins (`Translation.Y`, `Scale.Y`) plus pin defaults, which proved unusable in practice. **Use the documented layout pattern instead:**

- **The value-to-length primitive:** `ScaleBox` (Stretch = **User Specified**) wrapping a `SizeBox` with a fixed **Height Override** H (width override 1 for invisible spacers). Bind the ScaleBox's **`UserSpecifiedScale`** directly to a 0–1 ViewModel double, and the spacer's desired height becomes `value × H`. The ScaleBox scales uniformly, so use it only for invisible spacers and never wrap visible art you don't want shrunk.
- **Moving marker (scrubber/indicator art):** a vertical `StackBox` containing `[filler, slot Size = Fill] [marker, Automatic] [spacer ScaleBox, bound to CurrentMeterValue, H = trackHeight − markerHeight]`. The marker rides up from the bottom as the value goes 0→1. For a horizontal bar, use a horizontal StackBox with a Width Override instead.
- **A zone band between Min and Max without subtraction:** stack full-size layers in an `Overlay`, each a vertical StackBox `[colored Image, Fill] [spacer bound to X, H = trackHeight]`, which paints "color from X up to the top". Later layers cover earlier ones. For good and perfect zones:
  1. good color from `GoodZoneMin`
  2. perfect color from `PerfectZoneMin`
  3. good color from `PerfectZoneMax`
  4. background color from `GoodZoneMax`

  Bottom to top that yields background, good, perfect, good, background. Every binding is direct, with no conversion function. This relies on `GoodZoneMin ≤ PerfectZoneMin ≤ PerfectZoneMax ≤ GoodZoneMax`. The last layer is a flat color, so match it to the backfill art.
- **Epic's alternative** uses a spacer with `1 − Max` via `Add Int Double` (A = `1`, B = `…Max`, **Negate B** = true). It's fine when authoring in the editor, but see the tooling caveat below.
- **Give filler images zero desired height.** An `Image` with no fixed size reports its brush size (32 px by default) as its desired size. In a zone layer `[Image, Fill] [spacer = value × H]` the layer's desired height is then `32 + value × H`, which exceeds the track whenever the value is near 1. The track, the frame and the whole panel then grow and shrink as the zones move (for example with *Position Zone Randomly*), and a marker positioned from the bottom never reaches the top. Wrap every such filler `Image` in a `SizeBox` with Height Override = 0 (the slot stays Fill, so it still stretches), so the layer's desired height is exactly the spacer's.
- **Count-driven icon rows** (successes, failures, lives, charges): an integer ViewModel value can set how many icons show by sizing a clip region, with no per-icon bindings.
  - **Row shape:** an `Overlay` with `Clipping = ClipToBounds` whose width comes from a spacer, `ScaleBox(User Specified)` > `SizeBox(Width = one icon's width, Height = 1)`, bound to the int (spacer width = `count × iconWidth`).
  - **Icons:** next to the spacer, in the same Overlay, put `SizeBox(Width override = 0, Height = iconHeight)` > horizontal `StackBox` of N real icon `Image`s (brush `ImageSize` = the icon size, no tiling). The zero-width SizeBox keeps the icon stack from widening the row, and the clip hides every icon past `count`.
  - **Pre-build the maximum:** create as many icon images as the largest count the device can have, and document that number.
  - **Empty slots:** layer a dim/outline row (width from the target or total) under the lit row (width from the current count), left-aligned.
  - **Remaining vs lost (lives):** draw the full-color row at the total width, then overlay a "lost" row **right-aligned** whose width comes from the lost count. Icons stay grid-aligned because the offset is a whole number of icon widths, so nothing needs subtracting.
- **Panel chrome without textures:** an `Image` with brush `drawAs = RoundedBox` and no resource draws a solid rounded rectangle, so `tintColor.specifiedColor` is the fill and `outlineSettings` holds `cornerRadii` (x/y/z/w), `width` and `color`, with `roundingType = FixedRadius`. Use it for a dark translucent HUD backing panel (around `a = 0.8`) and for circles: a square image with radius = half its size. Put a circle layer under each icon slot (same clip-row trick, padded so the cell pitch is unchanged) and keep the glyph about 60–70% of the circle so it doesn't touch the rim.
- **Sizing a composite HUD:** wrap the whole assembly (for example gauge, icon rows, label) in a `ScaleBox` set to User Specified. Unlike a render transform, it scales the layout size too, so the backing panel hugs the content. Anything that intentionally overflows its parent (a decoration hanging above the gauge) needs matching padding on its slot so it stays inside the panel.
- **Input-prompt row ("[E] Interact"):** `/Script/FNE_UILibrary.ActionWidget` (a native class, so `AddWidget` works) is meant to show the platform-correct key icon. Set `displayType = Icon` and `enhancedInputAction` to an Enhanced Input action asset. **Unresolved: inside a device's custom widget it showed no icon in play.** The designer shows a white square placeholder, and in play the slot stayed empty with `IA_Use` (both the `/Game/Input/Actions` and `FortPlayerController/Use` assets), `IA_UI_Use` and `IA_Interact`, and also with the matching built-in mapping (`InventoryMenuMapping`) added to the player for the whole interaction. Removing every other UI mapping from the player did not help either. The editor log reports `CommonActionWidget:SetEnhancedInputAction was filtered`, so the action can only come from the designer value and there is no script-side setter to try. Don't promise a dynamic key icon here. A fixed glyph image (right only for one input device) is the reliable fallback. Put it in a horizontal StackBox next to a `UEFN_TextBlock` (added through the placeholder-and-`ReplaceWidgetWithTemplate` trick above; its properties are `text`, `font.size`, and `colorAndOpacity` as a SlateColor). Set the design-time text to the wanted default first, then bind a device ViewModel's label string (for example Skilled Interaction's `InteractTextLabel`) straight into the text block's `Text`. A string → text binding compiles with no conversion function, and the default covers the designer and any moment before the device pushes a value.
- **Built-in icon textures:** the engine's icon library (for example `/Game/Athena/UI/InGame/Creative/PropertyEditor/Icon_Picker/Textures/64x64/T_UI_IconLibrary_checkmark_64` and `..._X_64`, which are also the Skilled Interaction defaults for success and failure) can be used directly as an `Image` brush `resourceObject`. They are white glyphs, so `ColorAndOpacity` tints them (green check, red X) and `renderOpacity` dims empty slots. `TextureTools.export_png` and `read_texture` fail on them ("Failed to export texture") even though `get_size` works, so judge them in the designer instead.
- **Don't use a tiled `Image` for repeated icons.** Slate tiles at the **texture's real pixel size** and ignores the brush `ImageSize`. With a large source texture, a short tiled image shows a stretched sliver of one giant tile. `TextureTools.get_size` can report a much smaller size than the source (a 32×32 report for a 1067×1067 texture), so check the exported PNG's actual dimensions.

**Typical assembly (vertical gauge):**
- An `Overlay` of fixed size holds, in draw order: backfill art, a `Track` overlay, and frame art on top.
- The `Track` overlay is padded to the frame's inner area. Inside it go the zone layers, then the marker layer.
- Every spacer's Height Override is derived from the track's inner height, so a new frame only means updating those numbers.

### Tooling caveats (unreal-mcp)

- `MVVMToolset.AddViewModelToWidget` / `ListWidgetViewModels` handle the ViewModel. Read its property list with `ObjectTools.list_properties` on the ViewModel `_C` class. Binding paths use PascalCase (`CurrentMeterValue`), even though `list_properties` prints camelCase.
- `MVVMToolset.CreateViewBinding` works for **direct** bindings (source property → `UserSpecifiedScale` / `ColorAndOpacity`). With a `conversionName` it binds the source to the first type-compatible pin, which can be the wrong one (`MakeTransform` → `Angle`). When types already match it silently drops the conversion (`AddIntDouble` double → double came out as a plain binding). **Pin defaults and split pins can't be set**: `savedPins` on `MVVMBlueprintViewConversionFunction_N` is read-only through `ObjectTools`. So design for direct bindings, or leave conversion-pin setup to the user in the View Bindings panel.
- **Int → double bindings (`SuccessCount` → `UserSpecifiedScale`):** with an empty `conversionName`, the tool auto-picks **`MultiplyIntDouble`**, whose `B` pin defaults to `0.0`, so the result is always 0. Pass **`conversionName: "AddIntDouble"`** explicitly: `A` takes the int and `B = 0.0` passes it through unchanged. Confirm by reading `conversionFunction.functionReference.memberName` and `savedPins` (pin `A` should hold the ViewModel property) on the `MVVMBlueprintViewConversionFunction_N` object that `ListWidgetViewBindings` points to.
- Verify with `MVVMToolset.ListWidgetViewBindings` (`sourceToDestinationConversion: null` means a direct binding) and `UMGToolSet.CompileWidgetBlueprint`.
- **Cooked plugin widgets can't be inspected as widget blueprints:** device-shipped UI (for example `/CRD_*/UI/WBP_*`) only exists as generated classes, so `GetWidgets` and `MVVMToolset` reject the path ("not valid WidgetBlueprint"). `ObjectTools.list_properties` on the `_C` class shows its members but not its widget tree or styling. Match the device's look by eye instead.
- **Reordering children:** `AddWidget` with `childIndex` and `MoveWidget` (with `childIndex`) both insert at a position, but the resulting slot names (`OverlaySlot_N`) are just counters and don't reflect order. Use the slot path the call returns. Renaming a widget with `RenameWidget` updates the view bindings that point at it.
- **Checking a layout without running the game:** `EditorAppToolset.OpenEditorForAsset` on the WBP, then `CaptureEditorImage`, shows the designer with the properties' default values, since bindings don't run there. The designer is fit to the whole screen (about 1/6 scale at 1080p) and its zoom can't be changed through the tools. To inspect small widgets, temporarily set a render-transform scale on the widget, move it into view, and add a light backdrop so dark art is visible. Compile after each change or the preview goes stale, crop and enlarge the screenshot with PIL, and **restore everything before finishing**: remove the backdrop, reset the scale, and put the canvas slot back.

## Gotchas

- **Invalid identifiers are silently skipped.** A parameter/property whose name isn't a valid Verse identifier does not reflect; the digest records it in an embedded comment: `<#> MI_Arch contain properties that were not reflected. Some parameters ... were skipped due to being an invalid identifier: 'NormalMap Strength'`. Spaces and auto-generated GUID-suffixed material slots (`'Mesh-acb755bf-...-material_0'`) are the usual culprits. Fix: rename the parameter/slot in the editor to a plain identifier, rebuild, re-check the digest.
- **A "missing" param usually means a stale digest or a skipped identifier** — check the `<#>` comments before assuming the API doesn't exist, and regenerate digests after any asset change (`toolchain.md`).
- **Reflected classes are code-constructible but editor-authored** — you can't add fields or override members; treat the generated class as a value type you configure.
- **`@editable` on reflected vars cuts both ways**: a reflected component subclass placed via the editor exposes those params in the details panel, so a designer can override what your code assumes.
