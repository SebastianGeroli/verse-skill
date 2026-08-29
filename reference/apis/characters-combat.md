# Characters, Combat, Playspaces & Teams (`/Fortnite.com`)

The agent-centric gameplay surface: `fort_character`, damage/health/shield, playspaces, teams, animation, AI, vehicles. Verified against build 42.00.

> **Digest = source of truth.** Exact signatures live in your build's generated `Fortnite.digest.verse`; `grep` it (~12.4k lines — don't read it whole). See `../uefn/toolchain.md`.

## `/Fortnite.com/Characters` — `fort_character` (central)

`fort_character` is an `interface<unique>` implementing `positional, healable, healthful, damageable, shieldable, game_action_instigator, game_action_causer`.

```verse
Char := Agent.GetFortCharacter[]            # (agent).GetFortCharacter()<transacts><decides>:fort_character
Char.GetAgent[] / Char.GetEntity[]          # bridge to agent / Scene Graph entity
Char.EliminatedEvent()                       # listenable(elimination_result)
Char.GetViewRotation() / GetViewLocation()   # ⚠ returns Temporary (X/Y/Z) SpatialMath
Char.GetTransform()                          # ⚠ Temporary SpatialMath (from positional)
Char.GetGlobalTransform()                    # positional ALSO exposes the current (/Verse.org) transform
Char.IsActive[] / IsDownButNotOut[] / IsCrouching[] / IsOnGround[] / IsInAir[] / IsFalling[] / IsGliding[] / IsFlying[] / IsInWater[]
Char.TeleportTo[Position, Rotation]          # <transacts><decides>; Temporary vectors; applies yaw+pitch
Char.PutInStasis(stasis_args{AllowTurning:=…, AllowFalling:=…, AllowEmotes:=…}) / ReleaseFromStasis()
Char.Show() / Hide() / SetVulnerability(logic) / IsVulnerable[]
Char.GetLinearVelocity() / SetLinearVelocity(v) / ApplyLinearImpulse(v) / ApplyForce(v) / GetMass()   # Verse.org vectors; m/s, N·s, N, kg; no-ops if physics disabled
Char.JumpedEvent()                           # listenable(fort_character)
Char.CrouchedEvent() / SprintedEvent()       # listenable(tuple(fort_character, logic)) — logic = entering the state
```

- **The character changes on respawn** — re-fetch and re-subscribe; don't cache a `fort_character` across deaths.
- **Most actions silently fail when the character is inactive** — test `IsActive[]` when you care.
- `Show`/`Hide`/`SetLinearVelocity`/`ApplyLinearImpulse`/`ApplyForce` are plain `:void` (no_rollback) — hoist them out of `if` conditions/`for` domains.
- Instigator bridges: `(agent).GetInstigator()<transacts>:game_action_instigator`, `(game_action_instigator).GetInstigatorAgent[]:agent`.

## `/Fortnite.com/Game` — damage / health / shield (build your combat on this)

```verse
positional : GetTransform()<transacts>              # ⚠ Temporary SpatialMath
             GetGlobalTransform()<transacts>        # current /Verse.org SpatialMath
healthful  : GetHealth()<transacts>:float           # 0.0..MaxHealth
             SetHealth(H)<transacts>                # clamps to [1.0, MaxHealth]; CANNOT SetHealth(0) — eliminate via Damage instead
             GetMaxHealth(), SetMaxHealth(M)        # M clamps [1.0, Inf); current health RESCALES proportionally on change
shieldable : GetShield/SetShield (clamps [0.0, MaxShield]) / GetMaxShield/SetMaxShield (rescales like MaxHealth),
             DamagedShieldEvent(), HealedShieldEvent()
damageable : Damage(Amount:float), Damage(damage_args), DamagedEvent():listenable(damage_result)
healable   : Heal(Amount:float),  Heal(healing_args),  HealedEvent():listenable(healing_result)

damage_args   = struct{ ?Instigator:?game_action_instigator, ?Source:?game_action_causer, Amount:float }
damage_result = struct{ Target:damageable, Amount:float, ?Instigator, ?Source, IsWeakpointDamage:logic }
healing_args/healing_result — analogous (no IsWeakpointDamage).
game_action_instigator / game_action_causer — marker interfaces (who/what caused it).
elimination_result = struct{ EliminatedCharacter:fort_character, EliminatingCharacter:?fort_character }
```

