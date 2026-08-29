# Verse

A [Claude Code skill](https://code.claude.com/docs/en/skills) for the Verse programming language and the UEFN Scene Graph, including the Verse/UnrealEngine/Fortnite APIs.

Claude loads `SKILL.md` on its own when a task touches Verse or UEFN, and reads the deeper reference files as needed. They are organized into three folders — `language/` (the Verse language itself plus the `/Verse.org` native API), `uefn/` (toolchain and project specifics), and `apis/` (one file per engine/gameplay API area) — with two cross-cutting files at the root:

- `reference/language/language.md`: types, literals, variables, operators, functions, classes, modules, advanced types
- `reference/language/effects-failure-concurrency.md`: effect specifiers, failure, STM rollback, concurrency, events, live variables
- `reference/language/verse-org.md`: the `/Verse.org` native API — core functions, math, events, simulation, SpatialMath, input
- `reference/uefn/toolchain.md`: API digests, the build/verify loop, project and module layout, publishing, the two VMs
- `reference/apis/scene-graph.md`: entities, components, lifecycle, queries, scene events, transforms
- `reference/apis/characters-combat.md`: fort_character, damage/health, playspaces, teams, animation, vehicles
- `reference/apis/ai.md`: NPCs — behaviors, navigation, guard actions/awareness, sidekicks, spawner devices
- `reference/apis/abilities.md`: the gameplay-ability system (fort_template_ability, timelines, projectiles)
- `reference/apis/cameras.md`: camera components, camera director, camera modifier stacks
- `reference/apis/itemization.md`: inventories and items, weapons (Armory), item catalogs
- `reference/apis/marketplace.md`: in-island transactions — offers, entitlements, V-Bucks pricing, purchase gating
- `reference/apis/devices.md`: creative_device, creative_prop, the categorized 193-device catalog
- `reference/apis/ui.md`: UI widgets, player_ui, HUD control
- `reference/apis/input.md`: player input — actions/mappings, input events, built-in mappings, input-method detection
- `reference/apis/engine.md`: diagnostics/debug_draw, JSON, WebAPI, curves, quests, SortBy
- `reference/gotchas.md`: common mistakes and the right idiom (cross-cutting)
- `reference/patterns.md`: component and system patterns (cross-cutting)

## Install

For all your projects:

```bash
git clone https://github.com/magnusenebakk-epic/verse-skill.git ~/.claude/skills/verse
```

For a single project, so everyone who clones it gets the skill:

```bash
git clone https://github.com/magnusenebakk-epic/verse-skill.git <project>/.claude/skills/verse
```

The skill triggers on its own when you work on Verse code. Invoke it explicitly with `/verse`. Update with `git pull`.

## Notes

- Written for the current toolchain, where functions default to `<no_rollback>` and every `<decides>` function needs an explicit `<computes>` or `<transacts>`. Update the skill when that changes.
- Some helpers in `patterns.md` (`RecursiveSync`, `GetFirstDescendantComponent`, ...) are utils-file extension methods, not engine APIs. Their definitions are included so any project can adopt them.
- The generated `*.digest.verse` files for your build are the source of truth for exact API signatures.
