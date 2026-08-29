# Scene Graph (entity / component model)

The modern UEFN object model, in `/Verse.org/SceneGraph` (+ `/Verse.org/Simulation`). Replaces the older `creative_device`-centric style with **entities** that hold **components**. Lifecycle methods, signalling, and spawn/branch here are `<no_rollback>` (see `../language/effects-failure-concurrency.md`).

## Entities & prefabs

`entity` (`/Verse.org/SceneGraph`) is the base world object: `class<concrete><unique><transacts><castable>(has_tags)` — hierarchical, identity-comparable, castable, carries **tags**. A class that derives from `entity` is a **prefab** — usually authored in the editor and surfaced in `Assets.digest.verse`, but you can also subclass in code (e.g. marker bases `marker_base := class<castable>(entity){}`).

> Convention: keep logic in **components**, not in the entity subclass. Prefabs should be thin; behavior is composed from components so you can restructure prefabs without refactoring class hierarchies.

Client streaming enums exist (`entity_streaming_policy`: `Spatial`/`NonSpatial`/`Persistent`; `children_streaming_policy`: `Atomic`/`Discrete`) — no entity method consumes them yet in the 42.00 digest (editor-side config).

Entity methods:

```verse
Entity.GetParent[]                         # <reads><decides>:entity
Entity.RemoveFromParent()                  # detach (runs children's OnEnd/OnRemoving)
Parent.AddEntities(array{Child1, Child2})  # attach children (re-parents if needed)
Entity.GetEntities()                       # <reads>:[]entity  (direct children only)
Entity.GetComponent[some_component]        # <reads><decides>:some_component
Entity.GetComponents()                     # <reads>:[]component
Entity.AddComponents(array{Comp1, Comp2})  # add components
Entity.GetGlobalTransform() / SetGlobalTransform(T)   # Get falls back to first parent transform; Set auto-creates a transform_component
Entity.GetLocalTransform()  / SetLocalTransform(T)
Entity.GetOrigin[] / SetOrigin(O) / ResetOrigin()      # alternate origin (origin interface; entity_origin{Entity := E} re-parents the frame)
Entity.GetSimulationEntity[]               # root entity of the experience (<transacts><decides>)
Entity.SetPresentableToPlayers(?[]player)  # [3800+] per-player visibility; false = everyone, array{} = no one
Entity.GetPresentableToPlayers()           # [3800+] <transacts>:?[]player
# tags:
Entity.AddTag(SomeTag) -> tag_key          Entity.RemoveTag[Key]   Entity.RemoveAllTags[tag_type]
Entity.RemoveAllTagsExcept[tag_type or tag_types]
Entity.ContainsTag[tag_type]   Entity.ContainsAnyTag[tag_types]   Entity.ContainsAllTags[tag_types]
```

`GetComponent` is `<decides>` (fails if absent) — pair with the heap effect of its caller and use it in a failure context: `if (M := Entity.GetComponent[mesh_component]):`. During the AddedToScene/BeginSimulation phases, `GetComponent`/`AddComponents` guarantee the returned/added component has reached the same phase.

## Component

```verse
my_component<public> := class<final_super>(component):
    @editable var Speed:float = 100.0

    OnBeginSimulation<override>():void =
        (super:)OnBeginSimulation()
        # subscribe to events, set up state
```

- Subclass with **`<final_super>`** (required to be addable to an entity — guarantees it derives directly from `component`). Further subclassing of *your* component is fine without `<final_super>`.
- **One component per subclass-group per entity.** Only one `light_component` (any subtype) on an entity; use multiple entities for multiple lights.
- `component` is `<abstract><unique><castable>`. From inside, `Entity` is the parent entity (always present after construction). **Constructing a component in code requires supplying it** (`my_comp{ Entity := E, ... }` then `E.AddComponents(array{...})` on that same entity) — components cannot move between entities, and a removed component can only be re-added to its original entity.

### Lifecycle (override; always call `(super:)`)

```
OnAddedToScene            # in the scene; component queries are valid now
  OnBeginSimulation       # set up TickEvents/subscriptions that must complete synchronously
    OnSimulate <suspends> # async update logic (loops); cancelled before OnEndSimulation
  OnEndSimulation         # cancel cached cancelables here
OnRemovingFromScene       # final teardown
```

