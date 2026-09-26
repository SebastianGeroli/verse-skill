# Creative Devices & Props (`/Fortnite.com/Devices`)

The classic (pre-Scene-Graph) device surface, still common as a bootstrapping root and for the ~190 built-in devices. Verified against build 42.00.

> **Digest = source of truth.** Exact signatures live in your build's generated `Fortnite.digest.verse`; `grep` it. See `../uefn/toolchain.md`.

## Class hierarchy & shared surface

```
creative_object_interface (interface, positional)
└─ creative_object                    # shared transform/move API
   ├─ creative_device_base <abstract> # base of ALL built-in devices (epic_internal: can't subclass)
   └─ creative_prop <final>           # placed/spawned props
creative_device (creative_object_interface)  # <concrete> — the ONE class YOU subclass
```

- **`creative_device`** is the only public-subclassable entry point: override **`OnBegin()<suspends>`** / **`OnEnd()`** (coroutines spawned inside `OnEnd` may never run). Derived classes appear in the UEFN content browser after compile; drag into the island to place. A Scene-Graph project mostly uses components instead, but a `creative_device` is still a common bootstrapping root.
- Shared `creative_object`/`creative_device` methods: `GetTransform()<transacts>` (⚠ Temporary X/Y/Z SpatialMath — check `IsValid[]` first on disposables or you get a runtime error), `TeleportTo[Position, Rotation]` / `TeleportTo[Transform]` `<transacts><decides>`, `MoveTo(…, OverTime:float)<suspends>:move_to_result` (`DestinationReached | WillNotReachDestination`; interrupts any playing prop animation), `Show()`/`Hide()`. `[4000+]` adds `GetGlobalTransform()`/`SetGlobalTransform()` and `TeleportTo[]`/`MoveTo()` overloads taking **`/Verse.org/SpatialMath`** transforms — prefer these.
- Extension methods on any device/object: `.GetTags()<transacts>:tag_view`, `.GetPlayspace()<transacts>:fort_playspace`, `.GetSimulationEntity()<transacts><decides>:entity` (bridge into Scene Graph), and `.FindCreativeObjectsWithTag(tag_type:castable_subtype(tag))<transacts>:generator(creative_object_interface)` (also on `npc_behavior` and `entity`; the `Tag:tag` instance overloads and `GetCreativeObjectsWithTag(s)` are **deprecated**).

## `creative_prop` & `SpawnProp`

- **`creative_prop`** (`creative_object`, `invalidatable`): `Dispose()`, `IsDisposed[]`/`IsValid[]` `<transacts><decides>`, `SetMesh(mesh)`, `SetMaterial(material, ?Index:int)`, `Show()`/`Hide()` (also toggles collision), `var CanBeDamaged:logic`.
- Physics (all no-ops **if physics is disabled** / no FortPhysicsComponent; vectors are `/Verse.org/SpatialMath`): `Get/SetLinearVelocity` (m/s), `Get/SetAngularVelocity` (rad/s), `ApplyLinearImpulse`/`ApplyAngularImpulse`, `ApplyForce` (N)/`ApplyTorque` (N·m), `GetMass()` (kg), `Get/SetDynamic(logic)`.
- **`SpawnProp(Asset:creative_prop_asset, Position, Rotation | Transform)<transacts>:tuple(?creative_prop, spawn_prop_result)`** (Temporary SpatialMath, cm). `spawn_prop_result`: `Ok | UnknownError | InvalidSpawnPoint | SpawnPointOutOfBounds | InvalidAsset | TooManyProps` — **limits: 100 props per script device, 200 total per island**. `DefaultCreativePropAsset` is the placeholder for `@editable` prop-asset fields.

## Prop animation — `CreativeAnimation` submodule

`(Prop:creative_prop).GetAnimationController()<transacts><decides>:animation_controller` — fails for non-animatable props (building-attached walls, chests, llamas, props with *Register with Structural Grid* set).

