# Conversations — LLM NPCs (`/UnrealEngine.com/Conversations`)

`[4000+]` Generative-AI NPCs: a `persona_component` gives an entity a personality and a synthesized voice; its `ai_session` answers prompts as structured data and can call back into Verse. Overhauled in 42.00 — the old `llm_session`/`basic_prompt`/`persona_prompt` API is gone. Everything here depends on **voice channels** from `/Verse.org/Chat` `[4000+ experimental]` (`../language/verse-org.md`): a persona speaks *into a channel*, and a player can only prompt a persona sharing their channel.

> **Digest = source of truth.** Exact signatures live in your build's generated `UnrealEngine.digest.verse`; `grep` it. See `../uefn/toolchain.md`. Verified against build 42.00.

## `persona_component` (`class<final_super>(component)`)

Lives on an entity. **How it gets there** (engine-source-verified — the digest is silent on this):

- **NPCs: auto-attached by the editor modifier.** Give the NPC Character Definition the **"NPC Character Persona Modifier"** — at spawn it get-or-creates a `persona_component` on the NPC's entity and seeds `Personality` (from the modifier's prompt facts), `Voice`, and `PromptInterruptionRule`. Don't try to `AddComponents` a persona onto an NPC — impossible (user Verse can't add components to agents) and unnecessary (a component already present is reused, never duplicated).
- **Retrieval:** `Agent.GetFortCharacter[].GetEntity[].GetComponent[persona_component]` — `GetEntity[]` on `fort_character` is the only agent→entity bridge, and there is no direct agent→persona accessor (`persona_device` is `<epic_internal>`).
- **Standalone (announcer) personas:** `persona_component{}` IS user-constructible — `NonAgentEntity.AddComponents(array{persona_component{}})` works even on entities with no backing actor (the runtime spawns a hidden voice actor and auto-adds TTS playback). Configure via `SetPersonality`/`SetVoice` after attaching; construct-then-attach ordering is safe.

Configuration:

- `var<private> Personality<public>:message` + `SetPersonality(NewPersonality:message)<transacts><decides>:void` — **`SetPersonality`/`SetVoice` FAIL on Epic-bound (licensed-character) personas**: their persona data is locked, and reading `Personality` on them returns redacted text.
- `var<private> Voice<public>:voice_model` + `SetVoice(NewVoice:voice_model)<transacts><decides>:void` — an unset voice falls back to the NPC cosmetic's bound voice, else a stock default (`jonesy_voice`/`hope_voice`, heuristic pick).
- `var PromptInterruptionRule<public>:persona_interruption_rule` — what happens when the persona is prompted while already processing: `IgnoreUntilFinished` (new input dropped) / `InterruptOnPromptStart` (cut off immediately) / `InterruptOnResponseStart` (finish speaking, switch when the new response is ready). `enum<open>`.
- `GetAISession()<transacts>:ai_session` — the session that processes all this persona's prompts.

**Making it talk** (two overloads):

```verse
Persona.PromptToTalk[Instructions, Channel]                      # uses PromptInterruptionRule
Persona.PromptToTalk[Instructions, PerPromptRule, Channel]       # one-shot rule override
```

`(Instructions:message, Channel:voice_channel(member_info) where member_info:subtype(has_voice_member_info))<transacts><decides>:void`. **Not `<suspends>` — it's fire-and-forget**: `[]` succeeds when the prompt is *submitted*; generation and speech happen asynchronously and are observed via the events below. Fails immediately when the interruption rule blocks submission or the persona isn't in a channel. Errors after submission (timeout, moderation, …) surface on `PromptFailureEvent`, not as failure.

**Event surface** (subscribe/await like any `listenable`):

| Event | Payload | Fires |
|---|---|---|
| `StartHearEvent` / `StopHearEvent` | `agent` | player starts/finishes prompting the persona |
| `StartSayEvent` | `tuple(?agent, cancelable)` | persona begins speaking, **before moderation resolves**; `?agent` = the prompter (false for scripted prompts); `.Cancel()` the cancelable to shut it up |
| `CommitSayEvent` | `tuple(?agent, message)` | final text, **after moderation** — the original or a canned replacement. This is the text to mirror into captions/UI |
| `StopSayEvent` | `logic` | persona stops talking; `true` = it was interrupted |
| `PromptFailureEvent` | `ai_error` | an accepted prompt failed downstream |

## `ai_session` (`class<unique>`)

Obtained only via `Persona.GetAISession()`. Holds conversation history, pinned entries, and registered capabilities; **history is auto-compacted by the runtime** (completed goals get summarized). When the persona is in a voice channel, history propagates automatically between *all* sessions in that channel — but only conversation directly to/from a persona is recorded. `ClearHistory()<transacts>:void` wipes it.