All are `<no_rollback>` and `<native_callable>`. `OnSimulate` is where the component's behavior over time lives: await events and let structured concurrency (`race`/`sync`/sequencing) play the gameplay out — **not** a polling tick loop (see `../patterns.md` §1). ALL base `component` lifecycle bodies are behaviorally empty — `OnSimulate` is `= {}` in Verse; the native ones (`OnAddedToScene`/`OnBeginSimulation`/`OnEndSimulation`/`OnRemovingFromScene`) each contain only a "did you call super" tracking flag, checked in `DO_ENSURE` builds as a warning ("Super method chain is broken"), compiled to nothing otherwise. The framework's real work (task spawn/cancel, state transitions) wraps overrides from outside via the Notify* path. So super calls are skippable when deriving from `component` directly (at worst a dev-build warning for the native four; nothing for OnSimulate) — but call super when deriving from any intermediate class with its own lifecycle logic, where it is load-bearing. `RemoveFromEntity()` removes a component (flows `OnEndSimulation`→`OnRemovingFromScene`). Predicates `IsInScene[]`, `IsSimulating[]` (`<reads><decides>`).

Typical subscription lifecycle:

```verse
var<private> Subs:[]cancelable = array{}
OnBeginSimulation<override>():void =
    (super:)OnBeginSimulation()
    set Subs += array{ SomeEvent.Subscribe(OnSomething) }
OnEndSimulation<override>():void =
    (super:)OnEndSimulation()
    for (C : Subs) { C.Cancel() }
    set Subs = array{}
```

## Hierarchy queries — return **generators**

On any `entity` (and there are component/tag variants):

```verse
Entity.FindDescendantEntities(entity_type)            # generator(entity_type)
Entity.FindDescendantComponents(component_type)        # generator(component_type)
Entity.FindDescendantEntitiesWithComponent(comp_type)  # generator(entity)
Entity.FindDescendantEntitiesWithTag(tag_type)         # generator(entity)
Entity.FindAncestorEntities(entity_type)               # generator(entity_type)
Entity.FindAncestorComponents(component_type)          # generator(component_type)
Entity.FindAncestorEntitiesWithComponent(comp_type)    # generator(entity)
Entity.FindAncestorEntitiesWithTag(tag_type)           # generator(entity)
```

`Find*` **include** the starting entity (exception: `FindDescendantEntitiesWithTag` from the simulation entity excludes the simulation entity itself), order is unspecified, and they return **`generator(t)`** (lazy, single-iteration). To take one: `first{ X : Gen }`. To materialize: `for (X : Gen) { X }`.

```verse
# tag-based "find the manager" idiom
(InEntity:entity).GetMyManager()<transacts><decides>:my_manager =
    Sim := InEntity.GetSimulationEntity[]
    MgrEntity := first{ E : Sim.FindDescendantEntitiesWithTag(my_manager_tag) }
    MgrEntity.GetComponent[my_manager]
```

> **Convenience helpers are usually utils-file extensions, not engine APIs.** Calls like `entity.GetFirstDescendantComponent[type]` and `entity.GetDescendantComponentsWithInterface(some_iface)` are thin extension methods written on top of the engine's `Find*` generators. Typical definitions:
> ```verse
> (E:entity).GetFirstDescendantComponent(ct:castable_subtype(component))<transacts><decides>:ct =
>     first{ C : E.FindDescendantComponents(ct) }
> p_interface := interface{}   # empty marker base so interfaces can be a castable_subtype
> (E:entity).GetDescendantComponentsWithInterface(iface:castable_subtype(p_interface))<transacts>:[]iface =
>     for (C : E.FindDescendantComponents(component), IF := iface[C]) { IF }
> ```
> See `../patterns.md` for the full set of these idiomatic helpers.

## Scene events

A lightweight message bus distinct from `listenable` events:

```verse
my_event<public> := class(scene_event):  Payload:int      # scene_event is an interface
# send (each returns logic — true if any participant consumed it):
Entity.SendDown(my_event{Payload := 5})  # this entity's components, then each child (recursive)
Entity.SendUp(my_event{...})             # this entity's components, then up to parent
Component.SendDown(...)                   # this single component only (invokes its OnReceive)
# receive (override on a component; [4000+]):
OnReceive<override>(SceneEvent:scene_event):logic =
    if (E := my_event[SceneEvent]) { DoThing(E); true } else { false }
```

`OnReceive` returns `logic` — return `true` to **consume** and halt propagation. (Itemization uses scene events like `find_inventory_event`/`add_item_query_event` to choose/veto inventories.)