- **`animation_controller`**: `SetAnimation(Keyframes:[]keyframe_delta, ?Mode:animation_mode)`, `Play()`/`Pause()`/`Stop()` (Stop resets to first keyframe AND the prop transform), `GetState()<transacts>:animation_controller_state` (`InvalidObject | AnimationNotSet | Stopped | Playing | Paused`), `IsValid[]`, `AwaitNextKeyframe()<suspends>:await_next_keyframe_result` (`KeyframeReached | NotPlaying | AnimationAborted`); events `KeyframeReachedEvent:listenable(tuple(int, logic))` (index, in-reverse — index is 1-based over deltas; a PingPong reverse-final is index 0), `MovementCompleteEvent` (OneShot only), `StateChangedEvent`.
- **`keyframe_delta`** = `{DeltaLocation:vector3, DeltaRotation:rotation, DeltaScale:vector3 (multiplicative), Time:float, Interpolation:cubic_bezier_parameters}` — Temporary (X/Y/Z) SpatialMath, cm. **Deltas, not positions**: keyframe 0 is the prop's current pose; each delta is a *world-space* transformation concatenated onto the previous. `animation_mode`: `OneShot | PingPong | Loop` (**Loop requires net translation AND rotation of zero**).
- Interpolation: `InterpolationTypes.Linear/Ease/EaseIn/EaseOut/EaseInOut` (CSS-easing semantics) or custom `cubic_bezier_parameters{X0,Y0,X1,Y1}` (`0.0 <= X0,X1 <= 1.0` or `SetAnimation` errors).
- This is the `creative_prop` mover. The Scene-Graph equivalent (and the only client-smooth option for entities) is `keyframed_movement_component` — see `scene-graph.md`.

## The device catalog (~190 concrete devices)

All are `class<concrete><final>(creative_device_base)` unless noted; the abstract bases (`trigger_base_device`, `effect_volume_device`, `powerup_device`, `base_item_spawner_device`, `storm_controller_device`, `vehicle_spawner_device`, `gameplay_camera_device`, `gameplay_controls_device`, `physics_object_base_device`, `prop_spawner_base_device`) are `<epic_internal>` — usable as variable types, not subclassable. Uniform shape: `Enable()`/`Disable()`, `listenable(...)` events (payload `agent`, `?agent`, or `tuple()`), setters take `message` for player-facing text. Names only below — `grep` the digest for the API.

