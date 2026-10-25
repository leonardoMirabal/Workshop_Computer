# MLRws: test rig and iteration procedure

Status: plan, nothing built yet. Branch `mlrws-test-rig` (from `mlrws-overdub`).

## 1. Goal

A loop where Claude can change firmware, flash it, drive it, listen to it and
profile it on the real card without anyone pressing keys:

```
edit -> build -> flash -> boot check -> drive (virtual grid) -> record audio (MOTU)
     -> read counters -> analyse -> report -> next edit
```

Target: under 90 s from "build" to the first test result, and every test run
leaves a report that can be compared with the last one.

Two rules for every change:

1. **Grid mode works the norns way.** The default is: monitor always on, and the
   arm key records (Direct flow). An empty track starts on the first audio. The
   whole set works from the grid alone.
2. **Gridless mode must not change.** The switch flow and its LEDs and timings
   stay as they are in 1.2.1. Every change runs the gridless regression suite
   (§6.4) before it is pushed.

## 2. What the card already gives us

Checked in the code. These need no new firmware:

| Need | Already there |
|---|---|
| Drive it like a grid | **Device mode** (`Mode::DeviceMLR`). When the card's USB is plugged into a computer, the card treats the computer as the grid and speaks mext over the USB serial port. This is the path norns' gridproxy uses. The Mac acts as the grid. |
| Read track audio exactly | **Sample-manager mode** (`'I'`, `'R'`, `'W'`, `device_mode.h`). Download a track's ADPCM bytes and header straight from flash, to check overdub results bit for bit. |
| Profiling counters | The `MLR_PERF_PROFILING` build: `process_sample_max_us`, overruns, PCM underruns, flash erase/program times, page-ring use and the overdub counters. |
| Scripted actions | `mlr_perf_cmd`: play, stop, rec start/stop, loop, copy, pan, LPF, CV and overdub. Today it can only be written over a debug probe (OpenOCD). |

Limits found in the code:

- **Hardware.** Device mode only exists on **Rev 1.1** hardware
  (`USBPowerState()` returns `Unsupported` otherwise, which means host mode).
  Check the card revision first.
- **Boot timing.** The mode is chosen at boot. The card waits about 2 s for the
  computer to open the port:
  - the first byte is one of `I S R E W P X`: sample manager;
  - silent, or a mext byte: grid mode;
  - nothing connected: gridless.

  The rig keeps the port open and reconnects at once after every reboot.
- **One protocol per boot.** Switching between the grid and the sample manager
  needs a reboot.

## 3. Bench setup (one time)

1. **Card revision.** Confirm the card is Rev 1.1.
2. **Audio patching:**
   - MOTU out 1 → card Audio In 1, and MOTU out 2 → card Audio In 2.
   - Card Audio Out 1 → MOTU in 1, and card Audio Out 2 → MOTU in 2.
   - Card outputs are modular level, about ±6 V. Set the MOTU input gain or pads so a full-scale card output doesn't clip the MOTU.
   - MOTU at 48 kHz, its current setting.
3. **USB.** Card USB → Mac. The real grid is **not** on the card. It can go on the Mac instead (§4.1, pass-through).
4. **Knob X** at a marked position. Once the test hooks exist (§5), the rig overrides it in the perf build.
5. **Tools on the Mac:**
   - Python packages `pyserial`, `sounddevice`, `numpy`, `scipy`;
   - `sox`;
   - the ARM toolchain and pico-sdk, already in the scratchpad. Move them to a stable path, e.g. `~/dev/toolchains`, so they survive the session.
6. **Microphone permission.** macOS asks once for the terminal to use the MOTU inputs.
7. **Baseline backup.** Download all 6 tracks and the scene with the sample manager, and keep the copy outside the repo. Every destructive test starts from this known set and can restore it.

## 4. Host tools (`releases/15_MLRws/tools/rig/`, Python)

### 4.1 `vgrid.py`: virtual grid

- Answers the card's mext queries as a 16×8 grid. With `--size 8x8` it answers as 8×8, so both layouts can be tested.
- Keeps a mirror of all 128 LED levels from the card's LED messages. It can wait for a key to reach a state ("blinking", "lit ≥ 12", "dark") with a timeout.
- Gives a script API: `press(x, y)`, `release`, `tap`, `hold(ms)`, `chord(...)`.
- **Pass-through mode.** A real grid plugged into the Mac is forwarded to the card, so you can play by hand while the rig logs every key and LED. It can also inject scripted presses alongside yours. Same feel as norns + gridproxy.
- Logs every key and LED event with timestamps to the run folder.

### 4.2 `audio.py`: MOTU I/O

- Plays and records at the same time on the MOTU: test signals out to the card, both card outputs in.
- **Signals:**
  - sine tones at 100 Hz, 1 kHz and 5 kHz;
  - log sweep, 20 Hz–12 kHz;
  - pink noise;
  - click trains for timing;
  - silence for the noise floor;
  - a trigger burst for the auto-record threshold.
