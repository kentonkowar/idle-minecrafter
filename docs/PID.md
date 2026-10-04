# Idle Minecrafter — Project Initiation Document

2026-09-30 · smithy

## Purpose

Idle Minecrafter is a Minecraft Java Edition mod that grants a player's farms, quarries and crafting setups their output for the time the player was offline. The player can log off and shut the computer down overnight, then return to the resources they need for the next day.

Today a player who wants overnight progress must stay logged in: the computer runs all night, the machine is loud and draws power, and the idle character can die. Idle games solved this long ago by calculating offline progress when the player returns. This mod brings that model into Minecraft.

This document fixes the scope, the success factors, and how the project is managed and documented before any code is written. The package namespace is a placeholder, `com.smithy.idleminecrafter`, until the final one is chosen; "smithy" also stands in as the author name throughout.

## Scope

Version 1.0 covers offline progress for three kinds of player-built setups (farms, quarries, crafting) on one Minecraft version and one mod loader. Anything beyond that is a later release or a change request.

### In scope (v1.0)

- **Offline production.** When the player returns, the mod calculates what each registered setup would have produced during the absence and delivers it.
- **Three setup types.** Mob farms, quarries (block mining), and crafting (turning inputs into outputs).
- **Player-chosen setups.** The player decides which setups count; the mod never guesses.
- **Production based on real behaviour.** Offline rates come from what a setup actually does while the player is present.
- **Inputs and limits respected.** Offline crafting consumes inputs; output stops when inputs run out or storage fills.
- **A catch-up summary.** On return, the player sees what was produced and where it went.
- **Configuration.** Single-player worlds can set an offline-time cap (default 2 hours) and toggle features on or off.
- **Single-player worlds only** (multiplayer not in v1.0).

### Out of scope (v1.0)

- Simulating the world while offline: mobs, crop growth in unregistered chunks, weather and redstone stay frozen.
- Resources from nothing: no idle currency, prestige or upgrades detached from the player's builds.
- Keeping chunks loaded or running a background process while the computer is off.
- Automating the player's character (AFK bots, auto-clickers).
- Machines from other mods (such as Create or Mekanism).
- Bedrock Edition, and multiplayer servers. Only Java Edition single-player worlds.

### Assumptions

- The player's computer may be fully off during the absence; all catch-up work happens when the world next loads.
- The owner is the sole decision-maker, with Claude as the implementation assistant.
- Players accept that offline output is an estimate of online output, not a tick-by-tick replay.
- Each player runs their own single-player world; no shared servers in v1.0.

### Constraints

- Minecraft Java Edition 26.2 on Fabric loader only.
- The mod must not corrupt saves, and a world must still load after the mod is removed.
- Catch-up must finish at world load without a noticeable freeze.

## Success factors

The project succeeds when every essential holds and the goals are met at release. Objectives say why the mod exists, goals say how we measure it, and essentials are pass/fail: missing any one means v1.0 does not ship.

### Objectives

1. Let players progress overnight without leaving the game or the computer running.
2. Keep offline progress fair, so it mirrors what the player's builds would really have done.
3. Make the mod feel native to Minecraft.
4. Build a codebase that can be ported to later Minecraft versions without a rewrite.

### Goals

The targets below are proposed starting points; adjust them before the Design phase closes.

| Goal | Measure | Target |
| --- | --- | --- |
| Offline output matches online output | Same setup run online and offline for the same time, output compared | Within 5% for every v1 setup type |
| Catch-up is fast | Time added to world load for 12 hours offline with 20 registered setups | Under 1 second |
| All three setup types work | Farm, quarry and crafting setups each produce offline output | 3 of 3 |
| Setup is easy to learn | New player registers a first setup without reading external docs | Under 5 minutes in playtests |
| Stable in play | Crashes or save errors reported during the beta | 0 open at release |
| Released | Published on the chosen platforms | v1.0 at the Release gate |

### Essentials

- [ ] No save corruption or item loss, and a world still loads after the mod is removed.
- [ ] Offline output never exceeds what the same setup would produce online in the same time.
- [ ] No duplication exploits: logging in and out repeatedly never creates extra items.
- [ ] Works with the computer fully off; nothing runs in the background.
- [ ] Calculations run on the server (or the integrated server in single-player), never trusted from the client.
- [ ] Server owners can cap offline time or switch the mod's features off.

## Management

The project runs in six phases separated by five gates, and smithy approves each gate before the next phase starts. Scope changes go through a written decision, never straight into code.