- **Game flow, rounds & settings**: `end_game`, `round_settings`, `experience_settings`, `team_settings_and_inventory`, `class_designer`, `class_and_team_selector`, `class_selector_ui`, `matchmaking_portal`, `down_but_not_out`, `player_spawner`, `player_checkpoint`, `reboot_van` (+ `reboot_card_purchase_options`), `timer`, `real_time_clock`, `support_a_creator`, `item_shop`.
- **Storm**: `basic_storm_controller`, `advanced_storm_controller`, `advanced_storm_beacon`.
- **Triggers, switches & logic**: `trigger`, `pulse_trigger`, `perception_trigger`, `attribute_evaluator`, `input_trigger`, `button`, `conditional_button`, `switch`, `lock`, `channel`, `signal_remote_manager`, `rng`, `vote_group`/`vote_option`.
- **Score, stats & objectives**: `score_manager`, `tracker`, `stat_creator`, `player_counter`, `accolades`, `analytics`, `objective` (healthful/damageable/healable), `timed_objective`, `capture_area`, `capture_item_spawner`, `collectible_object`, `race_manager`/`race_checkpoint`, `elimination_feed`, `elimination_manager`.
- **Items & economy**: `item_granter`, `item_spawner`, `item_placer`, `item_remover`, `vending_machine`, `sword_in_the_stone`, `hero_chest`, `bank_vault`, `hive_stash`, `supply_drop_spawner`, `nitro_barrel_spawner`, `fuel_pump`, `carryable_spawner` (+ `carryable_spawner_agent_impact_result` payload struct).
- **NPCs, AI & wildlife**: `guard_spawner`, `npc_spawner`, `character` (`<concrete>`, not final; `PlayEmote()`), `creature_manager`/`creature_placer`/`creature_spawner`, `wildlife_spawner`, `ai_patrol_path`, `sentry`, `automated_turret` (healthful/healable), `roly_poly_spawner` (+ `roly_poly` class), `conversation`, `dance_mannequin`, `firefly_spawner`, `healing_cactus`, `wilds_plant`, `earth_sprite`, `crowd_volume`, `shooting_range_target`/`shooting_range_target_track`. (AI payload struct: `device_ai_interaction_result`. NPC *behaviors* live in `/Fortnite.com/AI` — `characters-combat.md`.)
- **Prop hunt / stealth**: `prop_o_matic_manager`, `disguise`, `hiding_prop`, `changing_booth`.
- **Movement & traversal**: `teleporter`, `bouncer`, `crash_pad`, `air_vent`, `grind_rail`, `vine_rail`, `movement_modulator`, `player_movement_settings`, `boost_pad_rocketracing`, `nitro_hoop`, `trick_tile`, `color_changing_tiles`, `pinball_bumper`/`pinball_flipper`, `chair`, `skilled_interaction` (its UI can be replaced with a custom WBP bound to the device's ViewModel; see `../uefn/asset-reflection.md` → Device custom-UI widgets).
- **Volumes & zones**: `volume`, `mutator_zone`, `damage_volume`, `fire_volume`, `skydive_volume`, `barrier`, `water`, `rift_point_volume`, `emp_volume_hazard_rocketracing`, `fishing_zone`, `campfire`.
- **Powerups**: `stat_powerup`, `health_powerup`, `damage_amplifier_powerup`, `visual_effect_powerup`, `grind_powerup` (base `powerup_device`; `[4120+]` `Clear(Agent)`).
- **Props, physics & world**: `prop_mover`, `prop_manipulator`, `physics_boulder`, `physics_tree`, `ball_spawner`, `explosive`, `progress_based_mesh`, `animated_mesh`, `service_station` (healthful/damageable), `spire_spike`, `scout_spire`, `overlord_spire`, `beacon`.
- **Vehicles**: abstract `vehicle_spawner_device` + ~37 `vehicle_spawner_*` variants (`sedan`, `sports_car`, `atk`, `taxi`, `pickup_truck`, `big_rig`, `boat`, `biplane`, `helicopter`, `quadcrasher`, `dirtbike`, `sportbike`, `tank`, `ufo`, `baller`, `cannon`, `driftboard`, `shopping_cart`, `surfboard`, `octane`, `hammerhead_choppa`, `siege_cannon`, `heavy_turret`, `armored_battle_bus`, `armored_transport`, `armored_assault_tank`, `nitro_drifter_sedan`, `valet_suv`, `getaway`, `drivable_reboot_van`, `war_bus`, `rocketracing`, `xwing`, `tie_fighter`, `n1_starfighter`, `turbolaser`, `df9`), `vehicle_mod_box_spawner` (+ `vehicle_mod_box_settings`).
- **Camera, controls & cinematics**: `gameplay_camera_fixed_point`/`_first_person`/`_fixed_angle`/`_orbit`, `gameplay_controls_side_scroller`/`_third_person`, `cinematic_sequence` — device-based; the Scene-Graph camera components (`/Fortnite.com/SceneGraphCameras`) are in `cameras.md`.
- **HUD, map & UI**: `hud_controller`, `hud_message` (`[3100+]` per-agent `Hide(Agent)`), `popup_dialog`, `map_controller`, `map_indicator`, `player_marker`, `player_reference`, `billboard`, `holoscreen`, `video_player`.
- **Audio-visual & ambience**: `audio_player`, `audio_mixer`, `radio`, `vfx_creator`, `vfx_spawner`, `post_process`, `skydome`, `customizable_light`.
- **Patchwork** (`Devices/Patchwork` submodule; music synthesis — all subclass `patchwork_device`, which is `<concrete>` non-final with only `Enable`/`Disable`): `music_manager` (shared tempo/key/timeline), `instrument_player`, `drum_player`, `omega_synthesizer`, `drum_sequencer`, `note_sequencer`, `note_progressor`, `note_trigger`, `speaker`, `lfo_modulator`, `step_modulator`, `value_setter`, `cable_splitter`, `distortion_effect`, `echo_effect`, `song_sync`.
