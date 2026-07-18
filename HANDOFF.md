# HANDOFF — Mounts of Mayhem Agent (for the next AI assistant)

Read `CLAUDE.md` too (it holds the locked decisions + full roadmap). This file is the **current-state
snapshot (v0.7.0) + what to build next**. Everything below is built and verified on the user's
Windows machine.

---

## 1. What this project is

A **Fabric mod for Minecraft Java 1.21.11** + a **Python controller** that lets an AI observe and play
the game over a loopback TCP socket (`127.0.0.1:25599`, newline-delimited JSON). The mod is the server
(exposes state, applies control by holding vanilla keybindings — legal input, not teleport); Python
decides actions. Long-term goal: an **LLM-planner-over-scripted-skills** agent with **narrow RL**,
always honest about scripted-vs-learned and what it can actually execute.

**The jar is the only file the user installs.** It self-extracts the whole Python controller into
`<gameDir>/moamagent-AI/` on load and (by default) auto-launches it.

## 2. Current state — deployed jar is `moamagent-0.8.0.jar`

**v0.8.0 (next-steps item #4 — THREE LEARNERS + HYPER-AWARENESS).** The most important step.
* **Persistent world model** (`python/world_model.py`, NEW). Per-tick state was bounded (≤16 blocks) —
  perception, not awareness. `WorldModel.observe(state)` turns the snapshot stream into a durable,
  queryable map: it remembers every dropped item / mob / scanned resource with last-known position +
  last-seen time. Things inside the 14-block bubble that stop being reported are treated as GONE
  (picked up / moved / died) and dropped; things that merely left range are REMEMBERED until a
  per-category TTL (items 180 s, mobs 45 s, resources 900 s). Queries: `nearest_item/hostile/resource`,
  `threats`, `count_item`, plus a `telemetry()` block and `describe()`. Fed every supervisor pass.
* **Richer perception from the mod.** `StateSerializer` now emits a dedicated **`items`** array
  (dropped stacks: id/item/count/pos/rel/distance) and enriches **`entities`** with a stable network
  `id`, `category` (hostile/passive/mob/player/other), `health`, and absolute `pos`.
* **Third learner — step-interpreter** (`python/step_interpreter.py`, NEW). Maps a fuzzy instruction
  ("go chop some oak", "head to 120 -40") → an EXACT runnable skill `{kind,item/block/id/count/x/z}`.
  The hard part (picking the skill KIND — "go chop oak" is *collect* not *goto*) is a learned online
  multiclass **averaged perceptron** over token features, bootstrapped from templates then rewarded on
  valid mappings; args are extracted deterministically. Same facade as the RL trainers
  (train_slice/metrics/info/save/load/episodes/curve). Persists to `data/models/interpreter.json`.
  **Actually used:** `PlanRunner` calls it to rescue messy/unknown plan steps, and self-trains it on
  every step that runs successfully.
* **`/train` now drives THREE learners** (`planner`, `nav`, `interpreter`), each toggled independently:
  in-game `/train <planner|nav|interpreter|all> [stop]` (autocompleted) + three dashboard buttons +
  a **LEARNERS panel** showing each one's on/off, live metric, episodes and best. The supervisor’s
  idle loop advances every enabled learner (nav = live/sim RL, interpreter = synthetic practice,
  planner = periodic `bake()`), and streams them in `agentStatus.ai.learners`.
* **Awareness everywhere.** New `agentStatus.awareness` block → a dashboard **AWARENESS** section and an
  always-on **HUD line** (`◎ 3 items · 2 mobs !1 · threat zombie 8m`) that shows even when idle.
* **Tests:** `tools/test_world_model.py`, `tools/test_interpreter.py`, `tools/test_learners.py`
  (integration vs an extended mock that now emits items + a cow + a zombie via `--populate`). All pass.
* **Honest scope:** navigation is still reactive-RL + hazard-mask, NOT voxel A\* routing (that stays a
  later phase); the planner "learns" by exemplar-distillation, not gradient LoRA. Nothing is faked.


**v0.7.2 (next-steps item #3 — BETTER PLANNER MODEL):** model selection is now availability-aware and
prefers stronger models. `PREFERRED_MODELS` is reordered best-first with **phi3:mini LAST**
(llama3.1:8b → qwen2.5:7b → qwen3:8b → mistral:7b → dolphin-mistral → phi3:mini), so on this machine
the planner auto-picks **dolphin-mistral (7B)** instead of the weak phi3:mini — zero download.
`resolve_model` now matches tags leniently (`dolphin-mistral` ↔ `dolphin-mistral:latest`,
`llama3.1:8b` ↔ `…-instruct-q4`), honours `MOAM_LLM_MODEL` (hard override) and `MOAM_PREFERRED_MODELS`
(reorder), and prefers the reward-baked `moamagent-planner` when present. New `agent_client.py models`
prints the resolved model + why + how to change it; the resolved model now also shows on the dashboard
MODEL panel while idle (`_send_idle` runtime block). `describe_model()` + `preferred_models()` +
`match_model()` added to `llm_planner.py`.

**v0.7.1 (next-steps item #1 — SHOW THE PLAN):** the LLM's decomposed plan is now visible.
`PlanRunner` streams a new `plan` block in `agentStatus` (goal, strategy, est/elapsed/ETA seconds,
current step, reward, and every step with a live `state` = pending|active|done|skipped|failed and an
`executable` flag). `AgentStatus.java` parses it (`PlanStep` record + accessors + a change-signature).
Rendered three ways: a **PLAN panel** on the dashboard (left column under the chart, colour-coded
steps), a compact HUD line (`▷ Plan 2/5 · mine · ETA 0:45`), and a **one-time chat echo** of each new
plan (`MoamAgentClient.echoNewPlan`). Verified with `tools/test_plan_stream.py`. Items #2–5 below
still pending.


In `C:\Users\nmighu\AppData\Roaming\ModrinthApp\profiles\Minecraft AI bot\mods\` (with the required
`fabric-api-0.141.5+1.21.11.jar`). What's built and verified:

- **P1–P3.5** — socket bridge, per-tick state, HUD + Right-Shift dashboard, scripted goal executor
  (`/goal goto|mine|biome|findmob|cancel`), survival/hazard-avoidance + `RewardLedger`, on-demand
  `findBlocks`/`scanArea` queries, registry autocomplete.
- **Packaging (v0.3.2)** — `AgentHome.java` self-extracts bundled `/agent/*` into `moamagent-AI/`
  (version-stamped, never clobbers user `logs/`/`data/`). Python bundled via `build.gradle`
  `processResources`.
- **Always-on + RL (v0.4.0–0.5.0)** — mod auto-launches `agent_client.py serve` (resilient reconnect).
  Dashboard **AI: ON/OFF** + **Trainer: ON/OFF**. Two auto-selected RL backends: `trainer.py`
  (stdlib tabular Q-learning) / `deep_trainer.py` (PyTorch **DQN**, CUDA-capable, defaults CPU because
  the net is tiny). One-time self-healing dep install `setup_env.py` (gated by `setup.json`; installs
  torch/numpy/gymnasium into the current interpreter). Weights persist to `data/models/`.
- **Live RL + planner + collect + search fix (v0.6.0)**:
  - **`live_train.py` (`LiveReach`)** — the RL **drives the REAL character** to chained waypoints (no
    teleport) and learns from real transitions; same 4-feature obs / 5 actions as the sim so the model
    transfers. Supervisor prefers **live** when a world is open, else **sim**. Dashboard shows
    `Mode: LIVE (real play)`. This closed the "always learning but never trained because it isn't
    controlling anything" gap.
  - **`/goal collect <n> <item>`** — planner skill: item→source-block map, search-outward → approach →
    harvest, counting the item across the WHOLE inventory (new `inventory.counts` in state).
  - **Search fix** — old `_explore` circled in place; now an expanding outward star (ranges >20 blocks).
  - **`/train`** [stop] — turns the live learner loose on the character.
- **Local LLM planner (v0.7.0)** — `llm_planner.py` + `plan_runner.py`:
  - Talks to **local Ollama** over REST (stdlib `urllib`, no pip dep). Strict JSON plan schema
    `{goal, estimatedSeconds, strategy, steps:[{skill,args,note}]}`; robust parser normalises loose
    args and drops un-runnable steps (a `goto "oak forest"` is dropped, not walked to (0,0)).
  - Executable skills = goto/mine/collect/biome/findmob; higher-level steps (craft/portal/kill_boss/
    build) are kept but **marked not-executable, not faked**.
  - **`PlanRunner`** runs each executable step via `GoalRunner`; on a failed step calls
    `LLMPlanner.replan()` from live state and splices in the fix (≤2 replans).
  - **Reward + learning:** `score_plan()` = success + faster→more + ETA-accuracy (|pred−actual|,
    closer→more). Best plans go to a reward-ranked `ExemplarStore` and feed back as few-shot (online
    learning); `bake()` distils the top ones into a local `moamagent-planner` model via an Ollama
    **Modelfile** every 10 plans; `export_sft()` writes a JSONL for optional LoRA. Data in
    `moamagent-AI/data/planner/`.
  - **Wiring:** in-game `/plan <text>` and `/goal collect` route through the planner when Ollama is up
    (else scripted). Python `agent_client.py plan "<text>"`. `serve --no-llm` / `--backend` / `--device`.
  - **Ollama on this machine:** installed (client 0.13.3); models pulled: `phi3:mini` (3.8B),
    `dolphin-mistral:latest` (7B), `MyDarkGPT`. Server is normally run by the Ollama desktop app; start
    manually with `ollama serve` (binary at `C:\Users\nmighu\AppData\Local\Programs\Ollama\ollama`).
    phi3:mini follows the JSON format but is weak — the user wants a better model (see §7).

## 3. Build environment — READ before building

- **Windows 11.** Bash tool (Git Bash/POSIX) + PowerShell tool. Use Unix syntax in Bash. **cp1252
  console** — keep Python `print()`/test output ASCII (`→`, `×` beyond 0xD7 crash the terminal, not the
  logic; use `->`).
- **JDK 21 (required):** `C:\Program Files\Java\jdk-21`. Set `JAVA_HOME` to it.
- **Gradle 9.4.0** (Loom 1.16.3 needs ≥9.4.0; the pinned wrapper 8.12 is WRONG, and there's no wrapper
  jar). A working Gradle is already unpacked at
  `C:\Users\nmighu\.gradle\wrapper\dists\gradle-9.4.0-bin\lcvyxq3t37f6mx9miaydrrgs\gradle-9.4.0\bin\gradle`.
  Build:
  ```bash
  export JAVA_HOME="C:/Program Files/Java/jdk-21"
  cd "C:/Users/nmighu/Downloads/moam-agent-phase1"
  "<that gradle>" build --no-daemon --console=plain
  # -> build/libs/moamagent-<version>.jar  (use the NON -sources jar)
  ```
  If that dist is gone: `curl -sL -o g.zip https://services.gradle.org/distributions/gradle-9.4.0-bin.zip && unzip -q g.zip`.
  The `.gradle` caches already hold the MC/yarn/fabric-api deps (first cold build needs internet).
- **Version matrix (all real):** MC 1.21.11, yarn 1.21.11+build.5, loader 0.19.2, fabric-api
  0.141.4/5, loom 1.16.3, Gradle 9.4.0, Java 21. Mapping fixes applied: `GameMode.getId()`,
  `KeyBinding.Category` is a record (`.MISC`). `javap` the loom named jars under
  `<GRADLE_USER_HOME>/caches/fabric-loom/minecraftMaven/.../minecraft-{clientonly,common}-1.21.11-*.jar`
  to look up other mappings.
- **Deploy** = copy the jar into the Modrinth mods folder, delete the old versioned jar, keep Fabric
  API. Bump `mod_version` in `gradle.properties` so `AgentHome` re-extracts on next launch.
- Stray locked `./moam-agent-phase1/` nested folder is NOT in the build — ignore it.

## 4. Tests (offline, no Minecraft — all pass). `tools/mock_mod_server.py` is the fake bridge.

```bash
python tools/test_protocol.py    # wire protocol
python tools/test_goals.py       # goto + mine + collect vs mock
python tools/test_survival.py    # hazard/threat/reward logic
python tools/test_trainer.py     # tabular RL learns + weights round-trip
python tools/test_deep.py        # setup.json state machine + DQN learns (needs torch)
python tools/test_live.py        # LIVE RL drives the mock character + learns
python tools/test_planner.py     # LLM plan parse/reward/store + real Ollama plan+exec (skips w/o Ollama)
python tools/test_plan_stream.py # v0.7.1 "show the plan": PlanRunner streams a well-formed plan block
python tools/test_world_model.py # v0.8.0 awareness: remember/forget items+mobs+resources
python tools/test_interpreter.py # v0.8.0 3rd learner: NL->skill mapping accuracy + learning + save/load
python tools/test_learners.py    # v0.8.0 integration: 3 learners + awareness vs mock (--populate)
```
The mock now takes `--populate` (or `serve(populate=True)`) to spawn dropped items + a cow + a hostile
zombie for awareness tests; it's OFF by default so goal/nav tests keep a clean, threat-free world.
The mock simulates toy physics, a small oak-log world, `findBlocks`, and (v0.6.0) inventory pickup on
break. It has **no entities, no biome changes, no chests** — extend it when testing those.

## 5. Protocol quick reference (newline JSON, loopback :25599)

- **Mod→controller each tick (state):** `player` (pos/look/health/food/air/xp/velocity +
  inWater/submerged/inLava/onFire), `world` (dimension/difficulty/hardcore/gameMode/biome), `nav`
  (hazard probe wallAhead/edgeAhead/dropDepth/waterAhead/lavaAhead/facing), `targetBlock`,
  `inventory` (`hotbar` + **`counts`** = item→total across whole inventory), bounded `entities`
  (now each with **`id`/`category`/`health`/`pos`**) + a dedicated **`items`** array (dropped stacks:
  id/item/count/pos/rel/distance).
- **Controller→mod commands:** `look`, `move`, `mine`, `use`, `stop`, `agentStatus` (telemetry: goal/
  action/cost/runtime/hyperparams/metric/limits/`score`/**`plan`**{steps,current,eta,reward}/
  **`ai`**{enabled,training,steps,backend,setup,mode,best,successRate,**`learners`**[{name,label,enabled,
  metric,value,episodes,best,note,available}]}/**`awareness`**{items,mobs,hostiles,resources,biome,pos,
  nearestItem,nearestHostile,nearestResource}), `query` (`findBlocks`/`scanArea`), `quitWorld`.
- **Mod→controller events (tagged with `event`):** `goalRequest` (goal kinds: goto/mine/collect/biome/
  findmob/**plan**), `goalCancel`, `queryResult`, `control` (cancelGoal/stopAfterTime/stopAfterCost/
  setAi/setTrain/**train**{**which**:planner|nav|interpreter|all,value}/**setLearner**{which,value}).

## 6. Data home layout (`<gameDir>/moamagent-AI/`)

`*.py` (extracted controller) · `run.bat`/`run.sh` · `version.txt` · `setup.json` ·
`logs/` (session + `controller.log`) · `data/history.jsonl` · `data/models/` (`reach_q.json` /
`reach_dqn.pt`) · `data/planner/` (`exemplars.jsonl`, `Modelfile`, `sft_dataset.jsonl`).

---

## 7. WHAT TO BUILD NEXT (the user's request — do these first, in order)

The current LLM (phi3:mini) isn't smart, so the plan is to (a) make the loop robust to a weak model by
grounding every step in the live world, (b) make the plan visible, (c) move to a better model, and
(d) turn `/train` into a trainer for THREE learners. Then add crafting.

1. **Show the LLM's plan.** ✅ DONE (v0.7.1). `PlanRunner._stream` now emits a `plan` block in
   `agentStatus` (goal/strategy/est/elapsed/ETA/current/reward + per-step state & executable flag);
   `AgentStatus.java` parses it into `PlanStep`s; the dashboard PLAN panel, HUD line, and a one-time
   chat echo render it. See `tools/test_plan_stream.py`.

2. **Step-by-step, world-grounded execution (bare-minimum steps).** Because the model is weak, don't
   trust a big upfront plan — make it plan/execute the **smallest next step**, re-checking reality each
   time:
   - Before each step, gather **context** = what's around us (`scanArea`), **what's in our inventory**
     (`inventory.counts`), and **nearby chest contents** — this needs a NEW mod query
     (`nearbyContainers`/`scanChests`: find chests/barrels within N blocks and, when reachable, report
     their contents) plus a skill to open/take from a chest. If a nearby chest already has what we
     want, take it instead of mining.
   - Feed that context into the planner each iteration and ask for just the next 1–2 concrete steps;
     re-ground after each. Keep steps atomic. (`PlanRunner` already re-plans on failure — extend it to
     re-plan/observe **every step**, and to prefer chest-grab > craft > mine when the item is close.)

3. **Select + keep training a better model.** ✅ DONE (v0.7.2). Auto-selection now prefers stronger
   models (phi3:mini last) and is availability/tag-aware, so it picks **dolphin-mistral (7B)** here
   with no download. Override via `MOAM_LLM_MODEL`, reorder via `MOAM_PREFERRED_MODELS`; inspect with
   `agent_client.py models`; the resolved model shows on the dashboard. The reward→exemplar→`bake()`
   loop (and SFT export) is unchanged and keeps improving whichever base is selected. *Optional later:*
   `ollama pull llama3.1:8b` / `qwen2.5:7b` — they're already first in the preference order, so pulling
   one makes the planner use it automatically.

4. **Make `/train` train THREE learners.** ✅ DONE (v0.8.0) — planner + navigator RL + the NEW
   step-interpreter, each toggled via `/train <which>` + dashboard buttons + a LEARNERS panel, plus the
   persistent world model for hyper-awareness (HUD + AWARENESS panel). Details in §2. The original ask:
   1. **the LLM planner** — via the reward/exemplar/bake loop (and SFT export) already in `llm_planner.py`.
   2. **the future PATHFINDING AI** — the RL/model that will replace the reactive navigator (today it's
      `LiveReach` + the DQN/tabular reach policy; grow it toward real routing).
   3. **the STEP-INTERPRETER AI** — a model/agent that turns an LLM step (or a user instruction) into
      the exact executable skill call + args. This is a new learner: train it to map fuzzy steps
      ("go chop some oak") → `{kind:collect,item:minecraft:oak_log,count:N}` reliably, rewarded on
      correct/valid mappings. (Bootstrap it with the normaliser in `llm_planner.normalize_step` +
      exemplars; it's the piece that lets a weak planner still produce runnable actions.)
   `/train` should let you train/toggle each of the three (dashboard buttons + `/train <which>`), and
   the dashboard should show all three learners' progress.

5. **THEN add crafting (roadmap #1).** New executable skills — craft (recipe lookup + crafting-table
   interaction), smelt (furnace) — so the LLM's plans that need intermediate items actually run. This
   is the biggest unlock toward "collect by any means" and "beat the game". After crafting: dimension
   traversal (nether/end), then boss fights/building.

## 8. Eventually (later roadmap, from CLAUDE.md + memory)

- Real **A\*** pathfinding (nav is still reactive — dense terrain stumps it).
- True **LoRA/gradient fine-tune** of the planner (SFT dataset is already exported) instead of only
  Modelfile exemplar-distillation.
- **`/train <n>` parallel headless practice bots** (server-side fake players) for faster RL.
- **`/goal collect` by ANY means** fully (farm/trade/other-dimension/chests/set-spawn), driven by the
  planner once the skills exist.
- **Speedrun** goal (Phase 5) as a legal, self-audited scripted route.

Guiding constraints (do not regress): keep it **honest** (label scripted vs learned; never fake a
skill the bot can't do); **the jar stays the only file**; socket is **loopback only**; per-tick state
stays small, rich detail via on-demand queries; keep everything **verifiable against the mock**.
