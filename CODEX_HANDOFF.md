# Handoff: finish the Jelly Drop mobile performance work

You are taking over an unfinished task from another AI agent (Claude Code) that stopped mid-task when it hit a usage limit. Roblox Studio is open on this PC with the place **"Ball Guys Game"** (game name "Jelly Drop", placeId 119248750585731). You are connected to it through the Roblox Studio MCP server. Use it to read and edit scripts, run Luau, start and stop playtests and run the test suites.

Read this whole file before you change anything.

---

## 1. The goal (from the owner's original request)

The game is a working single-player merge/physics game (drop jellies into a cup; two of the same tier merge into the next tier). It had bad mobile performance: input/network latency, physics and merge jitter, FPS drops and CPU spikes. GPU and memory were fine.

The owner asked for:

- **Gameplay that doesn't depend on ping.** Dropping, physics, collisions, merges, animations, VFX, SFX and UI must feel instant even at 200+ ms ping, so this gameplay runs on the client.
- **A server that stays authoritative** over coins, progression, inventory, upgrades, achievements, purchases, DataStore saving and loading, and persistent score. Never trust the client to report rewards; validate gameplay events on the server.
- **Lower CPU use** (less frame time and fewer spikes) without changing how the game looks or feels.
- **The same game.** Do NOT change gameplay rules, ball sizes, merge tiers, physics feel, scoring balance, progression, abilities, graphics, animations, UI, sounds or visual identity.

The task is complete only when all of these are true:
1. Existing gameplay still works correctly.
2. The visual and physics experience is the same or better.
3. Moment-to-moment gameplay no longer significantly depends on ping.
4. CPU usage and frame-time spikes are **measurably** reduced.
5. Mobile gameplay is noticeably smoother.
6. Persistent and economy systems remain secure and server-authoritative.

---

## 2. What the previous agent already did (all in the open Studio place)

**Architecture after the refactor:**
- **Device simulates the cup.** `MergeClient.LocalGame` handles drops, physics, merges, predicted combos, spill and pile detection, the merge-everything show and powers. It uses:
  - `Shared.JellyWorld`: an Instance-free port of the old server `BallWorld` + `SoftWorld`.
  - `Shared.SoftBody2D`: the optimized solver, verified bit-for-bit identical to the old one over 1500 substeps with 34 jellies.
- **Device reports events once per frame** with `Request:FireServer("Events", batch, round)`. The event codes are in `Shared.GameProtocol`: Drop, Merge, Spill, PileHigh, MergeAllDone, Ability, Dismiss and Forget.
- **Server `MergeServer.Session` is now a ledger and validator.**
  - A drop's tier must match the server's bag at that drop index.
  - Event rates are limited with token buckets.
  - A merge needs two same-tier balls that are in the ledger, and mass is conserved.
  - Combos are clamped (`ComboTracker:claim` with `Protocol.Grace`), and there's an area cap (`Protocol.AreaLimit`).
  - A rejection sends a "Ledger" resync: the device pops unknown balls and sends Forget.
- **Board attributes:** "Queue" (`round:base:tiers`) and "Tally" (`round:merges:score`). `LocalGame.view` overlays its predictions for Hud, Coach, BossFight, Celebrations, CupRenderer and AppShell.
- **ComboTracker** now lives in `Shared.ComboTracker`; the server `ComboTracker` is a shim.
- **AccountUpdate** pushes deltas (`{Delta = true, ...}`), and AppShell merges them.
- **Client cost cuts:**
  - Renderer: affine projection per frame, sub-pixel thresholds, allocation-free Vector3 contour math, and shape refresh capped at 30 Hz.
  - `Feedback` pools its parts and billboards.
  - `Hud.fitLayout` only runs when the layout signature changes.
  - Device physics has a per-frame budget (`world.budget = 0.010` in LocalGame, `maxSubsteps = 4`).
- **Studio-only hooks** in the client Bootstrap, via workspace attributes:
  - `StudioAutoDrop = n` makes the game auto-drop n jellies.
  - `StudioNoMerge = true` builds a pile that never merges.
- **Backup of every original script:** `ServerStorage.PerfBackup_2026_10_03`.
- **New suite `MergeServer.ProtocolTests`:** a device↔server harness at 0 ms and 250 ms links, plus cheat checks. 12 checks, all passing.
- **Test suites ported to the new harness `MergeServer.CupHarness`, all passing on the last run:**

| Suite | Checks |
|---|---|
| CoreTests | 31 |
| ComboTests | 35 |
| CelebrationTests | 20 |
| ReviveTests | 17 |
| MergeAllTests | 25 |
| BossTests | 32 |
| FreezeTests | 30 |
| ExpansionTests | 32 |
| MergeQueueTests | 26 |
| SoftBodyTests | 19 |
| ReleaseTests | 12 |
| ProgressionTests | 53 |

- **Manual playtest:** drops, merges, a 2x combo, the new-jelly unlock reveal and dismiss all worked, and server and device agreed on the score.