- **`Damage(…)`/`Heal(…)` are plain `:void` (no_rollback)** — amounts < 0 do nothing; don't call them inside `if` conditions/`for` domains.
- Apply damage with a known source: `Char.Damage(damage_args{Amount := 25.0, Instigator := MyInstigator})`. Listen: `Char.DamagedEvent().Subscribe(OnDamaged)`.
- **`fort_round_manager`** (via `entity.GetFortRoundManager[]`): `SubscribeRoundStarted(Callback:type{_()<suspends>:void})` — invoked on round start **and immediately if a round is already ongoing**; all callbacks are cancelled when the round ends. `SubscribeRoundEnded(Callback:type{_():void})`. Both return `cancelable`. `var RoundNumber:int` — 0 before the first round, incremented *before* RoundStarted callbacks fire, retained between rounds, reset to 0 at game end.

## `/Fortnite.com/Playspaces` & `/Fortnite.com/Teams`

- **`fort_playspace`** (`creative_object.GetPlayspace()` or `entity.GetPlayspaceForEntity[]` — fails if the entity isn't in the scene): `GetPlayers()` (humans), `GetParticipants()` (humans + registered AI agents), `GetTeamCollection()`, `PlayerAddedEvent()`/`PlayerRemovedEvent()` (`listenable(player)`), `ParticipantAddedEvent()`/`ParticipantRemovedEvent()` (`listenable(agent)`).
- **`fort_team_collection`**: `GetTeams()`, `AddToTeam[Agent,Team]`, `IsOnTeam[Agent,Team]`, `GetAgents[Team]`, `GetTeam[Agent]`, `GetTeamAttitude[Team1,Team2]` **and** `GetTeamAttitude[Agent1,Agent2]` → `team_attitude` (`Friendly`/`Neutral`/`Hostile`). All `<transacts><decides>` (fail on unknown team/teamless agent).

## `/Fortnite.com/Animation/PlayAnimation`

`fort_character.GetPlayAnimationController[]` → `play_animation_controller`:

- `PlayAndAwait(Seq:animation_sequence, ?PlayRate, ?PlayCount, ?StartPositionSeconds, ?BlendInTime, ?BlendOutTime)<suspends>:play_animation_result` (`Completed`/`Interrupted`/`Error`).
- `Play(Seq, ?…):play_animation_instance` — fire-and-hold handle: `GetState()` (`BlendingIn`/`Playing`/`BlendingOut`/`Completed`/`Stopped`/`Interrupted`/`Error`), `Stop()`, `Await()<suspends>:play_animation_result`, `IsPlaying[]`, events `CompletedEvent`/`InterruptedEvent`/`BlendedInEvent`/`BlendingOutEvent`.

## `/Fortnite.com/AI` `[3900+]`

Custom NPC brains (`npc_behavior`), navigation (`navigatable` vs `npc_actions_component` — different vector and result types), guard actions/perception with alert levels, focus/leash, sidekicks `[3800+]`.
Covered in depth in `ai.md`.

## `/Fortnite.com/Vehicles`

- **`fort_vehicle`** (`interface<unique>(positional, healthful, damageable, game_action_causer, showable)`; via `fort_character.GetVehicle[]`): `GetOccupants()<reads>:[]agent` (`GetPassengers()` is `@deprecated` for it), `GetDrivers()<reads>:[]agent`, `AddAgent[Agent]`/`RemoveAgent[Agent]`/`RemoveAll()`, `GetSeats()<reads>:[]fort_vehicle_seat`, `TeleportTo[Pos, Rot]` (Temporary vectors), `IsOnGround[]`/`IsInAir[]`/`IsInWater[]`, `var Speed:float` (m/s), fuel `GetFuelRemaining()`/`GetFuelCapacity()` (**-1.0 when the vehicle doesn't use fuel**), boost `var BoostRemaining/BoostCapacity:?float` (`false` when unused).
- **`fort_vehicle_seat`**: `IsDriverSeat[]`, `var Occupant:?agent`, `SetOccupant[?agent]` (fails if occupied; pass `false` to eject), `var Vehicle:fort_vehicle`.

## `/Fortnite.com/FortPlayerUtilities`

`(player).SendToLobby()`, `(agent).Respawn(Pos, Rot)<transacts>` (Temporary vectors; applies yaw only), spectator queries: `(agent).IsSpectator[]`, `(player).GetPlayersSpectating():[]player`, `(agent).GetSpectatedAgent[]<decides><reads>:agent`, `(agent).GetSpectators():[]agent`.

## Related

`/Fortnite.com/Progression` `[4200+ experimental]` (`fort_quest_category`, `fort_quest_collection`) builds on the engine quest system — see `engine.md`.