## Per-frame ticks

```verse
var<private> TickEvents<protected>:tick_events   # on every component
# TickEvents.PrePhysics / .PostPhysics are execution_listenable (payload = DeltaTime:float)
Cancelable := TickEvents.PrePhysics.Subscribe(OnPrePhysics)   # <transacts>:cancelable; OnPrePhysics(DeltaTime:float):void
DT := TickEvents.PostPhysics.Await()                          # <suspends>:float — await a single tick
```

Use ticks only for frame-coupled work that must run before/after physics. Everything else should be event-reactive in `OnSimulate` (await, don't poll); reserve `Sleep`-cadence loops for effects that are genuinely periodic.

## Transforms & the TWO SpatialMath namespaces

There are **two** `vector3`/`rotation`/`transform` types. Mixing them is a type error.

| Namespace | `vector3` axes | Handedness | Used by |
|---|---|---|---|
| `/Verse.org/SpatialMath` (current) | **Forward / Left / Up** | right-handed | Scene Graph `entity` transforms, `creative_prop` physics, most new APIs |
| `/UnrealEngine.com/Temporary/SpatialMath` (deprecated) | **X / Y / Z** | left-handed | `fort_character.GetTransform()`/`GetViewRotation()`/`GetViewLocation()`, many older device APIs, `vector2`/`vector2i` |

Convert with the bridge functions in `/UnrealEngine.com/Temporary/SpatialMath`:
`FromTransform`, `FromRotation`, `FromVector3` (translation), `FromScalarVector3` (scale/magnitude), and the reverse overloads.

```verse
# fort_character.GetTransform() is Temporary (X/Y/Z) → convert to Verse.org (LUF)
CharXform := FromTransform(Char.GetTransform())
Forward   := CharXform.Rotation.GetForwardAxis()        # vector3 Forward/Left/Up
SpawnPos  := CharXform.Translation + Forward * 200.0
New.SetGlobalTransform(transform{ Translation := SpawnPos, Rotation := IdentityRotation(),
                                  Scale := vector3{Forward := 1.0, Left := 1.0, Up := 1.0} })
```

`/Verse.org/SpatialMath` essentials: `vector3{Forward:=, Left:=, Up:=}`, `rotation` (`MakeRotationFromYawPitchRollDegrees`, `IdentityRotation`, `Slerp`, `MakeShortestRotationBetween`, `GetForward/Left/UpAxis`, `Invert`), `transform{Translation:=, Rotation:=, Scale:=}`, `DotProduct`, `CrossProduct`, `Distance`, `(V).Length()`, `(V).MakeUnitVector()`, `Lerp`, `v * rotation`, `v * transform`. (To pin the namespace when both are imported, qualify: `(LUF:)vector3{...}` via a module alias, or `(/Verse.org/SpatialMath:)vector3{...}`.)

## Collision / spatial queries

On `entity`:

```verse
Entity.FindOverlapHits()                                   # generator(overlap_hit)
Entity.FindOverlapHits(GlobalTransform)                    # as if placed at GlobalTransform
Entity.FindOverlapHits(GlobalTransform, Volume)            # custom pose + volume (entity = scene context only)
Entity.FindSweepHits(Displacement)                         # generator(sweep_hit)
Entity.FindSweepHits(Displacement, StartGlobalTransform)
Entity.FindSweepHits(Displacement, StartGlobalTransform, Volume)
```

- **Sweep semantics:** returns the first `Block` hit plus every `Overlap` hit encountered *before* it, sorted by distance — **the blocking hit is last**.
- `collision_volume` shapes: `collision_sphere{Radius}`/`collision_capsule{Radius, Length}` (Z-aligned)/`collision_box{Extents}`/`collision_point` (each a `collision_element` with a `CollisionProfile`; base `collision_volume` has `var Collidable`/`var Queryable` and `Get/SetLocalTransform`).
- `collision_profile` = a `Channel:collision_channel` (`CollisionChannels.stationary/dynamic/avatar/visibility/camera/physics` — closed set) + `GetChannelInteraction:collision_channel_to_interaction` (a `<computes>` function mapping channel→`collision_interaction` `Ignore`/`Overlap`/`Block`). Build custom ones with `MakeCollisionProfile(Channel, Fn)`; a pair's effective interaction is the **Min** of the two directions. Presets in `CollisionProfiles`: `Stationary/Dynamic` × `IgnoreAll/OverlapAll/BlockAll`, `StationaryBlockVisible`, `VisibilityOverlapAll`.
- `sweep_hit` fields: `ContactPosition`, `ContactNormal`, `ContactFaceNormal` (most-opposing face normal at edges/vertices), `SourceHitDistance`, `SourceHitTranslation`, `TargetComponent` (→ `.Entity`)/`TargetVolume`, `SourceComponent:?component`/`SourceVolume`/`SourceStartGlobalTransform`. `overlap_hit`: `SourceComponent`/`SourceVolume`/`SourceGlobalTransform`/`TargetComponent`/`TargetVolume`.

```verse
Probe := collision_sphere{ Radius := 5.0, CollisionProfile := VisibilityOverlapAll }
Hits  := Sim.FindSweepHits(Delta, StartTransform, Probe)
if (Hit := first{ H : Hits }):  Place(Hit.ContactPosition, Hit.ContactNormal)
```

`mesh_component` exposes overlap **events**: `EntityEnteredEvent`/`EntityExitedEvent : listenable(entity)` (needs `Queryable := true`).

## Built-in components (catalog)

All are `class<final_super>(component, ...)` you add to entities.

- **`transform_component`** — `var GlobalTransform`, `var LocalTransform`, `var Origin:?origin` (alternate reference frame; `entity_origin{Entity := E}` makes transforms relative to another entity — see also `entity.Set/Get/ResetOrigin`). (Or use the `entity.Get/SetGlobalTransform` extension methods, which auto-create one.)
- **`mesh_component`** — render a mesh; `var Collidable/Queryable/Visible:logic`, `EntityEntered/ExitedEvent`, `enableable` (Enable/Disable = rendering). Subclasses in `/UnrealEngine.com/BasicShapes`: `cube`, `sphere`, `plane`, `cone`, `cylinder`.
- **`light_component`** family — base has `var CastShadows`, `var ColorFilter:color`, `var SpecularScale`/`DiffuseScale`, `enableable`. `directional_light_component{Illuminance}` (Lux, parallel sun shadows, `SourceAngleDegrees`); the local lights `sphere_/spot_/rect_/capsule_light_component` use `var Intensity` (Candela) + `var AttenuationRadius:?float` + shape dims (`SourceRadius`/`SourceLength`/`SourceWidth`+`SourceHeight`+barn doors/`Inner/OuterConeAngleDegrees`).
- **`camera_component`** family `[4110+ experimental]` — `perspective_/orthographic_/physical_camera_component`, `camera_director_component`, modifier stacks. Full contract (incl. Fortnite camera/control-rig components): `cameras.md`.
- **`sound_component`** / **`particle_system_component`** — `Play()`/`Stop()`, `var AutoPlay`, `var Enabled`, `enableable`.
- **Skeletal animation** `[4100+ experimental]` — `(entity).PlaySkeletalAnimation(Animation:skeletal_animation, ?EaseInWindow, ?EaseOutWindow:easing_window)<transacts><decides>:play_skeletal_animation_result` (an `Easeable:easeable` handle, which is a `cancelable`).
- **`keyframed_movement_component`** (`KeyframedMovement`) — `SetKeyframes([]keyframed_movement_delta, playback_mode)`, `Play`/`Pause`/`Stop()`/`Stop(BlendOutTime)`; events `PlayedEvent`/`PausedEvent`/`StoppedEvent`/`FinishedEvent` (finite animations only)/`KeyframeReachedEvent:listenable(tuple(int, logic))` (index, IsReversed); predicates `IsPlaying[]`/`IsPaused[]`/`HasValidAnimation[]`; `Duration:?float` (false when looping). Plays back in the **Pre-Physics** tick phase; with a parent constraint, animation is relative to the parent. Great for projectiles/movers. `keyframed_movement_delta{Transform:=, Duration:=, Easing:=}` with easing functions (`linear_/ease_*`); playback `oneshot_/loop_/pingpong_*`. **The ONLY client-smooth Verse mover** (no client `SetTransform` prediction exists). Semantics: deltas are ADDITIVE — `SetKeyframes` prepends the entity's current **local** transform as post 0 and folds deltas into absolute posts (Translation/Scale add — `Scale := vector3{}` means "no change"; Rotation composes `Prev * Delta`, i.e. the delta lives in the **previous post's local frame** — push a world arc `W` as `Cur * W * Cur.Invert()`). All posts are parent-relative: unrotate world translation deltas by the parent's rotation. `Play()` replicates ONE command; clients evaluate the whole curve locally against server time (slerp between posts) — motion is framerate-smooth *while one command plays*, but every re-command has a latency window where the old OneShot clamps at its end (freeze → skip-ahead jump = chop). **Continuously-steered motion (boids, homing, follow-cam): push MULTI-keyframe commands covering the next N substeps plus a predicted TAIL, re-command exactly where the tail begins** — the tail glides through the replacement window and both curves agree on the trajectory, so seams vanish. Start each command's first delta from the entity's *actual* pose (self-correcting). A finished OneShot clamps and freezes; `Stop()` snaps, `Stop(BlendOutTime)` decelerates.
- **`interactable_component`** / **`basic_interactable_component`** — player interaction prompts. `StartedEvent`/`SucceededEvent`/`CanceledEvent : listenable(agent)`; override `CanInteract` (`<decides><reads>`)/`OnStarted`/`InteractMessage`; drive programmatically with `Start[Agent]`, and on the basic variant `Succeed[Agent]`/`Cancel[Agent]`, `InteractingAgents:[]agent`, `GetRemainingCooldownDurationAffectingAgent(Agent)`. Config objects (`?`-fields on the basic variant): `interactable_cooldown{Duration, RemainingDuration, ExpiredEvent}` / `interactable_cooldown_per_agent` / `interactable_duration{InteractDuration, MaxSimultaneousInteractors:?int, Get/SetRemainingInteractDurationForAgent}` / `interactable_success_limit{MaxSuccessfulInteractions:?int, SuccessfulInteractionCount, ClearSuccessfulInteractionCount()}`. `[4020+]` `var CanInteractMessage`/`CannotInteractMessage`. Foundation for pickups, craft buttons, slots.
- **`stackable_component`** / **`basic_stackable_component`** `[4000+]` — `var StackSize`/`MaxStackSize:?int`, `SetStackSize`, `SetMaxStackSize(?ClampStackSize)`, `Split[Amount]:entity`, `CanMergeInto[Target]`, `MergeInto[Target, ?TargetAmount]` (check mergeability **both directions**; self-merge always fails), `ChangeStackSizeEvent`/`ChangeMaxStackSizeEvent`. The basic variant adds `split_prefab_type:castable_concrete_subtype(entity)`. Pairs with `has_merge_rules` (`AllowMergeInto[]`/`OnMergeInto`).
- **`possessable_component`** (`Agent:?agent`; query with `(agent).GetPossessedEntities()` / `(agent).IsEntityPossessed[Comp]`), **`icon_component`** (`var Icon:texture`), **`description_component`** (`/Verse.org/Presentation` — implements `has_description`: `var Name`/`Description`/`ShortDescription:message`), **`rarity_component`** `[4000+]` (`var Rarity:rarity`; `rarity` subclasses `common_…legendary_rarity` each carry a `Color`).
- **Itemization** (`/UnrealEngine.com/Itemization`): **`inventory_component`** (`AddItem`/`AddItemDistribute`/`RemoveItem` → `result`, `GetItems`/`FindItems`/`GetInventories`, `AddItemEvent`/`RemoveItemEvent`/`EquipItemEvent`/`UnequipItemEvent`), **`item_component`** (`GetParentInventory[]`, `IsEquipped[]`, `Equip`/`Unequip` → `result`, `Drop`/`PickUp`, `ChangeInventoryEvent`/`ChangeEquippedEvent`, `Categories`). Fortnite specializations in `/Fortnite.com/Itemization` (`fort_inventory_component`, weapon/build/resource/ammo hotbars). See `itemization.md`.

`enableable` interface (`Enable()`/`Disable()`/`IsEnabled[]`) is implemented by light/mesh/sound/particle/interactable components and many devices.

## Agents, players, session

`/Verse.org/Simulation`: `agent` is `class(entity)`; `player` is `class(agent)` (and a `weak_map` key); `session` (per-round, `weak_map` key, `GetSession()`); `team`. `agent`/`player` carry the gameplay extension methods (`GetFortCharacter[]`, …) — see `characters-combat.md`. Note: you currently **cannot attach entities/components to an `agent`** — hanging them off a player uses the state-entity proxy workaround; plain per-player data has several normal homes (`../patterns.md` §4).