### Phases and gates

No dates are set yet. The Prototype gate is the first real test of the fairness essential.

| Phase | Work | Gate to exit |
| --- | --- | --- |
| 1. Initiation | Agree this document and close the open decisions | Document approved, open decisions recorded |
| 2. Design | Design docs: registering setups, measuring rates, catch-up, delivery | Design docs reviewed and approved |
| 3. Prototype | One farm type producing offline output end to end | Prototype passes the fairness test |
| 4. Build | All three setup types, configuration and the catch-up summary | Feature complete, all essentials pass automated tests |
| 5. Beta | Playtesting, fixing, and measuring against the goals | Release gate: goals met, no open crash or save bugs |
| 6. Release | Publish v1.0 with the player guide and changelog | — |

### Roles

| Role | Who | Responsibilities |
| --- | --- | --- |
| Owner | smithy | Sets scope, approves gates and decisions, reviews and merges all changes |
| Implementation assistant | Claude | Drafts designs, writes code and tests, keeps documentation current, flags risks |
| Playtesters | Recruited for beta | Report bugs and confusion against the goals |

### Working practices

- **Backlog.** Work items are tracked as issues grouped into one milestone per phase, on the host chosen in Open decisions.
- **Branches.** `main` always builds and runs; each change lands on a short-lived branch and is merged after smithy's review.
- **Versioning.** Semantic versioning with the Minecraft version appended, for example `1.0.0+mc<version>`.
- **Testing.** Automated in-game tests cover the fairness and duplication essentials; every gate re-runs them.
- **Change control.** A change to scope, goals or essentials is written up as a decision record, approved by smithy, then reflected in this document. Design details inside the agreed scope are settled in design docs.

### Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Offline output can be gamed (duplication, inflated rates) | Breaks fairness and multiplayer economies | Rates only from observed output, server-side calculation, duplication tests at every gate |
| Minecraft or loader updates break the mod | Rework and delayed release | Pin one version for v1, keep game-specific code behind a thin layer |
| Long absences make catch-up slow | Freeze at world load | Offline-time cap, calculate totals in closed form rather than replaying ticks |
| Offline output differs noticeably from online | Players distrust the mod | Fairness goal measured in automated tests, results shown in the catch-up summary |
| Scope creep (more machines, other mods) | v1 never ships | Out-of-scope list above, change control for anything new |

## Documentation

All project documentation lives in the repository as Markdown under `docs/`, next to the code it describes, and is updated in the same change as that code.

| Document | Purpose | Location | Updated when |
| --- | --- | --- | --- |
| Project initiation document | Scope, success factors, management | `docs/PID.md` | A decision changes scope, goals or essentials |
| README | Player-facing pitch, install steps, links | `README.md` | Features or install steps change |
| Design docs | How each feature works and why | `docs/design/<feature>.md` | Before a feature is built, and when its design changes |
| Decision records | One numbered record per significant decision | `docs/decisions/NNNN-<title>.md` | Each decision; records are never edited, only superseded |
| Changelog | What changed in each release | `CHANGELOG.md` | Every merged change, grouped under the next version |
| Assistant guide | Build commands, conventions and structure for Claude | `CLAUDE.md` | Tooling or conventions change |
| Code comments | Intent of public classes and non-obvious logic | In source | With the code |
| Player guide | How to register setups, config options, FAQ | `docs/guide/` | Before beta, then with each release |

### Conventions

- Each design doc states the problem, the chosen approach, alternatives considered, and how it meets the essentials.
- Decision records follow a fixed template: context, decision, consequences, status.
- The changelog follows the Keep a Changelog format (Added, Changed, Fixed, Removed).
- Every document names smithy as author until the final namespace and name are chosen.

## Open decisions

These must be settled before the Initiation gate closes; each becomes one of the first decision records. Closed decisions are recorded below.

**Still open:**

- [ ] Final package namespace and mod id (placeholder: `com.smithy.idleminecrafter`, mod id `idleminecrafter`).
- [ ] License, and release platforms (Modrinth, CurseForge).

**Closed (recorded as decision records to follow):**

- Mod loader: Fabric
- Target Minecraft version for v1.0: 26.2 Java Edition
- Scope: Single-player worlds only in v1.0
- Default offline-time cap: 2 hours (configurable by world owner)
- Machines from other mods: Not in scope for v1.0
- Repository host: To be determined and set up at Initiation gate