- **`Prompt(Prompt:message, response_type:type)<suspends>:result(response_type, ai_error)`** `[4100+]` — sends instructions with the session's full context (personality, pinned facts, history) and parses the reply into your struct. `response_type` must be a struct containing only **numerics, `message`s, `logic`, enums, `agent`s, or structs/arrays of those**. Field names and type names steer the model — name them descriptively, and annotate with `@ai_description` (below). `<suspends>` (so it can't be `<decides>` — errors come back as the `result`): match on `.GetSuccess[]` / `.GetError[]`.
- **`RegisterAction(Definition:prompt_binding_definition, Required:logic, input_type:type, Callback:type{_(:input_type):void})<transacts>:cancelable`** `[4120+]` — registers a **fire-and-forget side-output the model may emit alongside any response** (during prompts and goals): the model fills an `input_type` value and your callback runs with it. `Required := true` forces the model to emit it on *every* response. `prompt_binding_definition = struct{Name:message, Description:message}` — both are injected into the AI prompt so the model knows when to use the capability; write the `Description` like a tool spec. `.Cancel()` the returned cancelable to unregister.
- **`@ai_description("...")`** `[4100+]` — data-field attribute (`@attribscope_data`): put it on `response_type`/`input_type` struct fields to tell the model what the field means. The attribute class is `<internal>`; use the `ai_description` constructor form only.

## Player-driven prompting

Bind a persona to the player's talk hotkey — the persona must share the player's voice channel:

```verse
Player.SetConversationTarget(Persona, Channel)   # <transacts>; channel is voice_channel(member_info)
Player.GetConversationTarget[]                    # <decides><reads>: tuple(persona_component, voice_channel(has_voice_member_info))
Player.ClearConversationTarget()                  # <transacts>
```

One conversation target per player; any number of players may target the same persona.

## Errors — `ai_error` hierarchy

`ai_error{Message:message}` with subclasses `ai_timeout_error` / `ai_throttled_error` (rate limit — back off) / `ai_moderated_error` (content blocked) / `ai_character_limit_error`. Downcast to react: `if (ai_throttled_error[E]) { ... }`. Delivered as `Prompt`'s `result` error or via `PromptFailureEvent`.

## Voices (`voice_model`)

`voice_model` is `<abstract><epic_internal>` with ~35 named `<final>` subclasses, each doc-tagged **Style, Pitch, Timbre** (Style ∈ Comic/Refined/Glamorous/Tough/Whimsical/Narrator/Cinematic; Pitch ∈ High/Med/Low; Timbre ∈ Warm/Natural/Gravelly). **16 of them (the licensed-character ones — `peely_voice`, `midas_voice`, `fishstick_voice`, `brite_bomber_voice`, …) have `<epic_internal>` constructors and cannot be instantiated from island code.** The 19 constructible ones: `elmira` (Refined/Med/Warm), `helsie_midnight` (Glamorous/Animated/Warm), `munitions_master` (Whimsical/High/Warm), `terns` (Narrator/Low/Warm), `lexa_hexbringer` (Whimsical/High/Warm), `battle_gamer_mae` (Whimsical/Med/Warm), `halley` (Tough/Med/Natural), `aura` (Tough/Low/Natural), `brute_gunner` (Cinematic/Med/Natural), `field_commander` (Comic/Med/Natural), `orin` (Comic/High/Natural), `moxie` (Narrator/Med/Warm), `clover_swift` (Cinematic/Med/Warm), `chase` (Comic/High/Natural), `guild` (Refined/Med/Warm), `sunspot` (Whimsical/High/Warm), `nezumi` (Glamorous/Low/Natural), `magnus` (Glamorous/Med/Natural), `revolt` — all `_voice` suffixed (`terns_voice{}` etc.).

## Gotchas

- **`PromptToTalk` succeeding ≠ the NPC spoke.** It only means "submitted". Await `CommitSayEvent` for the delivered text, and subscribe `PromptFailureEvent` or silent failures go unnoticed.
- **No channel, no conversation** — `PromptToTalk` fails and player prompting is inert until the persona (and player) are in a `voice_channel`. Channel setup lives in `/Verse.org/Chat` (`entity.AddChatChannel`).
- `ai_session.Prompt` is `<suspends>`: structure the flow like any async op (race with a `Sleep` timeout if the built-in `ai_timeout_error` window is too generous), and remember it can't sit in a failure context.
- Latency is real (LLM round-trip): drive NPC "thinking" states off `StartHearEvent`→`StartSayEvent` gaps rather than assuming instant replies.
- Shared-channel history propagation means two personas in one channel hear each other's conversations — isolate channels when personas must not share knowledge.
- Any use of `persona_component` (in code or via the editor modifier) stamps the island with the **"Conversations" publishing mutator** — expect the corresponding publishing requirements.

Related: NPC bodies/behaviors (`ai.md`), voice channels (`../language/verse-org.md` → Chat), quests that reward conversation goals (`progression.md`).
