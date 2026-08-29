# Cameras (Scene Graph)

Two layers, both **`@experimental`**: the **camera component family** in `/Verse.org/SceneGraph` `[4110+]` (which camera renders, projection, blending) and the **Fortnite camera modifiers** in `/Fortnite.com/SceneGraphCameras` `[4200+]` (fixed-angle / fixed-point / orbit follow behaviors). The older device-based cameras (`gameplay_camera_fixed_point/first_person/fixed_angle/orbit_device`, per-agent camera stacks via `AddTo(Agent)`/`RemoveFrom(Agent)`) are in `devices.md` — that family is the non-Scene-Graph path.

> **Digest = source of truth.** Exact signatures live in your build's generated `Verse.digest.verse` / `Fortnite.digest.verse`; `grep` them. See `../uefn/toolchain.md`. Everything below is version-gated (`@available{MinUploadedAtFNVersion := 4110/4200}`) and `@experimental` — expect churn.

## Camera components — `/Verse.org/SceneGraph` `[4110+]`

All are `class<final_super>(camera_component)`; add to an entity, position via the entity's transform.

- **`camera_component`** (`<abstract>`, base; implements `has_camera_modifier`): `var NearClippingPlaneDistance:float` (`<= 0.0` = default), `var FarClippingPlaneDistance:float` (`Inf` = default). Both `@editable`.
- **`perspective_camera_component`** — `var FieldOfViewDegrees:float`: *horizontal* FOV specified at 16:9; scaled Hor+ for other aspect ratios (ultrawide sees more, never less).
- **`orthographic_camera_component`** — `var OrthographicProjectionWidth:float` (centimeters).
- **`physical_camera_component`** — `var Body:camera_body`, `var Lens:camera_lens` (cinematographic exposure/DoF).
- **`camera_director_component`** — selects the active camera and blends on change; also the home for shared post-process and shakes. `AddCamera(Camera:camera_component, Priority:rational)<transacts><decides>:cancelable` — **highest priority wins**; hold the `cancelable` and `.Cancel()` to remove the camera again (there is no `RemoveCamera`).

`camera_body` / `camera_lens` are immutable `<computes>` value classes: body = `SensorWidth/HeightMillimeters`, `SensorHorizontal/VerticalOffsetMillimeters`, `ISO`, `ShutterSpeed`; lens = `FocalLengthMillimeters`, `FStop`, `SqueezeFactor`, `BladeCount`.

## The modifier pipeline — `camera_state` in, `camera_state` out

Camera rendering is a **`modifier_stack(camera_state)`** (see `../language/verse-org.md` → `modifier`/`modifier_stack`): every `camera_component` (via `has_camera_modifier`) exposes `CameraModifiers:modifier_stack(camera_state)` (`@editable`), evaluated in position order.

- **`camera_modifier`** (`class(modifier(camera_state))`) — base for anything that transforms camera state: `Evaluate(InValue:camera_state)<reads>:camera_state`. Subclass it for custom behaviors; the Fortnite modifiers below are prebuilt ones.
- **`camera_modifier_stack`** — the stack itself; `AddModifier(m, Position:rational)→cancelable`. Module-level anchor positions: `GlobalModifierPosition:rational`, `VisualModifierPosition:rational`.
- **`camera_state`** (`<computes><epic_internal>` — you **cannot construct one**, only read/derive in `Evaluate`): `RenderLocation:vector3` / `RenderRotation:rotation` (render pose only — moving these, e.g. a shake, does *not* move any entity), `ProjectionMode:camera_projection_mode` (`Perspective`/`Orthographic`, open enum), `OrthographicProjectionWidth`, `Body`, `Lens`, `PhysicalCameraWeight` (0.0–1.0), `Near/FarClippingPlaneDistance`. All spatial types are `/Verse.org/SpatialMath` (Forward/Left/Up).

## Transitions & blends

