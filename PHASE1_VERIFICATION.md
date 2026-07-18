# Phase 1 — Verification checklist

Phase 1 is "done" when every box below is checked. Tests are split into what can be verified
**without Minecraft** (automated, already run) and what needs a **real 1.21.11 client**.

## A. Automated / offline (no Minecraft) — verified in this environment

- [x] `fabric.mod.json` is valid JSON and declares `main` + `client` entrypoints.
- [x] Python controller and tools compile (`python -m py_compile`).
- [x] **Wire-protocol test passes**: `python tools/test_protocol.py` → `ALL PASSED`
      (state received, movement responds to `move`, `stop` halts, `look` applies yaw).
- [x] Version matrix pinned to values that exist on `meta.fabricmc.net`
      (MC 1.21.11, yarn 1.21.11+build.5, loader 0.19.2, fabric-api 0.141.4+1.21.11).

Re-run any time:

```bash
python tools/test_protocol.py      # expect exit code 0
```

## B. Build verification (needs JDK 21 + internet) — run on your machine

- [ ] `gradle wrapper --gradle-version 8.12` succeeds (or open in IntelliJ).
- [ ] `./gradlew build` completes; `build/libs/moamagent-0.1.0.jar` exists.
- [ ] No unresolved-mapping compile errors (see *Mapping fixes* below if any appear).

## C. In-game acceptance (needs a real 1.21.11 Fabric client) — run on your machine

- [ ] `./gradlew runClient` launches; log shows
      `[MoamAgent] Bridge listening on 127.0.0.1:25599`.
- [ ] Open a single-player world. `/agent status` reports the bridge port.
- [ ] `python agent_client.py watch` prints live state; values change as you move manually.
- [ ] `python agent_client.py raw --count 1` shows correct `gameMode`, `hardcore`,
      `biome`, `targetBlock` (look at a block), and nearby `entities` (spawn a mob).
- [ ] `python agent_client.py demo` → **player visibly sprints forward** and reports
      "distance travelled > 1 block" (**core acceptance test: the AI moved your character**).
- [ ] Aim at a tree/stone, `python agent_client.py mine --seconds 5` → **block breaks**.
- [ ] Kill the Python process → mod auto-releases keys (player stops; no stuck movement).

## D. Game-mode data sanity (foundation for Phase 3 awareness)

- [ ] Creative world → state shows `"gameMode":"creative"`.
- [ ] Hardcore world → state shows `"hardcore":true`.
- [ ] Difficulty change reflected in `world.difficulty`.

---

## Mapping fixes (only if step B reports errors)

Yarn method names occasionally change between versions. If `./gradlew build` fails on one of
these, apply the noted alternative in `StateSerializer.java` / `CommandExecutor.java`:

| Symbol used | If it fails, try |
|-------------|------------------|
| `player.getInventory().getSelectedSlot()` | field `player.getInventory().selectedSlot` |
| `world.getLevelProperties().isHardcore()` | `world.getLevelProperties().isHardcore()` is stable; else `mc.player.isHardcore()` on newer yarn |
| `mc.interactionManager.getCurrentGameMode()` | stable on `ClientPlayerInteractionManager` |
| `biome.getKey()` returns `Optional<RegistryKey>` | if signature differs, use `world.getBiome(pos).getIdAsString()` |

Paste any build error back and I'll pin the exact mapping for `1.21.11+build.5`.

---

## Known Phase-1 limitations (by design; addressed later)

- Controls the **local player only**; `/train` bots are Phase 4.
- Block scan is just the **crosshair target**; full nearby-block volume is Phase 3.
- No pathfinding/goals/HUD/learning yet.
- Single controller connection at a time.
