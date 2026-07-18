# Phase 4 verification — always-on AI + narrow RL trainer + dashboard on/off (v0.4.0)

Phase 4 (this pass) makes the jar's AI **always-on**, **toggleable from the Right-Shift
dashboard**, and adds a **real, dependency-free RL trainer** whose **weights persist** to the AI
home. This file is the checklist. Deferred (not stubbed): `/train` headless bots, the LLM planner
(Ollama), and deep RL (torch/gymnasium) — see `CLAUDE.md` §5.

## What shipped

- **`serve` supervisor** (`agent_client.py serve`): reconnects forever (survives no-world and
  disconnects), runs `/goal` requests, and trains in the background while idle. This is what the
  mod auto-launches and what `run.bat`/`run.sh` run.
- **Auto-launch ON by default** (`AgentHome.maybeAutostart`): opt out with
  `moamagent-AI/autostart.disabled` or `-Dmoamagent.autostart=false`. Best-effort; no Python =
  clear log line, no hard fail.
- **Dashboard toggles** (`DashboardScreen`): **AI: ON/OFF** and **Trainer: ON/OFF** send
  `{"event":"control","action":"setAi|setTrain","value":0|1}`; labels + the `AI / LEARNING` panel
  reflect the controller's reported state (`agentStatus.ai{...}`, parsed by `AgentStatus`).
- **Two RL backends**: `trainer.py` (stdlib tabular Q-learning) and `deep_trainer.py` (PyTorch
  **DQN**, CUDA-aware) on the shaped reach-target task (`ReachEnv`). Same interface; auto-selected
  (deep when torch present). Weights → `data/models/reach_q.json` / `reach_dqn.pt` (atomic replace).
- **One-time self-healing setup** (`setup_env.py`): installs torch/numpy/gymnasium into the
  current interpreter on first `serve`, gated by `moamagent-AI/setup.json`. Re-runs only if the
  marker is missing/not-done or a dep won't import; falls back to tabular on failure. `--no-setup`
  / `--backend tabular` skip it.

## A. Offline (no Minecraft) — all automated, must pass

```bash
python tools/test_trainer.py     # tabular learns (greedy 3%->~50-60%) + weights round-trip
python tools/test_deep.py        # setup.json state machine; DQN learns + round-trips (if torch)
python tools/test_protocol.py    # protocol still green
python tools/test_goals.py       # goal state machines still green
python tools/test_survival.py    # survival layer still green
```

Extra manual offline proof (done during development, reproducible):

```bash
# Train standalone and watch the curve climb, then save weights:
cd python && python trainer.py --episodes 4000 --model /tmp/reach_q.json
```

## B. End-to-end packaging (no Minecraft) — extract-from-jar + serve

1. Unzip the built jar's `agent/*` into a temp `moamagent-AI/`, write `version.txt`.
2. Start `tools/mock_mod_server.py`.
3. From that folder, run the supervisor (`agent_client.py serve`) for a few seconds.

Expect: `is_home: True`; episodes climb; `data/models/reach_q.json` written; a session log in
`logs/`; `data/history.jsonl` gains `{"kind":"train",...}` records. Re-launching resumes from the
saved episode count (warm-start). **Verified.**

## C. In-game acceptance (needs a human at the client) — PENDING

1. Drop `moamagent-0.5.0.jar` + Fabric API into `mods/`; launch the game.
   - Log shows: `Extracting AI controller to …moamagent-AI (upgrade … -> 0.5.0)` then
     `Auto-launched AI controller with 'python' (always-on…)`.
   - **First launch only:** `moamagent-AI/logs/controller.log` shows the one-time setup
     (`[setup] …installing deep-RL deps …` if any were missing), and `moamagent-AI/setup.json`
     appears with `"done": true`. Later launches skip straight past it.
   - The controller log shows the chosen backend (`RL backend: deep — DQN … on GPU · CUDA …`
     when torch is present) connecting once a world is open, and episodes advancing.
2. Open a world. Press **Right Shift**.
   - The `metric over time` chart climbs (train success %); `AI / LEARNING` shows Episodes rising,
     Success %, Best reward; Approach = `learned (Q-learning)`.
3. Click **Trainer: OFF** → episodes stop advancing, label flips to OFF. Click **AI: OFF** → the
   `AI` line/label shows OFF and background work stops. Toggle both back ON → resumes.
4. `/goal mine 5 minecraft:oak_log` → the agent executes the scripted goal (trainer pauses during
   the goal, resumes after).
5. Quit to title, relaunch: `moamagent-AI/data/models/reach_q.json` is present and the controller
   log reports `resumed training from N episodes`.

Phase 4 (this pass) is done when §A and §B pass (they do) and §C is confirmed by a human at the
client.
