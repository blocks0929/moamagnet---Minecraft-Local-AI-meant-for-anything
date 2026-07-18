# Phase 2 — Verification checklist (HUD + Dashboard)

Phase 2 adds the **display + telemetry layer**: an always-on HUD during a run, a Right-Shift
dashboard, and a two-way protocol extension (`agentStatus` in, `event`/control lines out). The
data feeding it is *synthetic* for now — the real goals/training/GPU numbers arrive in Phases 3–4.

Done when every box below is checked. Split into **offline** (no Minecraft, already run) and
**in-game** (real 1.21.11 client).

## A. Automated / offline (no Minecraft) — verified in this environment

- [x] Python + tools compile (`python -m py_compile python/agent_client.py tools/*.py`).
- [x] **Protocol test passes**: `python tools/test_protocol.py` → `ALL PASSED`, now including:
  - `agentStatus` telemetry is accepted by the bridge;
  - the mock's `{"event":"control","action":"ackStatus"}` is routed to the **event** channel;
  - the per-tick **state stream stays intact** after an event (events don't clobber `latest()`).
- [x] Mod compiles against real `1.21.11+build.5` mappings and remaps
      (`build/libs/moamagent-0.2.0.jar`). `KeyBinding.Category` (now a record) and
      `HudRenderCallback` both resolved.

Re-run any time:

```bash
python tools/test_protocol.py      # expect exit code 0 / ALL PASSED
```

## B. Build + deploy (needs JDK 21 + internet)

- [x] Build succeeds with **Gradle 9.4.0** (Loom 1.16.3 requires ≥9.4.0 — the old
      CLAUDE.md figure of 8.12 was wrong; see note there).
- [x] `moamagent-0.2.0.jar` deployed to
      `…/ModrinthApp/profiles/Minecraft AI bot/mods/` (0.1.0 removed).

## C. In-game acceptance (real 1.21.11 Fabric client) — run on your machine

Launch the "Minecraft AI bot" profile in Modrinth and open a world, then:

- [ ] **No HUD when idle** — before any telemetry, the screen is clean (HUD only shows during a run).
- [ ] In a terminal: `cd python && python agent_client.py dashboard`
  - Console prints "Streaming synthetic agent telemetry…".
- [ ] **HUD appears** while `dashboard` streams:
  - top-left: `▶ mine 10 oak_log` + a changing action line;
  - top-right: an **elapsed timer counting up** + `GPU x.xs · y.y Wh` **rising** over time.
- [ ] Press **Right Shift** → the **dashboard opens** and the **game does not pause**
      (mobs keep moving behind it; `shouldPause()` is false).
- [ ] Dashboard shows: a **reward line-chart that grows**, the **Model/Approach/Hardware +
      hyperparameters** panel, the **cumulative cost breakdown**, and the **Limits** section.
- [ ] Click **Stop after 60s** / **Stop at 25 GPU·s** / **Cancel goal** →
      each prints a `<- control event: {…}` line back in the Python console
      (proves the dashboard→controller channel; real enforcement is Phase 3).
- [ ] Press **Right Shift** again or click **Close** → dashboard closes, HUD still visible.
- [ ] `Ctrl-C` the `dashboard` command → it sends `running:false`; **HUD disappears**.
- [ ] `/agent dashboard` also opens the screen (fallback to the keybind).

Phase 1 behaviour is unchanged — `watch` / `demo` / `mine` still work (events are ignored by them).

---

## Notes / design

- **HUD API**: uses `HudRenderCallback` (compiles with a deprecation note — Fabric now prefers
  `HudElementRegistry`; migrating is a trivial later change and not required for 1.21.11).
- **One connection at a time**: the bridge still accepts a single controller, so run *either*
  `dashboard` *or* `watch`/`demo`, not both at once.
- **Cost model**: `dashboard` fakes ~0.8 GPU-seconds per wall-second and estimates energy from
  `--watts` (default 180 W). These are placeholders until the Python planner reports real
  GPU-seconds in Phase 4. Token-cost is wired through (`cost.tokenCost`) but 0 for local models.

## Protocol additions (Phase 2)

Controller → mod (one per line, in addition to Phase 1 commands):

```json
{"type":"agentStatus","running":true,"goal":"…","action":"…",
 "cost":{"gpuSeconds":42.5,"energyWh":2.1,"tokenCost":0.0},
 "runtime":{"model":"qwen2.5:7b","approach":"scripted","hardware":"local GPU"},
 "hyperparams":{"lr":"3e-4","gamma":"0.99"},
 "metric":{"name":"reward","series":[{"t":0,"v":0.1},{"t":1,"v":0.3}]},
 "limits":{"maxSeconds":600,"maxCost":50}}
```

Mod → controller (out-of-band; distinguished by the `event` key so it isn't parsed as state):

```json
{"event":"control","action":"stopAfterTime","seconds":60}
{"event":"control","action":"stopAfterCost","gpuSeconds":25}
{"event":"control","action":"cancelGoal"}
```
