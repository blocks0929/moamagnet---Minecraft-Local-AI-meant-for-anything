# Phase 3 — Verification checklist (Scripted goal executor)

Phase 3 adds a **scripted goal executor** (Python state machines) plus the mod-side
protocol growth that feeds it: an in-game `/goal` command, an on-demand `findBlocks`
world query, and an optional `quitWorld`. Goals: `goto`, `mine`, `biome`, `findmob`,
with **mode awareness** (Hardcore/Survival health aborts, Creative none).

Done when every box below is checked. Split into **offline** (no Minecraft, already run)
and **in-game** (real 1.21.11 client).

## A. Automated / offline (no Minecraft) — verified in this environment

- [x] Everything compiles (`python -m py_compile python/*.py tools/*.py`; mod builds to
      `moamagent-0.3.0.jar`).
- [x] **Phase 1+2 protocol test still passes**: `python tools/test_protocol.py` → `ALL PASSED`.
- [x] **Phase 3 goal test passes**: `python tools/test_goals.py` → `ALL PASSED`
  - `goto (8,8)` → player **arrives** (<2 m off);
  - `mine minecraft:oak_log ×3` → three logs are **located via `findBlocks`, approached,
    and broken** (the mock simulates the breaks; the executor confirms each via re-query).

Re-run any time:

```bash
python tools/test_protocol.py     # Phase 1+2
python tools/test_goals.py        # Phase 3   (both expect ALL PASSED)
```

## B. Build + deploy

- [x] Built with Gradle 9.4.0 + JDK 21; `moamagent-0.3.0.jar` deployed to the
      `Minecraft AI bot` profile (0.2.0 removed). No new mapping issues.

## C. In-game acceptance (real 1.21.11 Fabric client) — run on your machine

Launch the profile, open a world. In a terminal:

```bash
cd python
python agent_client.py agent      # long-running runner; listens for /goal
```

Then drive it from **in-game chat** (or use the one-shot CLI form shown after each):

- [ ] **`/goal`** with no args prints usage in chat.
- [ ] **goto** — stand somewhere open, run `/goal goto <x> <z>` (a spot ~20–40 blocks away).
      Character **walks/sprints there**, auto-jumps over small obstacles, and stops on arrival.
      HUD shows `goto (x,z)` + distance countdown; dashboard progress bar climbs to 1.0.
      *(One-shot equivalent: `python agent_client.py goal goto --x 100 --z -40`.)*
- [ ] **mine** — near some trees, `/goal mine 5 minecraft:oak_log`. Agent locates logs,
      walks to each, aims, breaks it, and reports `mined k/5` until done.
      *(One-shot: `python agent_client.py goal mine --block minecraft:oak_log --count 5`.)*
- [ ] **biome** — `/goal biome desert` (pick a biome not where you are). Agent wanders in a
      rotating search; completes when `world.biome` matches. Give it room / a reachable target.
- [ ] **findmob** — with a cow/animal nearby, `/goal findmob cow`. Agent approaches to ~2 blocks.
- [ ] **cancel** — during any goal, `/goal cancel` (or the dashboard **Cancel goal** button)
      stops it; HUD clears.
- [ ] **dashboard integration** — while a goal runs, press Right Shift: the chart is the real
      progress metric, the panel shows `approach: scripted · <mode>`. The **Stop after 60s** /
      **Stop at 25 GPU·s** buttons actually end the run.
- [ ] **mode awareness** — in a Hardcore world, if you take damage to ≤6 HP mid-goal, the agent
      **aborts** with a safety message (respects permadeath). In Creative, no health abort.
- [ ] **exit-on-complete (optional)** — run `python agent_client.py goal goto --x .. --z .. --exit-on-complete`;
      on arrival the client sends `quitWorld` and the game **saves + returns to the title screen**.

## Notes, honest limitations, and what's deferred

- **Navigation is reactive**, not full voxel A*. It handles open/normal terrain, small steps,
  and getting unstuck, but can struggle in caves, tall walls, or near cliffs (it may walk off
  edges — the Hardcore health-guard is the safety net). Real pathfinding is a later hardening step.
- **`findBlocks`** is bounded (radius ≤16, ≤64 results, ±8 Y) and only runs when a goal needs it,
  so it stays off the per-tick path (per the perf guidance in CLAUDE.md §7).
- **Deferred to Phase 4** (not stubbed/faked here): `/train` headless bots, **crafting**, mob
  **breeding**, the **LLM planner** that turns free-text goals into these structured ones, and the
  narrow RL experiment. Phase 3 goals are structured args; `mine`/`goto`/`biome`/`findmob` only.
- **One controller connection at a time** — run *either* `agent` *or* `dashboard`/`watch`, not both.

## Phase 3.5 — Survival hardening + awareness + reward (v0.3.1)

Added after the first in-game run showed the bot mindlessly sprinting to a target and dying
(drown → fall → wall-stuck → zombie). This is the scripted precursor to the Phase 4 reward/RL layer.

Offline-verified (`python tools/test_survival.py` → `ALL PASSED`, plus Phase 1/2/3 regressions):

- [x] **Hazard-aware navigation** — the mod exports a cheap per-tick `nav` probe (wallAhead,
      edgeAhead, dropDepth, waterAhead, lavaAhead, facing) + player water/lava/fire flags. The
      navigator **only steps forward in a heading whose look-ahead was safe**, so it won't walk off
      cliffs or into lava/water; it rotates to route around, or reports "stuck" rather than dying.
      *(Unit test asserts it never sends forward>0 while a hazard is ahead.)*
- [x] **Drowning avoidance** — when submerged or low on air it stops the goal and **surfaces**
      (looks up, swims/jumps up) until air recovers.
- [x] **Threat response** — nearest hostile within ~6 blocks: **fight** (face + attack) if healthy,
      **flee** if health is low; mode-aware (skipped in Creative).
- [x] **Reward ledger** — punishes damage (−per half-heart) and death (−50, with cause: drowning/
      fall/lava/fire/mob), rewards progress + completion (+100). Streamed as `score`
      {reward, deaths, damage} → shown on the **HUD** (green/red) and the dashboard **SCORE / SAFETY**
      panel. This is the exact reward shaping the Phase 4 RL layer will optimise.
- [x] **Autocomplete** — `/goal mine <block>`, `/goal biome <id>`, `/goal findmob <id>` now suggest
      real registry ids (blocks, biomes, entities) as you type.
- [x] **Awareness query `scanArea`** — on-demand compact local block map (non-air blocks within a
      bounded radius/height, each tagged solid / fluid / breakable). Lets the agent know *what is
      around it and whether it can break it* without bloating the per-tick state.

In-game to confirm (needs you): point `/goal goto` across water/near a cliff — the bot should route
around or stop instead of drowning/falling; near a zombie it should fight or flee; the HUD score
should drop on damage. **Honest limit:** still reactive, not full A* — a bot boxed in by hazards will
safely give up ("stuck") rather than magically pathfind out. That's the Phase 4/5 pathfinding upgrade.

## Protocol additions (Phase 3)

Controller → mod:
```json
{"type":"query","query":"findBlocks","block":"minecraft:oak_log","radius":16,"max":24,"id":123}
{"type":"quitWorld"}
```
Mod → controller:
```json
{"event":"queryResult","query":"findBlocks","id":123,"matches":[{"x":..,"y":..,"z":..,"distance":..}]}
{"event":"goalRequest","goal":{"kind":"mine","block":"minecraft:oak_log","count":10}}
{"event":"goalCancel"}
```
