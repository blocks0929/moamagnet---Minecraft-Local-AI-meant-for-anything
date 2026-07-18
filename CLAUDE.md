# CLAUDE.md — Mounts of Mayhem Agent (handoff to Claude Code)

You are taking over a multi-phase project from a prior assistant that could not run a
terminal on this machine. This file is your source of truth: the decisions already made,
the current state, what to do **right now**, and the full roadmap. Read it fully before acting.

---

## 0) One-paragraph summary

Build a **Fabric mod for Minecraft Java 1.21.11** ("Mounts of Mayhem") plus a **Python
controller** that lets an AI observe and play the game. The mod exposes game state and accepts
control commands over a loopback TCP socket; the Python side decides actions. **Phase 1 (the
socket bridge + state export + manual control) is written and unit-verified against a mock, but
has NOT been compiled with Gradle yet** — that's your first job, because it needs JDK 21 +
internet, which the prior assistant lacked.

---

## 1) Decisions already locked with the user (do not re-litigate)

- **Target:** Minecraft Java **1.21.11**. Confirmed supported by Fabric (checked meta.fabricmc.net).
- **AI architecture:** **Hybrid** — a local LLM planner decomposes goals into scripted skills,
  with a **narrow RL layer** added later for one or two well-shaped sub-tasks. NOT a from-scratch
  general RL agent (that's research-grade; see feasibility note §7).
- **Planner compute:** **local GPU, 8–12 GB VRAM** → plan around a 7–8B quantized model
  (Llama 3.1 8B / Qwen2.5 7B) via **Ollama** (already installed on this machine).
- **Deployment:** single-player first, but keep it **multiplayer/dedicated-server capable**.
- **`/train` bots:** **lightweight/headless** fake players are acceptable (favor throughput).
- **"Cost"** in HUD/dashboard = **local GPU-seconds + energy estimate** (no API $ unless a cloud
  model is later added; leave a hook for token cost).

## 2) Verified version matrix (in `gradle.properties`)

| Component      | Version           |
|----------------|-------------------|
| Minecraft      | `1.21.11`         |
| Yarn mappings  | `1.21.11+build.5` |
| Fabric Loader  | `0.19.2`          |
| Fabric API     | `0.141.4+1.21.11` |
| Loom plugin    | `1.16.3`          |
| Gradle wrapper | `9.4.0`           |
| Java           | **21** (required) |

> **Build note (corrected):** Loom `1.16.3` requires **Gradle ≥ 9.4.0** — the original
> `8.12` here was wrong and fails variant resolution (`org.gradle.plugin.api-version 9.4.0`).
> Phases 1 & 2 were built with Gradle **9.4.0** + JDK 21. One mapping drift was fixed for
> `1.21.11+build.5`: `GameMode.getName()` → `GameMode.getId()` in `StateSerializer.java`.
> `KeyBinding.Category` is now a **record** (use `KeyBinding.Category.MISC`), not a String.

If any of these fail to resolve, re-query `https://meta.fabricmc.net/v2/versions/` and
`https://maven.fabricmc.net/net/fabricmc/fabric-api/fabric-api/maven-metadata.xml` and pick the
newest build tagged `+1.21.11`.

---

## 3) Current state of the repo

**IMPORTANT cleanup first:** there is a stray duplicate folder `moam-agent-phase1/` nested
INSIDE the project root (a copy artifact the prior assistant couldn't delete due to a read-only
mount). The **top-level** files are the real project. Delete the nested `./moam-agent-phase1/`
duplicate before building.

Project layout (top level = project root):

```
build.gradle, settings.gradle, gradle.properties   # Loom build, versions pinned
gradle/wrapper/gradle-wrapper.properties           # wrapper jar is NOT present — generate it
src/main/java/com/moamagent/MoamAgentMod.java       # common entrypoint (calls AgentHome.ensure)
src/main/java/com/moamagent/AgentHome.java          # self-extracts /agent -> moamagent-AI/ (v0.3.2)
src/main/resources/fabric.mod.json                  # entrypoints: main + client
src/main/resources/assets/moamagent/icon.png
src/main/resources/agent/                           # bundled into jar; self-extracted at runtime
    manifest.txt, run.bat, run.sh, README.md        #   static resources (+ python/*.py copied by build)
src/client/java/com/moamagent/client/
    MoamAgentClient.java     # client entry: starts bridge, ticks, /agent command
    AgentBridge.java         # loopback TCP server, newline-JSON, 1 client
    StateSerializer.java     # builds the per-tick state JSON (Gson)
    CommandExecutor.java     # applies commands as LEGAL keybinding input
python/agent_client.py       # controller: AgentConnection + watch/raw/demo/mine/agent/goal CLIs
python/goals.py              # scripted goal state machines + survival + reward ledger
python/agent_home.py         # locates moamagent-AI/, writes logs/ + data/history.jsonl
python/trainer.py            # Phase 4 narrow RL: tabular Q-learning reach-target + weights I/O
python/deep_trainer.py       # Phase 4 deep RL: PyTorch DQN (CUDA) — same interface as trainer.py
python/live_train.py         # LIVE RL: RL drives the REAL character to waypoints + learns (v0.6.0)
python/llm_planner.py        # local LLM planner (Ollama REST) + reward + exemplar learning + bake (v0.7.0)
python/plan_runner.py        # executes an LLM plan via scripted skills, re-plans on failure, scores it
python/setup_env.py          # one-time self-healing deep-RL dep install, gated by setup.json
tools/mock_mod_server.py     # fake bridge (toy physics) for testing without Minecraft
tools/test_protocol.py       # automated protocol test — passes (ALL PASSED)
README.md, PHASE1_VERIFICATION.md, LICENSE
```

**Design rationale you must preserve:** control runs **client-side** and drives the local player
by holding **vanilla keybindings** + setting look angles (see `CommandExecutor`), NOT teleport or
velocity. This keeps movement legal (critical for the future speedrun goal) and avoids fighting
client/server reconciliation. Server-side bot control for `/train` comes in Phase 4.

**Protocol (Phase 1), newline-delimited JSON both ways:**
- Mod → controller: one state object per client tick (player pos/look/health/food/air/xp/velocity,
  world dimension/difficulty/hardcore/gameMode/biome, crosshair targetBlock, hotbar inventory,
  bounded nearby-entity list).
- Controller → mod, one per line: `look{yaw,pitch}`, `move{forward,strafe,jump,sneak,sprint}`,
  `mine{hold}`, `use{hold}`, `stop`. Inputs persist until changed; mod releases keys on disconnect.
- Default port `127.0.0.1:25599`.

---

## 4) DO THIS NOW — finish Phase 1 verification

Run from the project root. Use Java 21 (IntelliJ IDEA Community 2025.2.4 is installed and bundles
a JBR 21; you can also `runClient` straight from it).

```bash
# 0. remove the stray nested duplicate
rm -rf ./moam-agent-phase1

# 1. generate the wrapper jar (needs a system Gradle, OR let IntelliJ import handle it)
gradle wrapper --gradle-version 8.12

# 2. compile
./gradlew build            # Windows cmd: gradlew.bat build
```

**Expected failure mode:** Yarn method names occasionally drift. If compilation fails, the likely
spots and fixes are in `PHASE1_VERIFICATION.md` §"Mapping fixes". The usual suspects:
- `PlayerInventory#getSelectedSlot()` — if absent, use the `selectedSlot` field.
- `ClientWorld#getLevelProperties().isHardcore()` — else `mc.player.isHardcore()`.
- `ClientPlayerInteractionManager#getCurrentGameMode()`.
- `world.getBiome(pos).getKey()` returning `Optional<RegistryKey<Biome>>` — else use
  `getIdAsString()`.
Fix against the actual `1.21.11+build.5` mappings (you have them locally after the first build;
browse `~/.gradle` or use IntelliJ's "Go to Declaration").

**Then the in-game acceptance tests** (full checklist in `PHASE1_VERIFICATION.md`):

```bash
./gradlew runClient        # log must show: [MoamAgent] Bridge listening on 127.0.0.1:25599
# In game: open a world, /agent status
cd python
python agent_client.py watch          # live state should track your manual movement
python agent_client.py demo --seconds 3   # CORE TEST: character sprints forward, reports >1 block
python agent_client.py mine --seconds 5   # aim at a block first; it should break
```

Phase 1 is done when §B, §C, §D of `PHASE1_VERIFICATION.md` all pass. The offline protocol test
(`python tools/test_protocol.py`) already passes and needs no Minecraft.

---

## 5) Roadmap — get user sign-off on scope before each phase

The user explicitly wants **confirmation of scope before writing each phase's code**. Ask, then build.

- **Phase 2 — HUD + Dashboard. ✅ IMPLEMENTED (v0.2.0).** See `PHASE2_VERIFICATION.md`.
  Adds `AgentStatus`/`AgentHud`/`DashboardScreen`, Right-Shift keybind, and the
  `agentStatus`(in) / `event`-control(out) protocol. Data is synthetic until Phases 3–4;
  drive it with `python agent_client.py dashboard`. In-game acceptance (§C) still pending a
  human at the client.
  HUD (always visible during an active goal/run): top-left current action/goal text; top-right
  elapsed session time; a cost estimate (local GPU-seconds + energy). Dashboard toggled by
  **Right Shift**: training metrics over time (charts/numeric), model/approach + hyperparameters +
  hardware in use, cumulative cost breakdown, and controls to stop after a time OR cost limit and
  to view/cancel the current goal. Use Fabric `HudRenderCallback` for the HUD and a custom
  `Screen` for the dashboard. Cost meter reads GPU-seconds from the Python side over the socket.

- **Phase 3 — Scripted goal executor + commands + mode awareness. ✅ IMPLEMENTED (v0.3.0).**
  See `PHASE3_VERIFICATION.md`. Shipped: in-game `/goal goto|mine|biome|findmob|cancel`
  (emits `goalRequest`/`goalCancel` events); Python `goals.py` `GoalRunner` state machines
  driven by `python agent_client.py agent` (or one-shot `agent_client.py goal <kind> ...`);
  on-demand `findBlocks` query + optional `quitWorld`; mode-aware health aborts (Hardcore/
  Survival/Creative). Verified offline with `tools/test_goals.py`. **Deferred to Phase 4
  (not stubbed):** `/train` bots, crafting, breeding, the LLM planner, and RL. Navigation is
  reactive (seek + auto-jump + unstick), NOT full A* — noted as a later hardening step.
  **Phase 3.5 hardening (v0.3.1):** survival layer so the bot stops dying — mod exports a per-tick
  `nav` hazard probe + water/lava/fire flags; `goals.py` refuses to step off cliffs/into lava/water,
  surfaces when submerged, and fights/flees hostiles (mode/health aware). Added a scripted
  `RewardLedger` (punish death/damage, reward progress/completion) shown on HUD + dashboard as
  `score` — the reward shaping Phase 4 RL will optimise. Added `/goal` registry autocomplete and an
  on-demand `scanArea` awareness query (bounded local block map: solid/fluid/breakable). Verified
  with `tools/test_survival.py`. See memory: `game-awareness`, `phase4-reward-learning`.
  **Packaging (v0.3.2) — the jar is the only file needed.** The Python controller is bundled
  into the jar (build.gradle `processResources` copies `python/{agent_client,goals,agent_home}.py`
  + `requirements.txt` into `/agent`; run scripts + README + `manifest.txt` are static resources
  under `src/main/resources/agent/`). `AgentHome.java` (common init, called from
  `MoamAgentMod.onInitialize`) self-extracts `/agent/*` into `<gameDir>/moamagent-AI/` and creates
  `logs/` + `data/`. A `version.txt` stamp gates re-extraction: extract only when missing or the
  mod version changed (never clobbers user `logs/`/`data/`; same version = untouched). Ships
  `run.bat`/`run.sh` (turnkey `python agent_client.py serve`) + a folder README. Auto-launch is
  **ON by default as of v0.4.0** (opt out via `moamagent-AI/autostart.disabled` or
  `-Dmoamagent.autostart=false`; best-effort, never hard-fails without Python). `agent_home.py`
  writes session logs + `data/history.jsonl` **only** when running from an extracted home
  (`version.txt` present), so offline tests from `python/` stay clean. Socket/protocol unchanged.
  Original Phase-3 wish list (for reference):
  `/goal <description>` (structured goals: goto coord, mine block type N, craft item X, find
  biome/structure Y, find/breed mob Z), `/train <n>`, and world-mode awareness (Creative skips
  gathering; Hardcore avoids risk + respects permadeath; Survival normal). On goal completion,
  close/exit the world. Implement Baritone-style pathfinding + state machines. This is where the
  socket protocol grows (add: full nearby-block volume, pathfind requests, craft/inventory ops).

- **Phase 4 — Learning loop + always-on AI + dashboard on/off. ✅ PARTIALLY IMPLEMENTED (v0.4.0).**
  See `PHASE4_VERIFICATION.md`. Shipped:
  * **Always-on controller.** The mod now **auto-launches by default** (opt out via
    `moamagent-AI/autostart.disabled` or `-Dmoamagent.autostart=false`) and runs the new
    `agent_client.py serve` — a resilient supervisor that reconnects forever (survives no-world /
    disconnect) so the AI is "ready at all times". `run.bat`/`run.sh` also use `serve` now.
  * **Dashboard on/off (Right Shift).** Two new toggle buttons — **AI: ON/OFF** and
    **Trainer: ON/OFF** — send `{"event":"control","action":"setAi|setTrain","value":0|1}`; the
    controller obeys and echoes state back in `agentStatus.ai{enabled,training,steps,best,successRate}`,
    which `AgentStatus.java` parses and the buttons/`AI / LEARNING` panel reflect.
  * **Two RL backends, auto-selected.** `python/trainer.py` = **stdlib-only tabular Q-learning**
    (zero-install fallback); `python/deep_trainer.py` = **PyTorch DQN** (MLP over continuous
    features, runs on CUDA) — same interface (`train_slice`/`metrics`/`info`/`save`/`load`), picked
    automatically when torch is present. Both learn the shaped reach-target task in an internal sim
    (heading + forward motion, hazard discs), stream a live learning curve to the dashboard, and are
    labelled (`learned (deep RL · DQN)` / `learned (Q-learning)`). Trains in the background whenever
    AI is on and idle; the dashboard shows the active backend + setup state.
  * **One-time self-healing setup** (`python/setup_env.py`). On first `serve` it installs the deep
    deps (torch/numpy/gymnasium) **into the current interpreter** (reuses an existing CUDA torch),
    gated by `moamagent-AI/setup.json`: if the marker is missing/not-done OR any dep fails to
    import, it (re)installs and re-verifies, then writes `done`. Best-effort — if pip/network fail
    it marks `done:false` and the controller falls back to the tabular backend (retries next launch).
    Skip with `--no-setup` or `--backend tabular`.
  * **Persisted weights.** Tabular → `data/models/reach_q.json`; deep → `data/models/reach_dqn.pt`
    (both atomic-replace, reloaded on start to resume). Verified with `tools/test_trainer.py`,
    `tools/test_deep.py`, and an end-to-end extract-from-jar + `serve`-against-mock run.
  **Phase 4.5 — live control + planner + collect (v0.6.0).**
  * **Live RL on the REAL character** (`python/live_train.py`, `LiveReach`). Closes the
    "always-on learning but never trained because it isn't controlling anything" gap: when a world
    is open the RL policy DRIVES the player to auto-generated waypoints (chained — no teleport/reset)
    and learns from the real (state,action,reward) transitions, using the SAME 4-feature obs / 5
    actions as the sim so the model transfers. A hazard mask vetoes forward steps into cliffs/lava/
    water. The supervisor prefers live when a world is open, falls back to sim on the title screen;
    dashboard shows `Mode: LIVE (real play)` vs `sim`. Both trainers gained `act_features` /
    `learn_transition`. Verified against the mock (`tools/test_live.py`: RL drives the player to
    waypoints; deep adapter works).
  * **`/train` [stop]** — command that turns the live learner loose on your character (enables AI +
    live training); parallel headless bots still deferred.
  * **`/goal collect <n> <item>`** — planner skill: item→source-block map, then search-outward →
    approach → harvest, counting the item in the *whole* inventory (new `inventory.counts` in the
    state), adapting live. `describe_goal`/`plan_for` log the decomposed plan.
  * **Search fix** — the old `_explore` rotated in place (73°/4s toward an 8-block point) so
    biome/mob/block search "ran in circles". Replaced with an **expanding outward star** (real legs,
    growing radius, widening findBlocks queries) that covers new ground. Verified it ranges >20
    blocks out vs the old ~8-block wander.
  **Phase 4.6 — local LLM planner (v0.7.0).** `python/llm_planner.py` + `plan_runner.py`.
  * **Planner:** talks to a **local Ollama** over its REST API (stdlib `urllib`, no pip dep). System
    prompt teaches a strict JSON plan schema `{goal, estimatedSeconds, strategy, steps:[{skill,args,
    note}]}`; robust parser normalises the loose args small models emit and drops un-runnable steps
    (e.g. a `goto` with a place-name instead of coords). Executable skills = goto/mine/collect/biome/
    findmob; higher-level steps (craft/portal/kill_boss/…) are kept but marked not-executable, not
    faked. Model auto-resolved: `MOAM_LLM_MODEL` → baked `moamagent-planner` → first of
    llama3.1:8b/qwen2.5:7b/phi3:mini/dolphin-mistral. Graceful **fallback planner** when Ollama is
    down. Tested for real against `phi3:mini` (`tools/test_planner.py`).
  * **Executor + live re-plan:** `PlanRunner` runs each executable step via `GoalRunner`; on a failed
    step it calls `LLMPlanner.replan()` with the live state and splices in the fix (≤2 replans).
  * **Reward + learning "as we go":** `score_plan()` = success + **faster→more** + **ETA-accuracy**
    (closer predicted-vs-actual → more), exactly as asked. Every finished plan is scored and stored;
    `ExemplarStore` keeps the top-reward plans and feeds them back as **few-shot exemplars** (online
    in-context improvement). Durable learning: `bake()` distils the best exemplars into a local model
    via an Ollama **Modelfile** (`ollama create moamagent-planner`, auto every 10 plans); `export_sft()`
    writes a JSONL for an optional heavier LoRA/SFT pass. Data in `moamagent-AI/data/planner/`.
  * **Wiring:** in-game `/plan <text>` (free-form) and `/goal collect` both route through the planner
    when Ollama is up (else scripted `collect`); Python `agent_client.py plan "<text>"`; `serve
    --no-llm` opts out.
  **Still deferred (NOT stubbed):** `/train <n>` server-side headless bots; **full voxel A\***
  pathfinding; executing the high-level steps the LLM can already *emit* (crafting, nether/end
  traversal, boss fights, building) — those need new skills; and true gradient LoRA fine-tuning
  (we do exemplar-distillation via Modelfile now + export the SFT dataset for later).
  Original Phase-4 wish list (for reference): `/train <n>` fake-player bots for parallel practice;
  LLM planner decomposing `/goal` text into skills; a narrow RL experiment so the dashboard's
  "improvement over time" is real, not theater; label scripted vs learned in the UI.

- **Phase 5 — Speedrun goal.**
  `/goal speedrun` as a deterministic route executor using only legal player actions (the
  keybinding-input design already enforces this). Add movement-legality checks / an anti-cheat-style
  self-audit. Be clear with the user this is a scripted route, not a learned speedrunner.

## 6) Conventions

- Mod id `moamagent`, package `com.moamagent`, Java 21, MIT license.
- Keep client-only code in `src/client` (split source sets are configured in `build.gradle`).
- Gson ships with Minecraft — don't add a Gson dependency.
- Prefer public client APIs + Fabric events; add mixins only when an event/API doesn't exist.
- Never expose the control socket beyond `127.0.0.1`.
- Python controller stays stdlib-only until Phase 4 (then add ollama / torch / gymnasium to
  `python/requirements.txt`, which already lists them as "later").

## 7) Feasibility & compliance — keep the user honestly informed

- Open-ended goals ("beat the game", arbitrary redstone, any% speedrun from one text prompt) are
  **research-grade** (MineRL/VPT/Voyager). Deliver the **LLM-planner-over-scripted-skills** path
  with a **narrow** RL demo; do not promise a from-scratch agent that masters everything. Always
  label scripted vs learned in the dashboard.
- **EULA/distribution:** ship only your own MIT mod jar; never redistribute Minecraft assets or the
  deobfuscated/remapped jars Loom produces. Automating your own local game is fine; on servers you
  don't own, get the operator's permission (anti-cheat may flag automated input).
- **Performance:** keep per-tick state serialization bounded (entity scan is capped at 50 / 16
  blocks; block scan is crosshair-only in Phase 1). Profile before widening scans in Phase 3.

## 8) How to verify your work (the user values this)

Every phase ships with tests, not just code: mod loads, socket handshake works, and the phase's
headline behavior actually happens in-game (e.g., "mine 10 wood" completes). Reuse
`tools/mock_mod_server.py` to unit-test protocol changes without launching Minecraft, and add
in-game acceptance steps to a `PHASE<n>_VERIFICATION.md` like the Phase 1 one.

---

*Prior assistant's note:* the offline protocol test passes; the mod is un-compiled only because
this environment had Java 11 and no network. Nothing about the code is known-broken — start at §4.