- **Analyses:**
  - level and noise floor;
  - SNR and THD+N;
  - frequency response;
  - click and dropout detector, a sample-to-sample jump test against a smooth reference;
  - latency, by cross-correlation;
  - overdub pass gain: each pass's level against the dub setting;
  - seam detector at loop wraps.
- **Clock drift.** The MOTU and the card don't share a clock. Analyses align by cross-correlation per section and never assume a sample-exact timeline.

### 4.3 `card.py`: card control

- `flash(uf2)`: sends the reboot-to-bootloader command (§5), waits for the `RPI-RP2` drive, copies the file, then waits for the serial port to come back.
- `mode(grid|samplemgr)`: reboots and opens the port with the right first byte.
- `perf_read()` and `perf_reset()`: counters as a dict.
- `perf_cmd(...)`: the existing scripted actions.
- `override(knob_x=..., switch=...)`: Knob X and switch overrides (§5).
- `track_dump(n)`: header and ADPCM bytes. A host-side IMA decoder that matches `adpcm.h` turns them into a WAV.

### 4.4 `run.py`: suites and reports

- `run.py <suite> [--uf2 file]` does three things:
  - flashes, if a file is given;
  - checks boot, which mode it entered and the firmware version;
  - runs the suite.
- Each run gets a folder `runs/<date>-<git sha>-<suite>/`. It holds the WAVs, the key and LED log, the counters, `report.md` and `result.json`.
- `compare.py <run A> <run B>` diffs counters and audio metrics. It is the "did this change make it worse" check.
- `runs/` is git-ignored.

## 5. Firmware test hooks (perf build only)

Normal builds don't change. All of these sit under `#ifdef MLR_PERF_PROFILING`
and run on core 1, in the USB/serial task, so they cost core 0 nothing.

1. **Rig command channel in grid (mext) mode.**
   - It uses a mext header the parser doesn't use; pick and verify the byte in `monome_ws.c` when building it.
   - Commands:
     - `PERF_READ`: returns a fixed binary block of all counters and a version number;
     - `PERF_RESET`;
     - `PERF_CMD`: writes `mlr_perf_cmd` exactly like OpenOCD does and returns the ack;
     - `OVERRIDE`: Knob X value, switch position, or back to the real control;
     - `REBOOT_BOOTSEL`: `reset_usb_boot(0, 0)`;
     - `REBOOT`: watchdog reboot.
2. **Process-time histogram.** Count each `ProcessSample` duration in 2 µs buckets from 0 to 64 µs. The 24 and 32 kHz decisions need the distribution, not only the max.
3. **Core-1 loop time max and erase-hold count,** alongside the existing flash timers.
4. **A version tag in the counters block,** so `card.py` refuses to read a layout it doesn't know.

Size check: the perf build is at 97% RAM (254,300 B). Budget at most 1.5 KB for
the hooks. If they don't fit, move the counter readout to flash-resident code,
since core 1 may run code from XIP.

## 6. The iteration procedure

### 6.1 Every change (the inner loop, Claude runs it)

1. **Branch.** One feature branch per change, cut from the branch it builds on. Never main; push only to the `fork` remote (Actions stay off).
2. **Build both variants** (normal and perf).
   - Record the RAM and FLASH sizes in the report.
   - Fail on any new warning.
   - Restore the tracked `UF2/MLRws.1.2.1.uf2` after the build.
3. **Flash the perf build** with `card.py flash`.
4. **Smoke suite** (§6.3, under 1 min). Stop here if it fails.
5. **Feature suite** for what changed.
6. **Gridless regression** (§6.4) if anything in `main.cpp` record/monitor/flow code, `mlr.c` recording or the scene changed. In practice that's almost always.
7. **Profile** (§6.5) if anything on core 0's sample path or core 1's flash path changed.
8. **Compare** with the last good run of the same suites.
   - A metric that got worse past its threshold (table in §6.5) blocks the change.
9. **Flash the normal build** and repeat the smoke suite.
10. **Commit, push to the fork, report:**
    - what changed;
    - suite results;
    - size and CPU deltas;
    - anything that needs ears (§6.6).

### 6.2 When you are needed

- **One time:** setup and baseline (§3).
- **If the card hangs** and the bootloader command can't reach it: hold BOOTSEL and replug, or power-cycle the rack.
- **Listening checks** (§6.6), batched: one listening session per feature, not per change.
- **Decisions:** UI behaviour, defaults, whether a measured cost is worth it.

### 6.3 Smoke suite (every flash)

- **Boot:** it enters the expected mode, reports the version, and the counters read.
- **Grid basics:**
  - row 0 navigation draws;
  - a cut on a track with audio plays it, and audio appears on the right output;
  - stop works.
- **Record:** a 3 s Replace recording on a scratch track; the track then plays back what was sent (by correlation).
- **Counters:** 0 PCM underruns and 0 overruns over the suite.

### 6.4 Gridless regression (1.2.1 behaviour must hold)

Run with the computer connected but silent past the detection window, or with
the virtual grid off, so the card boots gridless. Switch actions use the
switch override (§5).

