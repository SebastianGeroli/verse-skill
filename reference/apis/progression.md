# Progression & Quests

The quest system, layered across all three API namespaces. **Entirely `@experimental`.** Gates: `/Verse.org/Progression` and `/UnrealEngine.com/Progression` `[4110+]`; `quest_category`/`has_quest_presentation` and all of `/Fortnite.com/Progression` `[4200+]`.

> **Digest = source of truth.** Exact signatures live in your build's generated digests; `grep` them. See `../uefn/toolchain.md`. Verified against build 42.00.

## The three layers — what you author against

| Layer | Role |
|---|---|
| `/Verse.org/Progression` | Participation bookkeeping: abstract `quest`, `quest_collection`, participants, join/abandon/complete events |
| `/UnrealEngine.com/Progression` | The authorable "do x, get y" machinery: `basic_quest`, objectives, rewards, quest-UI presentation |
| `/Fortnite.com/Progression` | Surfacing in the Fortnite quest log: `fort_quest_collection`, `fort_quest_category` |

You subclass the **UE layer** (`basic_quest`, `quest_objective`, `quest_reward`), drive progress from gameplay events, and join quests into a `fort_quest_collection` so they display. The Verse.org layer's types flow through everything as payloads.

## `/Verse.org/Progression` — participation

- **`quest`** (`class<abstract><castable><unique>`): `Complete()<transacts>:void` (**idempotent** — no-op if already complete; fires `CompleteEvent`), `IsComplete()<decides><reads>`, `JoinEvent`/`AbandonEvent:listenable(quest_membership)`, `CompleteEvent:listenable(quest)`. `<unique>` — quests compare by identity.
- **`quest_collection`** (`class<unique>`, concrete): `JoinQuest(Quest, Participant, Info)` / `AbandonQuest(Quest, Participant)` — both `<transacts>:result(void, []join/abandon_quest_error)` (**call with `()`, not `[]`** — "cannot join" is an error result, not Verse failure); collection-wide `JoinEvent`/`AbandonEvent`/`CompleteEvent`; membership maps `Quests:[quest][]quest_membership` and `Participants:[quest_participant][]quest_membership` (read-only — `var<private>`).
- **`quest_participant`** (abstract, `<unique>`) with per-participant `JoinEvent`/`AbandonEvent`/`CompleteEvent:listenable(quest_membership)`; **`agent_quest_participant`** (`<final>`, `Agent:agent`) is the concrete one — backed by any agent (player, NPC).
- **`quest_participant_info`** (`class(member_info_interface)` — the `/Verse.org/AgentGroup` interface): three `var logic` flags that gate everything downstream — `Contributes` (may progress objectives), `Observes` (sees quest state; **the Fortnite UI shows a quest once per observing participant**), `Receives` (gets rewards).
- **`quest_membership`**: the record `{Quest, Participant, Info}` — payload of every join/abandon event.

## `/UnrealEngine.com/Progression` — quests, objectives, rewards

- **`basic_quest`** (`class<abstract>(quest, has_icon, has_description)`): the "do x, get y" quest. `var<private> Objective:quest_objective` (required field — set at construction) + `SetObjective(o)<transacts>` (replaces the model on a **live** quest; auto-completes if the new objective is already complete), `var Rewards:[]quest_reward`. **Auto-complete:** when `Objective.IsComplete[]` succeeds the quest calls `Complete()` itself. **Auto-reward:** on completion each reward runs `GetRecipients()` → `GrantReward()`.
- **`quest_objective`** (abstract, `has_icon`+`has_description`): override `GetProgress()<reads>:message` (e.g. "3 / 10"), `IsComplete()<decides><reads>`, `GetContributors(Memberships)<reads>` (default: all with `Info.Contributes`); call `SignalProgressEvent()<transacts>` (`<final><protected>`) from your subclass whenever progress changes; `CompleteEvent`/`ProgressEvent:listenable(quest_objective)`.
- **`progress_quest_objective`** (concrete counter): `SetProgress(Amount:float)<transacts>` (clamped to `[0, RequiredCount]`), `SetRequiredCount(Amount)` (clamped `[1, MaxFloat]`); `RequiredCount:float` is a **required field** (no default); progress is `float`; `GetProgress()` renders "X / Y"; complete when `Progress >= RequiredCount`.
- **`quest_reward`** (abstract, `has_icon`+`has_description`): override `GetRecipients(Memberships)<reads>` (default: all with `Info.Receives`) and `GrantReward(EligibleParticipants)<transacts>:result(void, []grant_quest_reward_error)`; `GrantEvent:listenable(tuple(quest_reward, []quest_participant))` fires after granting.
- **`entitlement_quest_reward`**: `var Entitlement:concrete_subtype(entitlement)` + `var Quantity:int` — resolves each participant's agent as a `player` and grants via the platform entitlement service (the Marketplace tie-in — see `marketplace.md` for entitlement semantics and the disclosure rules that still apply).
- **`quest_category`** `[4200+]` (`class<abstract><unique><castable>(has_icon, has_description)`): subclass to define quest-UI categories; `SortOrder:rational` (lower sorts first), `Parent:?quest_category` for nesting.
- **`has_quest_presentation`** `[4200+]` (interface — implement on your quest class): `var ShowNotification:logic` (HUD toasts on granted/progress/completed), `var AllowFavorite:logic` (pin to HUD tracker), `var Categories:[]quest_category` (empty = uncategorized).

## `/Fortnite.com/Progression` `[4200+]` — the quest log

- **`fort_quest_collection`** (`class<final>(quest_collection)`, concrete — construct with `fort_quest_collection{}`): quests joined here show in the Fortnite quest UI, once per **observing** participant. No engine accessor exists in the digest — you create and share the instance yourself (e.g. on a manager component).
- **`fort_quest_category`** (`class<abstract><castable>(quest_category)`): subclass for Fortnite quest-log categories.

## Wiring & gotchas

- **Typical flow:** define `my_quest := class(basic_quest)` (implement `has_quest_presentation` for UI control) with a `progress_quest_objective`; per player, `JoinQuest(Q, agent_quest_participant{Agent := P}, quest_participant_info{Contributes := true, Observes := true, Receives := true})` into a `fort_quest_collection`; drive `SetProgress` from gameplay events (`DamagedEvent`, elimination, etc. — `damage_result.Source:game_action_causer` exists precisely to identify *what* caused the action for quest logic).
- **Nothing here persists automatically** — the digest says nothing about saving quest state across sessions; persist progress yourself via a persistent `weak_map` and re-`SetProgress` on join.
- **Battle Pass XP is NOT this API** — real XP comes from the calibrated `accolades_device` (`Award(Agent)`; see `devices.md`).
- The no-code alternative for simple goals is the `tracker`/`objective`/`timed_objective` devices (`devices.md`); this API is for quests whose logic, display, and rewards you own in Verse.
