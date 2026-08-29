# NPC / AI (`/Fortnite.com/AI`)

Custom NPC behaviors, navigation, perception, guard combat actions, focus/leashing, and sidekicks. Module gate `[3900+]` for the NPC/guard components; sidekicks and spark mode are `[3800+]`.

> **Digest = source of truth.** Exact signatures live in your build's generated `Fortnite.digest.verse`; `grep` it. See `../uefn/toolchain.md`. Verified against build 42.00.

## The shape of the system

- **You never construct or attach these components yourself** — every class here is `<epic_internal>` (and the components `<final_super>`). The NPC infrastructure (an **NPC Character Definition asset** or an `npc_spawner_device`) spawns the agent and wires the components; your code *gets* them: `agent.GetNPCBehavior[]`, `Behavior.GetEntity[]` then `Entity.GetComponent[npc_actions_component]`, or the `fort_character.Get*` extension methods below.
- **Two parallel navigation surfaces — don't mix their vector types or result types:** the older `navigatable` interface (per `fort_character`, Temporary X/Y/Z vectors, plain `navigation_result` enum) and the Scene Graph `npc_actions_component` `[3900+]` (current `/Verse.org` vectors, `result(...)` returns). Prefer the component on NPCs that have it.
- All the `<suspends>` actions are **interruptible** — cancelling the arm (e.g. losing a `race`) stops the action; several (`MaintainFocus`, `Focus` with `?LockFocus := true`) never complete on their own and are *meant* to be raced.
- The plain `:void` mutators (`StopNavigation`, `SetMovementSpeedMultiplier`, `Tether`/`Untether`, `SetLeashPosition`/`SetLeashAgent`/`ClearLeash`) are **no_rollback** — hoist them out of `if` conditions / `for` domains (error 3512).

## `npc_behavior` — your code's entry point

```verse
my_guard_brain := class(npc_behavior):
    OnBegin<override>()<suspends>:void =
        if (Agent := GetAgent[], Char := Agent.GetFortCharacter[], Ent := Char.GetEntity[],
            Actions := Ent.GetComponent[npc_actions_component]):
            loop:
                R := Actions.NavigateTo(MakeNavigationTarget(NextWaypoint()))
                if (R.GetError[]) { break }
    OnEnd<override>():void = {}   # NPC removed from simulation
```