- **`camera_transition`** (`<concrete>`): `Blend:camera_mode_blend` + `Duration:float`, both `@editable`.
- **`camera_mode_blend`** subclasses: `_pop` (instant snap), `_linear`, `_smoothstep`, `_smootherstep`, `_orbit` (wraps a `DrivingBlend`).
- **`camera_transition_initial_orientation`** enum: `PreviousYawPitch` / `PreviousAbsoluteTarget` / `PreviousRelativeTarget`.

## Fortnite camera modifiers — `/Fortnite.com/SceneGraphCameras` `[4200+]`

Prebuilt `class<concrete>(camera_modifier)`s — construct with a `TargetEntity` (immutable field) and add to a camera's `CameraModifiers` stack. All distances **centimeters**; all vectors/rotations `/Verse.org/SpatialMath`.

- **`fort_fixed_angle_camera_modifier`** — locked-angle follow cam. `TargetEntity:entity`; `var CameraDistanceCentimeters`, `var PositionOffsetCentimeters:vector3`, `var RotationOffset:rotation` (default pitch `-45.0` — looks down 45°), per-axis `var Forward/Lateral/VerticalDampingFactor`, collision avoidance (`var EnableCollision:logic`, `var CollisionSphereRadiusCentimeters`, `var CollisionSafePositionOffsetCentimeters`).
- **`fort_orbit_camera_modifier`** — follow + manual rotate. `TargetEntity:entity`; boom arm (`var BoomArmOffsetCentimeters:vector3`, `var BoomArmMaxForward/BackwardInterpolationFactor`), `var CameraDistanceCentimeters`, `var CameraHeightCentimeters` (orbit-pivot height above the entity root), `var PositionOffsetCentimeters` (applied after the boom arm), `var RotationOffset`, per-axis damping, `EnableCollision`/`CollisionSphereRadiusCentimeters`.
- **`fort_fixed_point_camera_modifier`** — stationary camera that *reframes* by rotating. `TargetEntities:[]entity` (frames a group); `var RotationOffset`. Screen-space framing model (all coords normalized, `0.5` = center):
  - Ideal position: `var IdealFramingLocationScreenSpaceLeft/Up`.
  - **Dead zone** (`var DeadZoneScreenSpaceLeft/Right/Top/Bottom` — margins from each screen edge): target moves freely, no reframing; `(0,0,0,0)` = always reframe.
  - **Soft zone** (`var SoftZoneScreenSpaceLeft/Right/Top/Bottom`): inside = normal damping, outside = stronger catch-up.
  - Damping: `var ReframeDampingFactor` (0 = instant snap; 1–10 typical), `var LowReframeDampingFactor:?float` (interpolates damping between ideal point and soft-zone edge when set), `var ReengageTimeSeconds` (ramp-up after leaving the dead zone; `<= 0.0` = stop reframing immediately on entry).
  - `var CanYaw`/`var CanPitch:logic` (tilt-only / pan-only cams), `var SnapToIdealFramingOnEnable:logic` (`false` for pop-free entry from a wider blend), `var TargetMovementAnticipationTimeSeconds` (predictive lead — frames where the target *will* be; `0` = none).

## Gotchas

- **Two version gates:** components/blends are `4110+`, the Fortnite modifiers are `4200+` — a project uploaded at 41.x can use `perspective_camera_component` but not `fort_orbit_camera_modifier`.
- `camera_state` is `<epic_internal>` — a custom `camera_modifier.Evaluate` must derive its output from `InValue`, not build a fresh state.
- `AddCamera` is `<decides>` — call with `[]` in a failure context; removal is via the returned `cancelable` only.
- Modifier `TargetEntity`/`TargetEntities` are **non-`var`** — retargeting means constructing a new modifier, not mutating.
- To make a camera the player's view you still need a director (or the device-based camera stack in `devices.md`); a bare `perspective_camera_component` on an entity renders nothing by itself.
