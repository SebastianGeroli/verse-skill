# Abilities (`/UnrealEngine.com/Abilities` + `/Fortnite.com/Abilities`)

Data-driven gameplay abilities: a scene-graph `ability` description (cooldowns/targeting/CanUse) that, on `Use`, spawns an `ability_effect` prefab entity whose `ability_effect_component` runs the lifecycle. The Fortnite layer adds a template ability driven by an input action, a target query, and a **timeline** of effect elements (damage, heal, animation, particles, sound, status effects, projectiles).

> **Digest = source of truth.** Exact signatures live in your build's generated digests; `grep` them. See `../uefn/toolchain.md`.

⚠️ **The entire surface is `@experimental`** and version-gated: the UE layer is `[4130+]`, everything in `/Fortnite.com/Abilities` and `/Verse.org/Timeline` is `[4200+]`. Expect churn; regenerate digests before relying on a signature.

## Core layer — `/UnrealEngine.com/Abilities` `[4130+ experimental]`

- **`ability_context`** (`class<concrete>`) — data passed on activation: `var Instigator:?agent`, `var Participants:[]entity`, `var Targets:[]entity`. Subclass to carry custom context for your effects.
- **`ability(context_type:subtype(ability_context), ability_effect_type:subtype(ability_effect))`** (`class<abstract>(has_icon)`) — the high-level description (can it run, cooldowns, distance/targeting). Members:
  - `Use<final>(AbilityEffectParent:entity, AbilityContext:context_type):?ability_effect_type` — **total, returns an option** (empty = didn't fire), not `<decides>`.
  - `CanUse(AbilityEffectParent:entity, AbilityContext:context_type)<decides><reads>:void` — the failable pre-check.
  - `BeginUseEvent`/`EndUseEvent : listenable(entity)` — payload is the spawned effect entity.
  - `MakeContext()<transacts>:context_type`, `MakeAbility()<transacts>:ability_effect_type` — factory hooks.
  - `var ActiveEffects:[]ability_effect_type` (read-only from outside), `var Icon:texture` (from `has_icon`).
  - It's a **parametric class** — the usual parametric-type toolchain hazards apply (`../gotchas.md`).
- **`ability_effect`** (`class<concrete>(entity)`) — the prefab spawned per use; carries `AbilityComponent:ability_effect_component`.
- **`ability_effect_component`** (`class<final_super>(component)`) — guaranteed on every effect prefab; owns the lifecycle:
  - protected overridables: `OnBeginUse()<transacts>`, `OnEndUse(Reason:?cancel_reason)<transacts>`, `CanCancel(Reason:cancel_reason)<transacts><decides>`.
  - `EndUse<final>(Reason:?cancel_reason)<transacts>` (protected), `Cancel<final>(Reason:cancel_reason)<transacts><decides>` (public — fails if `CanCancel` rejects).
  - `var Context:?ability_context`, `var Ability:?ability(ability_context, ability_effect)` (protected-write).
- **`MakeAbilityEffectComponent(component_type:castable_subtype(ability_effect_component))<converges>:component_type`** — factory for effect-component subclasses.
- **`cancel_reason`** (`class<abstract><castable>`) — lightweight "why did it end" tag; base subclasses `ability_ended`, `ability_removed_from_scene`. Downcast with `[]` to inspect.

## Timeline base — `/Verse.org/Timeline` `[4200+]`

Not abilities-specific, but abilities are its main consumer today. All elements are `cancelable`.

- **`timeline_element`** (`class(cancelable)`) — abstract base.
- **`timeline_element_point`** — one instant: `var Time:float` (`@editable`, private-write; `SetTime(NewTime)<transacts>`). May be ±`Inf`; **`NaN` is never updated or queried**.
- **`timeline_element_span`** — a range: `var BeginTime` (default `0.0`) / `var EndTime` (default `Inf`), `SetRange(Begin, End)<transacts>`. Must keep `BeginTime <= EndTime` and non-NaN or the element is ignored.

## Template abilities — `/Fortnite.com/Abilities` `[4200+ experimental]`

- **`fort_template_ability(context_type, prefab_type:subtype(fort_template_ability_effect))`** (`class<abstract>(ability(...))`) — the data-driven shape. `@editable` fields:
  - `var InputTrigger:?input_action(logic)` — activation binding (input actions: `../language/verse-org.md`).
  - `var TargetQuery:?ability_target_query` — who it can hit (below).
  - `var AbilityElements:[]timeline_element` — the effect timeline (elements below).
- **`fort_template_ability_effect`** (`class<concrete>(ability_effect)`) / **`fort_template_ability_effect_component`** — the matching prefab pair. Note `OnBeginUse`/`OnEndUse` here are **`<override><final>`** — you cannot override them further; customize behavior via timeline elements and `CanCancel` (still overridable).
- Cancel reasons: `fort_cancel_reason_ended` / `_canceled` / `_invalid` (all `<castable>`).

### Timeline element catalog (add to `AbilityElements`)

Points (`fort_ability_timeline_element_point`, sets `Time`):

- **`fort_ability_damage_point`** — `Amount:float`, `Behavior:fort_damage_behavior` (`enum<open>`: `EffectiveDamage`/`Health`/`Shield` — open, so `case` needs `_`). Damage semantics: `characters-combat.md`.
- **`fort_ability_heal_point`** — `Amount`, `Behavior:fort_heal_behavior` (`enum<open>`: `EffectiveHealth`/`Health`/`Shield`).
- **`fort_ability_projectile_point`** — `Projectile:concrete_subtype(projectile_template)` (class handle, not an instance), `SpawnOffset:transform` (Verse.org SpatialMath; muzzle nudge).
- **`fort_ability_particle_system_element_point`** — `ParticleSystem:particle_system`, `var ParticleAttachment:fort_ability_attach_socket`.
- **`fort_ability_sound_element_point`** — `Sound:sound_wave`.
- **`fort_ability_status_effect_point`** — `var StatusEffectDuration:float`; final subclasses `_icy_feet_`/`_slap_`/`_pepper_`/`_burn_point`.
- **`fort_ability_remove_item_participant_point`** — removes the granting item from the participant.

Spans (`fort_ability_timeline_element_span`, sets `BeginTime`/`EndTime`):

- **`fort_ability_animation_element_span`** — `Animation:animation_sequence`, `AnimationLayer:fort_ability_anim_layer` (`FullBody`/`UpperBody`), `EaseIn`/`EaseOut:float`.
- **`fort_ability_particle_system_element_span`**, **`fort_ability_sound_element_span`** — as the point forms, but active over the span.

`fort_ability_attach_socket` enum: `Root`/`LeftHand`/`RightHand`/`Head`/`Chest`/`LeftFoot`/`RightFoot`.

### Target queries

`ability_target_query` (base) → `self_ability_target_query` (caster only), `fort_reticle_ability_target_query{Target:fort_target_query_affiliation, Range:float}` (aim-at), `fort_wedge_ability_target_query{Target, Radius, CentralAngle}` (cone/AoE). `fort_target_query_affiliation`: `Any`/`Friendly`/`Hostile`.

### Projectiles

- **`fort_projectile_component`** (`class<final_super>(component)`) — `GetInstigator()<transacts><decides>:agent`, `GetSource()<transacts><decides>:entity` (the weapon/launcher/effect), `var Speed`/`Range`/`Gravity:float` (**cm/s, cm, cm/s² — 0 gravity = straight line**; runtime `SetSpeed`/`SetRange`/`SetGravity`), `HandleCollision(Result:projectile_impact_result)`, `OnImpactEvent:listenable(projectile_impact_result)`.
- **`projectile_impact_result`** (struct) — `Location`/`Normal` (Verse.org `vector3`), `HitEntity:?entity` (**empty for world geometry**).
- **`projectile_template`** (`class<concrete><internal>(entity)`) — **internal constructor**: you don't instantiate it in code; derive prefabs and reference them via `concrete_subtype(projectile_template)` handles (as in `fort_ability_projectile_point.Projectile`).

### Item integration

**`fort_item_ability_component`** (`class<final_super><final>(component)`) — put on an item entity to grant abilities: `var ItemAbilities` (active while held) vs `var ItemEquippedAbilities` (only while equipped) — both `@editable` `[]?ability(ability_context, ability_effect)`, with runtime `AddItemAbility(...)`/`AddItemEquippedAbility(...)<transacts>`; `var Equipped:logic` reflects state. Items/inventories: `itemization.md`.