- `npc_behavior` is `class<abstract>`: override `OnBegin()<suspends>` (called when the NPC joins the simulation — this is the brain's coroutine) and `OnEnd()` (removal). `GetAgent()`/`GetEntity()` are `<transacts><decides>`.
- Attach it in the **NPC Character Definition** asset (or on an `npc_spawner_device`); retrieve from outside with `(agent).GetNPCBehavior[]` and downcast: `if (Brain := my_guard_brain[Agent.GetNPCBehavior[]])`.
- `OnBegin` is a `<no_rollback>` lifecycle context — same rules as Scene Graph lifecycle methods (no rollback-dependent `<decides>` work at top level).

## Navigation

- **`MakeNavigationTarget(Position)`** — overloads for **both** SpatialMath `vector3` types — and `MakeNavigationTarget(Agent:agent)` (tracks the moving agent). `navigation_target` itself is opaque.
- **`navigatable`** (via `(fort_character).GetNavigatable[]`): `NavigateTo(Target:navigation_target, ?MovementType:movement_type, ?ReachRadius:float, ?AllowPartialPath:logic)<suspends>:navigation_result` (`Reached`/`PartiallyReached`/`Interrupted`/`Blocked`/`Unreachable`), `GetCurrentDestination()<transacts><decides>` (⚠ Temporary vector3), `StopNavigation()`, `Wait(?Duration)<suspends>`, `SetMovementSpeedMultiplier(f)` (clamped **[0.5, 2.0]**).
- **`npc_actions_component`** `[3900+]`: `NavigateTo(same args)<suspends>:result(navigation_action_success_type, navigation_action_error_type)` (`Reached`/`PartiallyReached` vs `Invalid`/`Interrupted`/`Blocked`/`Unreachable` — both enums `<open>`), `GetCurrentDestination()<transacts><decides>` (current `/Verse.org` vector3), `Idle(?Duration)<suspends>`, `Focus(Location, ?LockFocus)<suspends>` / `Focus(Target:entity, ?LockFocus)<suspends>` (with `LockFocus := false` it completes once facing; `true` holds forever), `var MovementSpeedMultiplier:float` (clamped [0.5, 2.0]), `StopNavigation()`.
- **`movement_type`** is `enum<open>`: `Walking`/`Running`/`Sprinting` — but `Walking`/`Running` collide with other in-scope identifiers, so the digest ships a **`movement_types` module with `Walking`/`Running` constants** (no `Sprinting` there; use `movement_type.Sprinting`). Prefer `movement_types.Walking`/`movement_types.Running`.

## Guard actions — `guard_actions_component` (extends `npc_actions_component`) `[3900+]`

All actions `<suspends>:result(void, ai_action_error_type)` where `ai_action_error_type` = `Failure` (failed during execution) / `Canceled` / `Disallowed` (not allowed to start; `enum<open>`):

- `RoamAround(?MovementType)` — roams **anywhere** unless tethered; `Tether(Location|Target, Radius)` (cm) / `Untether()` are the plain-`void` bounds.
- `MoveInRangeToAttack()`, `Attack(Target:entity)` — **the target must already be detected** (perception below), or you get `Disallowed`.
- `Revive(Target:entity)`, `Crouch()`/`StandUp()`, `Jump()`, `Slide()`, `PlayRandomEmote()`.

## Perception — `npc_awareness_component` / `guard_awareness_component` `[3900+]`

- **`npc_awareness_component`**: `DetectedTargets:[]npc_target_info` (read-only), events `DetectTargetEvent`/`SeeTargetEvent` (sight sense)/`HearTargetEvent` (hearing)/`TouchTargetEvent` (touch) — all `listenable(npc_target_info)` — and `ForgetTargetEvent:listenable(entity)`.
- **`guard_awareness_component`** adds obstacles (`DetectedObstacle:?entity`, `DetectObstacleEvent`/`ForgetObstacleEvent:listenable(entity)`), the primary threat (`PrimaryThreat:?npc_target_info`, `PrimaryThreatChangeEvent`), and alert state: `AlertLevel:guard_alert_level` + `AlertLevelChangeEvent:listenable(guard_alert_level)`, per-target `GetAlertLevel(Target:entity)<reads>:guard_alert_level`.
- `guard_alert_level` = `Unaware` → `Suspicious` (seen, not identified) → `Alerted` (identified) / `LostTarget` (identified but no longer visible).
- **`npc_target_info`**: `Target:entity`, `HasLineOfSight:logic`, `Attitude:team_attitude`, `LastKnownPosition` (`/Verse.org` vector3) — all externally read-only — plus `OnUpdateEvent:listenable(tuple())` (await it to react to field changes on a tracked target).

## Focus & leash (per `fort_character`)

- `(fort_character).GetFocusInterface[]` → `focus_interface`: `MaintainFocus(Location|Agent)<suspends>` — **never completes unless interrupted**; run it as a `race`/`sync` arm alongside the movement it should accompany. (⚠ Temporary vector3.)
- `(fort_character).GetFortLeashable[]` → `fort_leashable`: `SetLeashPosition(Location, InnerRadius, OuterRadius)` / `SetLeashAgent(Agent, InnerRadius, OuterRadius)` / `ClearLeash()` — radii in cm, range 0..20000, `OuterRadius >= InnerRadius`. (⚠ Temporary vector3; plain voids.)

## Sidekicks `[3800+]`

- **`sidekick_component`** (abstract base): `GetMood()<reads>:sidekick_mood`, `var MoodOverride:?sidekick_mood` (locks the automatic mood system), `ChangeMoodEvent:listenable(tuple(sidekick_mood, sidekick_mood))` (previous, new), `PlayReaction(Reaction)<transacts><decides>` (queued, not immediate — monitor `StartPlayReactionEvent`/`StopPlayReactionEvent:listenable(sidekick_reaction)`), `var IdleAnticsEnabled:logic`.
- **`npc_sidekick_component`** (NPC sidekicks) adds `ApplyEquippedSidekickCosmetic(Agent)<transacts><decides>` (copies the agent's locker sidekick look; fails without the FortniteSidekick cosmetic look or an equipped sidekick).
- **`equipped_sidekick_component`** (an agent's own equipped sidekick; also `showable` — `var Show:logic`) adds `GetOwningAgent()<transacts><decides>:agent`, `var AutomaticReactionsEnabled:logic`, `var DefaultInteractionEnabled:logic` (the built-in player interaction).
- Moods (`enum<open>`): `Neutral`/`Combat`/`Worried`/`Bored`. Reactions (`enum<open>`): `Happy`/`Dance`/`Emote`/`Angry`/`Worried`/`Attack`/`HitReact`/`Sleeping`/`Eat`.
- **`spark_mode_component`** `[3800+]`: the entity auto-transforms into a floating spark in impassable terrain / to cut clutter — `var SparkModeAlwaysActive:logic` forces it; `BeginSparkModeEvent`/`EndSparkModeEvent:listenable(tuple())`.

## Related surfaces

- **Spawning NPCs** — devices in `/Fortnite.com/Devices` (details in `devices.md`): `npc_spawner_device` (`SetNPCCharacterDefinition(...)<transacts><decides>`, `Spawn()`, `SpawnAt(Position, ?Rotation)<suspends>:?agent` (current-namespace vectors), `DespawnAll(?Instigator)`, `GetAgents()`, `SpawnedEvent:listenable(agent)`, `EliminatedEvent`), `guard_spawner_device` (hireable Fortnite guards), `character_device` (single placed NPC), `creature_spawner_device`/`creature_placer_device`, `wildlife_spawner_device`, `sentry_device`, and `ai_patrol_path_device` (`Assign(Patroller:agent)`, `GoToNextPatrolGroup(...)`, node/patrol events) for waypoint patrols without hand-rolled `NavigateTo` loops.
- **LLM-driven conversation NPCs** — `/UnrealEngine.com/Conversations` (`persona_component`, `ai_session`): `engine.md`.
- **Playing animations on characters** — `characters-combat.md` → Animation; **combat/health** — same file.
