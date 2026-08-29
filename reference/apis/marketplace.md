# Marketplace — In-Island Transactions (`/Fortnite.com/Marketplace`)

Real-money (V-Bucks) commerce inside an island: define `entitlement`s, sell them via `offer`s, query/consume ownership. Everything a player buys is tracked by the platform commerce system (refunds, moderation, cross-session persistence included).

> **Digest = source of truth.** Exact signatures live in your build's generated `Fortnite.digest.verse`; `grep` it. See `../uefn/toolchain.md`. Verified against build 42.00.

## `entitlement` — the thing a player owns

`entitlement := class<castable>(has_icon, has_description)` — subclass it per sellable thing; **your derived type must be `<concrete>`** to be used by the purchase system. Display comes from the inherited interfaces: `var Icon:texture` (`has_icon`) and `var Name/Description/ShortDescription:message` (`has_description`, all `@editable`).

Fields (all `logic`/`int` policy knobs with defaults):

- `MaxCount:int` — a player may own up to this many of a **consumable**; a **non-consumable** is capped at one regardless.
- `Consumable:logic` — enables `ConsumeEntitlement`.
- `PaidRandomItem:logic` — mark loot-box-style offers (gates on `RestrictPaidRandomItems` platforms/territories).
- `PaidArea:logic` — marks paid-access areas.
- `ConsequentialToGameplay:logic` — **must be `true` if the entitlement gives a meaningful gameplay advantage** (disclosure requirement).

## `offer` — the thing a player buys

`offer := class<abstract><castable><internal>(has_icon, has_description)` — abstract base with an internal constructor; instantiate the subclasses:

- **`entitlement_offer`** — sells one entitlement: `@editable EntitlementType:concrete_subtype(entitlement)` `[3800+]`.
- **`bundle_offer`** — sells several offers at once: `Offers:[]tuple(offer, int)` (offer + quantity).

Shared surface:

- `@editable Price:price_dimension`. The only concrete price today is **`price_vbucks`** (`<final><computes>`, internal constructor): create with `MakePriceVBucks(Amount:float)<converges>`, read back with `GetPriceVBucks(P):float`.
- **Region/age gating:** override `GetMinPurchaseAge(CountryCode:string, SubdivisionCode:string, PlatformFamily:string)<computes><decides>:int` — codes are ISO-3166-1 A-2 / ISO-3166-2 (subdivision may be `""`), `PlatformFamily` ∈ Android/iOS/macOS/Nintendo/PlayStation/Windows/Xbox/Luna/GeForceNow. **Fail = don't sell there**; succeed = minimum purchase age (an age above the region's highest available age also hides the offer).

## The purchase & entitlement API (module functions — all player-scoped)

All the flows are **`<suspends>`** (platform round-trips) — so none of this can sit in a `<decides>` chain; sequence it in `OnSimulate`/`spawn` and convert failures beforehand.

- `BuyOffer(Player, Offer)<suspends>:logic` — shows the Epic purchase UI for one offer; `true` iff purchased.
- `ShowOffersDialog(Player, Offers:[]offer, ?Title:message)<suspends>:void` — the Epic-provided storefront over several offers.
- `GrantEntitlement(Player, concrete_subtype(entitlement), ?Count)<suspends>:logic` `[3800+]` — direct grant (rewards, comp): **bypasses disclosure gating**, respects `MaxCount`.
- `ConsumeEntitlement(Player, concrete_subtype(entitlement), ?Count)<suspends>:logic` `[3800+]` — spends a consumable; `false` if not consumable or the player owns fewer than `Count`.
- `GetPurchasedEntitlements(Player, entitlement_type:subtype(entitlement))<suspends>:[]tuple(entitlement_type, int)` — owned entitlements + counts, **including derived types**. ⚠ **Passing the base `entitlement` type directly suspends forever** — always pass your concrete/derived subtype.
- `GetEntitlementsChangedEvent(Player, subtype(entitlement)):listenable(tuple(player, []entitlement_change(entitlement_type)))` — fires on quantity changes **including refunds and moderation**, so reconcile state here, not only on purchase. `entitlement_change{Entitlement:t, Quantity:int (total now owned), Change:int (delta)}`. ⚠ **Runtime error if the base `entitlement` type is used directly.**
- Restriction predicates (`<decides><reads>`): `RestrictPaidRandomItems(Player)` / `RestrictDirectPromptsToPurchase(Player)` — succeed when the player's platform/territory/age/user-config forbids that mechanism; check before offering loot-box or direct-prompt flows.

```verse
# typical startup reconcile + live updates for one entitlement type
Owned := GetPurchasedEntitlements(Player, my_vip_pass)      # <suspends>; NEVER pass `entitlement` itself
GetEntitlementsChangedEvent(Player, my_vip_pass).Subscribe(OnVipChanged)
```

## `offer_interactable_component` — buy-in-world (Scene Graph)

`class(interactable_component)` (subclassable — no `<final_super>`): attach to an entity for an interact-to-purchase prompt.

- `@editable var Offer:offer` — what interacting sells.
- `CanInteract<override>(Agent)<decides><reads>` — pre-wired to the offer's availability; `InteractMessage<override>(Agent)<decides><reads>:message`.
- `OnSucceed<protected>(Agent):void` — override for post-purchase behavior; `var SucceededEventHandle:?cancelable` holds its subscription.
- Interactable knobs (cooldowns, duration, limits): see `scene-graph.md` → interactable components.

## Cross-module tie-ins

- **`entitlement_quest_reward`** (`/UnrealEngine.com/Progression` `[experimental]`): a `quest_reward` that grants `var Entitlement:concrete_subtype(entitlement)` × `var Quantity:int` to each eligible participant via the platform service — auto-rewarded quests can pay out entitlements. See `engine.md` → Progression.
- Rarity/description/icon presentation on sellable item entities: `scene-graph.md` (`rarity_component`, `description_component`, `icon_component`).
