# Mounts of Mayhem Agent — Phase 1

A Fabric mod for **Minecraft Java 1.21.11** plus a **Python controller** that talk over a
loopback TCP socket. Phase 1 delivers the foundation everything else builds on: a reliable
**game-state export** and a **legal manual-control API**. No AI, HUD, or learning yet — those
are later phases (see *Roadmap*).

> **Architecture (agreed):** hybrid — a local LLM planner (7–8B class, sized for an 8–12 GB
> GPU) decomposing goals into scripted skills, with a narrow RL layer added later. Multiplayer-
> capable, with lightweight/headless `/train` bots. Phase 1 lays the client-side bridge that the
> planner will drive.

---

## Why control happens on the client

To let the AI drive **your own player**, Phase 1 runs on the client and injects **vanilla
keybinding input** (press "forward", set look angle, hold "attack") instead of teleporting or
forcing velocity. The unmodified movement/mining code then runs exactly as if you were at the
keyboard. This keeps movement **legal** (important for the future speedrun goal) and avoids
fighting the client/server position reconciliation. Server-side control of `/train` bot players
comes in Phase 4.

---

## Requirements

- **JDK 21** (Minecraft 1.21.x requires Java 21 to build and run).
- **Minecraft Java 1.21.11** with **Fabric Loader** and **Fabric API**.
- **Python 3.9+** (controller uses the standard library only).
- Internet access on first build (Gradle downloads Minecraft, mappings, and Fabric API).

Pinned versions (verified against `meta.fabricmc.net`, July 2026), in `gradle.properties`:

| Component      | Version             |
|----------------|---------------------|
| Minecraft      | `1.21.11`           |
| Yarn mappings  | `1.21.11+build.5`   |
| Fabric Loader  | `0.19.2`            |
| Fabric API     | `0.141.4+1.21.11`   |
| Loom plugin    | `1.16.3`            |
| Gradle wrapper | `8.12`              |

---

## Build the mod

This project ships **without** the `gradle-wrapper.jar` binary (it can't be generated in a
text-only environment). Create the wrapper once, then build:

```bash
cd moam-agent-phase1

# One-time: generate the Gradle wrapper (needs a system Gradle, OR just open the
# project in IntelliJ IDEA, which downloads Gradle for you and skips this step).
gradle wrapper --gradle-version 8.12

# Build the mod jar:
./gradlew build          # Windows: gradlew.bat build
```

The mod jar lands in `build/libs/moamagent-0.1.0.jar`.

### Run in a dev client (recommended for testing)

```bash
./gradlew runClient
```

Loom launches a dev Minecraft client with the mod already loaded — no manual install needed.

### Or install into a real instance

Copy `build/libs/moamagent-<version>.jar` **and** the Fabric API jar into your `.minecraft/mods`
folder (with Fabric Loader installed for 1.21.11). **The mod jar is the only mod file you need** —
the Python controller ships *inside* it (see below).

### The jar is the only file needed (self-extracting controller)

From v0.3.2 the mod bundles the Python controller and self-extracts it on load, so you don't
copy any Python around by hand. On init the mod creates a **`moamagent-AI/`** folder next to your
`mods/` folder (the game/profile root) and unpacks everything the AI side needs:

```
moamagent-AI/
  agent_client.py  goals.py  agent_home.py  trainer.py  requirements.txt
  run.bat  run.sh  README.md      # turnkey launchers + docs
  version.txt                     # mod version that extracted this (re-extract policy)
  logs/                           # per-day session logs + controller.log
  data/                           # run/reward history (history.jsonl)
    models/                       # learned RL weights (reach_q.json)
```

**Always-on (v0.4.0+):** the mod **auto-launches the controller for you** by default, so the AI
is ready as soon as the game is. It runs `agent_client.py serve`, which reconnects on its own and
**trains an RL model in the background** while idle. You **turn the AI (and the trainer) on/off
from the dashboard — press Right Shift**. If Python isn't on PATH the mod logs a clear pointer and
never hard-fails.

- **Two RL backends, auto-selected (v0.5.0+):** a **PyTorch DQN** (deep) when torch is available,
  else a **stdlib tabular Q-learner** (zero-install). On the first launch a **one-time setup**
  (`setup_env.py`) installs torch/numpy/gymnasium and writes `moamagent-AI/setup.json` marking it
  done; later launches skip straight past it. If the install can't run, it falls back to the
  tabular backend and retries next time. Force the light path with `run.bat --backend tabular`.
- **Learns from real play (v0.6.0+):** when a world is open and the trainer is on and idle, the RL
  policy **drives your character** to practice navigation and learns from the real transitions
  (dashboard shows `Mode: LIVE (real play)`); on the title screen it keeps learning in an internal
  sim. `/train` (and `/train stop`) turns the live learner loose on demand. New goal
  **`/goal collect <n> <item>`** gets N of an item by mining its source, and the biome/mob/block
  search now explores outward in an expanding star instead of circling.
- **Local LLM planner (v0.7.0+):** with **Ollama** running, **`/plan <goal>`** (e.g. `/plan get 3
  diamonds`) has a local model decompose the goal into a step-by-step plan with a time estimate; the
  bot executes what it can and the LLM **re-plans** if a step fails. It **learns from a reward**
  (success + speed + how close its time estimate was), feeding its best plans back as few-shot
  examples and periodically baking them into a local `moamagent-planner` model. Falls back to the
  scripted skills without Ollama. `/goal collect` also routes through the planner when it's up.