- **Arm:** hold Down 2 s, and the LEDs fill one to six. Armed means the slow flash. Up cancels; Down records, with the fast flash.
- **Monitor:** while recording, the monitor goes to the track's output (Out 1, Out 2, or both for A+B). Knob X sets record gain and monitor level.
- **Stop:** release stops the recording; after about ½ s the track loops from the start.
- **Auto-stop** at the maximum length.
- **Saved settings:**
  - with the scene's monitor setting Off, nothing from the inputs reaches the outputs;
  - Always and Armed behave the same here;
  - the Direct flow setting and Overdub mode saved from grid mode are ignored in gridless mode.
- **Sample manager.** It still enters sample-manager mode when the web app's first byte arrives.
- **Timings.** The LED timings and the ½ s settle are compared with the reference recordings made on 1.2.1 in the baseline run.

### 6.5 Profiling protocol

**Scenarios:** each runs for 60 s, after `PERF_RESET`.

| ID | Load |
|---|---|
| P0 | idle, 6 tracks with audio, none playing |
| P1 | 6 tracks at 1× |
| P2 | 6 tracks at mixed odd speeds (⅔, 3/2, ¼, 4×, reverse), filters on, pan on |
| P3 | P2 on tracks 1–5 + a Replace recording on track 0 |
| P4 | P2 + overdub on track 0 (5 others at 4× reverse), 20 passes |
| P5 | P4 + the grid rig pressing cuts every 50 ms and a scene save every 4 s |
| P6 | P4 with a 2 s loop (worst erase rate) |

**Recorded per scenario:**

- `process_sample_max_us`, the histogram, 99.9th percentile and overruns;
- PCM underruns per track;
- page-ring max and drops;
- erase holds and forced erases;
- flash erase and program max;
- the overdub counters;
- core-1 loop max.

**Pass rules at 24 kHz:**

- **Hard limits:** 0 overruns, 0 underruns, 0 page drops, 0 overdub overflows, starves and junctions.
- **Max time per sample:** ≤ 38 µs, under the 41.7 µs budget.

**32 kHz decision:** worst-scenario max ≤ 28 µs at 24 kHz, with the 31.25 µs
budget at 32 kHz. Per-sample work hardly depends on the rate. Core-1 load scales
by 1.33×, so P4 and P6 must also show `ring_max` ≤ 18 pages before 32 kHz is
tried.

**Regression thresholds:**
- `process_sample_max_us` more than 1 µs or 3% worse than the last good run;
- any new underrun;
- RAM more than 512 B worse without a reason in the report.

### 6.6 Listening checks (you, batched)

The rig flags them; the machine can't judge them:

- monitor fades;
- the A+B image;
- overdub pass character;
- auto-record pre-roll feel;
- the norns-like workflow overall.

Each check comes with the WAVs from the run and one or two sentences on what to
listen for.

## 7. Defaults: norns way on the grid, gridless unchanged

Code check, done now:

- **Flow.** Gridless never reads `mlr_rec_flow`.
- **Monitor.** It only checks `mlr_monitor_mode != MLR_MON_OFF` while recording, so Always and Armed are the same there.

So the defaults can change without touching gridless:

- `mlr_rec_flow` default → **Direct**;
- `MLR_MONITOR_DEFAULT_MODE` → **2 (Always)**.

Scenes saved by earlier builds keep their saved values. A fresh card, or a
scene without those blocks, gets the norns way. The gridless suite (§6.4)
verifies this when the change is made.

## 8. Order of work

| Step | What | Who | Done when |
|---|---|---|---|
| R0 | Bench setup, revision check, baseline backup and reference recordings on 1.2.1 | you + Claude | baseline run folder exists |
| R1 | `vgrid.py` + `card.py` (grid mode only) | Claude | smoke suite drives the card |
| R2 | Firmware hooks (§5) | Claude | `card.py flash` and `perf_read` work without BOOTSEL |
| R3 | `audio.py` and analyses | Claude | smoke suite checks audio by correlation |
| R4 | Sample-manager dump + host ADPCM decoder | Claude | a dumped track matches the played audio |
| R5 | Suites: smoke, gridless, A+B/monitor/Direct (plan §7, §9, §15), overdub (§C.9), profile | Claude | all suites run, reports compare |
| R6 | Defaults change (§7) through the full procedure | Claude | first change done the new way |
| R7 | Then: Part D quality work, the 32 kHz decision | Claude + you | per §6 |

Until R2 is done, each flash needs BOOTSEL by hand. After that, a hang is the
only reason to touch the card.

## 9. Risks

- **Rev 1.0 card:** no device mode, so no virtual grid. Fallback: debug probe and OpenOCD for commands and counters, and the real grid for keys.
- **Missing the boot window:** the 2 s detection window could be missed after a reboot. `card.py` polls the port every 50 ms. If it still misses, add a perf-build-only longer window.
- **Clipping:** MOTU input clipping hides real distortion. The baseline run checks the headroom with a full-scale tone.
- **Hooks don't fit in RAM:** move the readout to XIP on core 1 (§5).
- **Flash wear from the rig:** about 1 flash per change, plus stress tests. Overdub stress (P4/P6) wears the track sectors; it runs only on a scratch track, and the plan doc's wear figure (about 110 h for a 4 s loop) applies.