---

## 3. Paths and tools

| Name | Path |
|---|---|
| PROJECT | `C:\Users\Magzhan\OneDrive\Documents\BallGameClaude` |
| WORK (previous agent's working folder) | `C:\Users\Magzhan\AppData\Local\Temp\claude\C--Users-Magzhan-OneDrive-Documents-BallGameClaude\c892a841-191b-4146-9c58-f2405328f692\scratchpad` (if it's gone, look for a copy at `PROJECT\scratchpad`) |
| MEMORY (previous agent's notes, read these) | `C:\Users\Magzhan\.claude\projects\C--Users-Magzhan-OneDrive-Documents-BallGameClaude\memory\` (`MEMORY.md`, `jelly-drop-perf-architecture.md`, `jelly-drop-redesign-plan.md`) |

**What's in WORK:**
- `src\work\{Shared,MergeServer,MergeClient}\<Name>.lua` are the edited copies of the scripts.
- `src\live\` is a mirror of the place's scripts at sync time.
- `src\work\probes\client.lua` is the client measurement probe for the new build.
- `sync_server.py` serves `WORK\src`. Run it with `python sync_server.py 8766`; it listens on 127.0.0.1:8766, GET reads a file and POST writes one.
- `port_world.py` is described in task 4.

**The sync module in Studio** (`ServerStorage.DevTools.Sync`):
```lua
local Sync = require(game.ServerStorage.DevTools.Sync)
Sync.pull("MergeServer", "CoreTests")      -- loads src/work/MergeServer/CoreTests.lua into the place
Sync.refresh()                             -- swaps every ModuleScript for a fresh copy
print(Sync.run("MergeServer", "CoreTests")) -- runs a suite and returns a report
```
The folder argument is `Shared`, `MergeServer` or `MergeClient`. Run suites in the **Edit** data model, not during a playtest.

**Gotchas the previous agent hit:**
- The MCP command thread caches `require` results per ModuleScript instance. Always `Sync.pull` and then `Sync.refresh()` before running anything, or you'll test old code.
- The command thread can't see game globals, and you can't pass functions across VMs ("Attempt to load a function from a different Lua VM"). To control code that's running in a playtest, install a Script or LocalScript that exposes a BindableFunction taking string commands (that's how the `PerfCtl`, `ClientPerfCtl` and `StressCtl` probes worked).
- The test harness runs faster than real time, so anything on a wall clock needs the injectable clock (`LocalGame` already has one).
- Studio throttles rendering when its window isn't in front. **Keep Studio in front for every FPS measurement.**

**Before you do anything else:**
1. Copy WORK to `PROJECT\perf_work`. The Temp folder can be cleaned automatically. Work from the copy, and run `sync_server.py` from there (the sync server reads its own folder's `src`). Check that `http://127.0.0.1:8766/work/Shared/JellyWorld.lua` responds.
2. Studio is the source of truth. Before you edit any file in `src\work`, compare it with the script's current `Source` in Studio. If they differ, Studio wins: refresh the file from Studio first.
3. Save a safety copy of the place with File → Save to File As → `PROJECT\JellyDrop_before_codex.rbxl`.

---

## 4. Remaining work, in priority order (the owner has 1–2 days)

### Task 1 (most important): a fair before/after measurement

The main open question is whether the game is actually smoother now. Physics moved from the server to the device, so the device now does work it never did before. These are the numbers so far:

| Scenario | Metric | Old build | New build |
|---|---|---|---|
| S1: 70 auto-drops, natural merging | client FPS | 59.7 | 51.4 |
| | frame p99 | 19.7 ms | 32.2 ms |
| | spikes >33 ms | 1 in 100 s | 17 in 47 s |
| | render-step avg | 1.0 ms | 3.4 ms |
| | server step avg | 0.76–1.25 ms | **0.012 ms** |
| | server send | 13–25 kbps | **~5.3 kbps** |
| S2: 38-jelly pile, no merging, then 10 more drops (48 jellies) | client FPS | 45.1 | 31.1 |
| | frame p50 / p99 | 17.3 / 50.2 ms | 26.7 / 55.6 ms |
| | spikes >33 ms | 124 in 22 s | 183 in 13 s |
| | render-step avg | 5.7 ms | 9.0 ms |
| | device physics (`JellyWorld.simulate`) | (ran on server: avg 3.3 ms, max 22 ms) | 4.9 ms/frame, max 13.6 ms |

The old numbers were taken before Studio was brought to the front, and the probes differed, so this comparison is **not fair**. Still, it suggests the device side may be slower now. Find out for real:

1. Don't touch the main place for the old-build run. Save a separate local copy (`PROJECT\JellyDrop_OLD_measure.rbxl`) and restore the original scripts there from `ServerStorage.PerfBackup_2026_10_03`. Inspect how the backup is organized first.
2. Run S1 and S2 on both builds under identical conditions:
   - Studio in front, same window size, same device emulator setting.
   - Same drop count, drop interval (~0.62 s), x positions and random seeds.
   - Same probes. Check what `StudioAutoDrop` does in the new client Bootstrap and match its positions and timing in the old build's server-side auto-drop. The old build drops through `Session:request("Drop", x, round, now)` on the server.
3. Record: FPS; frame p50, p99 and max; spikes over 33 ms; render-step time; per-module client timings; server step time; send and receive kbps; and RemoteEvent counts per second.
4. Phones are several times slower than this PC, so treat device-side milliseconds here as optimistic.

**If the new build is slower on the device**, optimize the device side without changing the feel:
- **Physics.** Profiling at 48 jellies put most of a solver substep in collision (~51%) and constraints (~37%), with about 227 collide calls and 525 broad-phase candidates per substep.
  - Try a tighter broad phase (spatial hash or sorted sweep).
  - Skip sleeping-vs-sleeping pairs, and sleep more aggressively when nothing moves.
  - Avoid re-solving unchanged contacts.
  - Any solver change must stay bit-for-bit identical: compare against the original solver from the backup over a long recorded simulation, as the previous agent did.
- **Rendering.** `BallRenderer.update`/`pose`, `JellySurface.pose` and `SmoothPolygon.draw` are the next biggest costs.
- **The physics budget.** Check that the 10 ms per-frame budget isn't itself causing visible slow-motion or stutter.

Re-measure after each change. Report the numbers honestly, even when they're worse.

### Task 2: latency and gameplay test
- `ProtocolTests` already covers a simulated 250 ms link. Also do a real playtest with Studio Settings → Network → **Incoming Replication Lag = 0.25**.
- Check that these feel instant: dropping, merging, combos, powers (Freeze, Tall and the rest), revive, merge everything, the Developer boss, the new-jelly unlock reveal, game over and restart.
- An honest player must never get a Ledger resync (jellies popping). Score and coins must match the server.

### Task 3: save and rejoin test
- Earn coins and progress, stop the playtest, start a new one, and confirm everything loaded.
- In Studio, DataStores only work if Game Settings → Security → "Enable Studio Access to API Services" is on. Use only the owner's own test data.
- Confirm purchases and Developer Product rewards are still granted only by the server.

### Task 4: finish the test-suite port (where the previous agent stopped)
- `port_world.py` in WORK moves ContactTests, JellyTuningTests and PerformanceTests from the old server `BallWorld`/`Arena` to `Shared.JellyWorld`. It was **written but never run**.
  - Run it. It asserts exact matches, so if a match fails, fix the script, don't force it.
  - Then `Sync.pull` the three suites and run them.
- `PerformanceTests.lua` is generated new by that script and has never been run. Some of its checks look wrong: it compares `world.softTime` to the solver step, and it calls methods like `world:simulate` and `world:wakeAll` that may not exist. Read the real `JellyWorld` API and make each check test real behavior.
- Never weaken a test just to make it pass. If a test finds a real bug, fix the game code.

### Task 5: remove dead code
- Once nothing references them, retire `MergeServer.BallWorld`, `MergeServer.SoftWorld` and probably `MergeServer.Arena`. Also check for leftovers like `Shared.SoftBody2D_New`.
- Search **every script in the place**, not just `src\work`, before you remove anything.
- Move retired scripts to `ServerStorage.Retired_2026_10` instead of deleting them.
- Then run **every** `*Tests` module in `MergeServer`, including ProtocolTests, and do one playtest.

### Task 6: final cleanup and report
- Remove all measurement probes from the place: `PerfProbe*`, `ClientPerfProbe*`, `HudProbe`, `PerfCtl`, `ClientPerfCtl`, `StressCtl`, `SnapCtl`, `NetCtl`, `ProfCtl`, `ClientProbeSource` and any you add.
- Make sure the Studio-only hooks (`StudioAutoDrop`, `StudioNoMerge`, `StudioShowPage`) only work when `RunService:IsStudio()`.
- Run all suites again.
- Save the place to a file: File → Save to File As → `PROJECT\JellyDrop_perf_final.rbxl`.
- Write `PROJECT\PERF_REPORT.md` covering:
  - what changed and why;
  - the before/after table from Task 1 (identical conditions);
  - which RemoteEvents are left on the gameplay path;
  - what was tested and the results;
  - any known risks or anything still worse than before.

---

## 5. Rules
- Preserve the game exactly (see section 1). Don't touch tuning constants, sizes, tiers, scoring, visuals or sounds.
- **Never publish** to Roblox. The owner publishes after reviewing.
- Make changes in small steps, and run the relevant tests after each one.
- Don't delete `PerfBackup_2026_10_03` or the safety copies.
- Don't claim something works unless you tested it. Report failures and bad numbers plainly.
- If something needs a decision from the owner, such as a trade-off between smoothness and exact physics feel, stop and ask.