- Opt out of auto-launch: create an empty `moamagent-AI/autostart.disabled` file (or pass
  `-Dmoamagent.autostart=false`), then start it yourself with `run.bat` / `run.sh`.
- Learned weights persist to `moamagent-AI/data/models/` (`reach_q.json` tabular / `reach_dqn.pt`
  deep) and reload on the next launch, so training resumes across sessions.

**Re-extract policy:** `version.txt` stamps the extracting mod version. A newer jar re-extracts
the bundled files (matching the controller to the mod); the same version leaves them alone, and
your `logs/`/`data/` are never touched. Customising the controller? Copy it to a new filename —
shipped files are replaced on upgrade.

---

## Run the controller

With Minecraft running, the mod loaded, and a **world open**, the bridge listens on
`127.0.0.1:25599`. In-game you can type `/agent status` to confirm. The turnkey path is
`moamagent-AI/run.bat` / `run.sh` (above); the raw sub-commands below are the same CLI and are
handy for dev/testing from the `python/` source tree:

```bash
cd python   # or: cd moamagent-AI

# Stream a one-line state summary each half-second:
python agent_client.py watch

# Dump the full JSON of the next snapshot:
python agent_client.py raw --count 1

# Scripted proof of control: look, sprint forward 3s, stop, report distance moved:
python agent_client.py demo --seconds 3

# Hold attack for 5s to mine whatever block you're aiming at:
python agent_client.py mine --seconds 5
```

`agent_client.AgentConnection` is also importable as a library — that's the entry point the
Phase 3+ planner will use.

---

## Protocol (Phase 1)

Newline-delimited JSON, both directions.

**Mod → controller** (one per client tick):

```json
{
  "schema": 1, "tick": 1234, "inWorld": true,
  "player": { "pos": {"x":0,"y":64,"z":0}, "yaw":0, "pitch":0, "onGround":true,
              "health":20, "maxHealth":20, "food":20, "air":300, "xpLevel":0,
              "velocity": {"x":0,"y":0,"z":0} },
  "world":  { "dimension":"minecraft:overworld", "difficulty":"normal",
              "hardcore":false, "gameMode":"survival", "biome":"minecraft:plains" },
  "targetBlock": { "x":1,"y":63,"z":2, "block":"minecraft:stone", "side":"up" },
  "inventory": { "selectedSlot":0, "hotbar":[ {"slot":0,"item":"minecraft:air","count":0} ] },
  "entities": [ {"type":"minecraft:zombie","hostile":true,
                 "rel":{"dx":3,"dy":0,"dz":-2},"distance":3.6} ]
}
```

**Controller → mod** (one command per line):

| Command | Fields | Effect |
|---------|--------|--------|
| `look`  | `yaw`, `pitch` | set look angles |
| `move`  | `forward` (−1..1), `strafe` (−1..1), `jump`, `sneak`, `sprint` | hold movement keys |
| `mine`  | `hold` (bool) | hold/release attack (breaks targeted block) |
| `use`   | `hold` (bool) | hold/release use (place/interact) |
| `stop`  | — | release all inputs |

Inputs **persist** until changed, so issue a command once and it keeps applying. On controller
disconnect the mod releases all keys automatically.

---

## Testing without Minecraft

`tools/` contains a mock bridge that speaks the identical protocol with a toy physics model,
so the controller and wire format can be verified in plain Python:

```bash
python tools/test_protocol.py     # automated: connect, move, stop, assert — exit 0 = pass
python tools/mock_mod_server.py   # run the mock manually, then point agent_client.py at it
```

See `PHASE1_VERIFICATION.md` for the full manual + automated checklist (including the
in-game acceptance tests that require a real client).

---

## Roadmap

- **Phase 1 (this):** mod skeleton, JSON state export, legal manual-control bridge, Python client. ✅
- **Phase 2:** HUD (goal / elapsed time / session cost) + Right-Shift dashboard (metrics, hyperparameters, hardware, cost, stop-by-time/cost controls).
- **Phase 3:** scripted goal executor + `/goal`, `/train`, game-mode awareness (Creative/Survival/Hardcore), close-world-on-complete.
- **Phase 4:** learning loop (narrow RL) + parallel headless `/train` bots + LLM planner integration.
- **Phase 5:** `/goal speedrun` route executor with movement-legality checks.

Each phase gets its own scope sign-off and verification tests before code.

## Notes & caveats

- **Feasibility:** open-ended goals ("beat the game", redstone, any% speedrun) are research-grade
  (cf. MineRL/VPT/Voyager). The realistic path is an LLM planner over reliable scripted skills,
  with narrow RL layered on — not a from-scratch agent that masters everything. The dashboard will
  always label what is *scripted* vs *learned*.
- **EULA / distribution:** don't redistribute Minecraft assets or the deobfuscated jars produced
  during the build. Mod code here is MIT; ship only your own jar. Automation that plays the game is
  fine for personal/local use; if you ever point this at a multiplayer server you don't own, get
  the operator's permission (server anti-cheat may also flag automated input).
- **Mapping-sensitive spots** to re-check if the build complains (yarn method names occasionally
  shift between versions): `PlayerInventory#getSelectedSlot`, `LevelProperties#isHardcore`,
  `ClientPlayerInteractionManager#getCurrentGameMode`. `PHASE1_VERIFICATION.md` lists the fixes.
