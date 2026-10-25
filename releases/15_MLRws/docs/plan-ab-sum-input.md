# Plan: A+B summed record input (mono buffer, dual-mono output)

Status: plan only, nothing implemented. Base: local `main` (MLRws 1.2.1).
All line numbers refer to that tree.

## 1. Goal

Each track gets a third record source next to the existing two:

| Track source | Records into the (mono) buffer | Out 1 | Out 2 | Pan |
|---|---|---|---|---|
| A (today, `recorded_channel = 0`) | Audio In 1 | per pan | per pan | works as today |
| B (today, `recorded_channel = 1`) | Audio In 2 | per pan | per pan | works as today |
| **A+B (new, `recorded_channel = 2`)** | Audio In 1 + Audio In 2 | **full level** | **full level** | **ignored** |

- Buffers stay mono. No change to the ADPCM format, the flash layout, the
  page ring, the encoder, core-1 work or RAM.
- An A+B track always plays the same signal at full level (unity gain) on
  both outputs (dual mono). The pan gesture, pan hop and pan CV have no
  audible effect on it. The filter, volume, speed, reverse and loop keep
  working.
- Monitoring while an A+B track is armed or recording: both inputs summed,
  on both outputs.
- Still one recording at a time. Recording In 1 and In 2 to two tracks at
  once (stereo pairs) is out of scope. See the feasibility report.

## 2. Hardware feasibility (summary)

- Core 0: one add plus a clamp per sample on the record path, and one extra
  output add per A+B track. That is a handful of cycles out of the ~8000
  per-sample budget (192 MHz / 24 kHz).
- Core 1, flash, RAM: unchanged. The encoder and erase rate are the same as
  a normal mono recording.
- Verify with the perf build (§8). The numbers should not move.

## 3. Decisions (proposed defaults, confirm before coding)

| # | Question | Proposed default | Alternative |
|---|---|---|---|
| D1 | Input sum level | Saturating full sum `clamp(In1 + In2)`, then Knob X gain. Each input stays at unity; Knob X tames hot signals. | `(In1 + In2) / 2`: never clips, but each source is 6 dB quieter and Knob X can't boost it back. |
| D2 | 16-wide arm gesture | Hold one arm key (col 0 or col 1) and press the other on the same row → A+B. Tapping an unlit arm key switches to that single input; tapping a lit key disarms. | Cycle A → B → A+B by tapping the other key while armed. |
| D3 | 8×8 grids (one arm key) | Plug auto-detect: both inputs plugged and no explicit choice → A+B. Otherwise as today. | Always A or B on 8×8, never A+B. |
| D4 | Gridless | Both jacks plugged → A+B (today: In 1). One jack → that input. | Keep In 1. |
| D5 | 16-wide auto-detect with both plugged and no explicit choice | Keep today's behaviour (previous input kept). A+B only when chosen explicitly. | Auto A+B as on 8×8. |
| D6 | Sample-manager info string (`device_mode.c:237`) | Report the raw value (0/1/2) once the web app (`dessertplanet/MLRws-web`) is checked to accept 2. Until then keep `& 0x01`. | Always keep the mask. |

Why D2 doesn't clash with pan mode: `process_pan_gesture` (`main.cpp:2856`)
returns early when any track is armed (`main.cpp:2876`). Holding both keys
on an armed track's row is therefore free today. The A+B gesture applies
only when the track is already armed, or is being armed by the first of the
two keys. On an unarmed track, holding both keys still enters pan mode as
today.

## 4. Implementation steps

### Step 1. Constants and docs in `mlr.h`
- Add the following, and use them everywhere instead of raw 0/1:
  ```c
  #define MLR_CH_A   0u   /* Audio In 1 */
  #define MLR_CH_B   1u   /* Audio In 2 */
  #define MLR_CH_AB  2u   /* Audio In 1 + 2 summed, dual-mono out, pan ignored */
  #define MLR_CH_MAX MLR_CH_AB
  ```
- Update the field comments on `recorded_channel` in the header struct
  (`mlr.h:184`) and in `mlr_track_t` (`mlr.h:360`).
- Header compatibility: `recorded_channel` is a full byte in
  `mlr_track_header_t`, so the size checks stay valid. Older firmware reads
  2 as `2 & 0x01 = 0`, so an A+B track plays as an A track there. That is
  harmless and needs no magic bump.

### Step 2. Stop truncating the value (`mlr.c`, `main.cpp`, `device_mode.c`)
Replace each `& 0x01` with a clamp helper, `ch > MLR_CH_MAX ? MLR_CH_A : ch`
(an unknown value falls back to A):
- `mlr_set_recorded_channel` (`mlr.c:1011`: `channel &= 0x01`)
- header load (`mlr.c:1097` `mlr_init` scan, `mlr.c:1191` `mlr_rescan_track`)
- header write (`mlr.c:4265` `write_track_header`)
- track copy (`mlr.c:4367` header, `mlr.c:4382` RAM)
- pattern/recall replay (`mlr.c:2313` `MLR_EVT_CHANNEL`)
- recall snapshot (`mlr.c:2803`)
- `set_track_recorded_channel` (`main.cpp:2578`)
- `device_mode.c:237` according to D6
- The legacy converter (`mlr.c:4673`, `:4758`) passes the byte through
  unchanged. Old tracks are only ever 0/1. No change.

### Step 3. Record input (`main.cpp`)
- Grid modes (`main.cpp:674-684`): replace the 2-way pick with
  ```c
  if (rec_ch == MLR_CH_AB)      dry_in = sat12(audio_in1 + audio_in2);  /* D1 */
  else if (rec_ch == MLR_CH_B)  dry_in = (int16_t)audio_in2;
  else                          dry_in = (int16_t)audio_in1;
  ```
  `sat12` clamps to −2048..2047. `dry_in` feeds both recording
  (`:689-711`) and monitoring (`:947`), so monitoring picks up the sum
  automatically.
- Gridless record (`main.cpp:1734`): same three-way pick for `dry_in_l`.
- Gridless auto-pick at record start (`main.cpp:1715-1722`): implement D4.
  Both plugged → `MLR_CH_AB`.

### Step 4. Output routing: full level on both, pan ignored (`mlr.c`)
- `pan_update_gains` (`mlr.c:857`): if `tr->recorded_channel == MLR_CH_AB`,
  set `pan_gl = pan_gr = 32768` (unity) and skip the pan table. This one
  change covers:
  - the pan-on mixer path (`mix_track_route` `mlr.c:1625-1629`, which
    already has a `== 32768` unity fast path);
  - pan glides and pan CV (`mlr_set_pan_cv` → `pan_update_gains`). They
    still update `tr->pan`/`pan_pos`, but the gains stay at unity;
  - the monitor gains, which are latched from `pan_gl/pan_gr`
    (`main.cpp:935-936`).
- `mlr_set_recorded_channel` (`mlr.c:1008`): call `pan_update_gains(tr)`
  after every channel change, since the gains now depend on the channel.
  Also call it after the fade-swap commit at `mlr.c:1850`. The existing
  `pan_slew` smooths the gain jump on a playing track.
- `mlr_pan_default` (`mlr.c:879`): return 0 (centre) for A+B. The stored
  value doesn't matter audibly, but it keeps the LEDs and the hop logic sane
  if the track is later switched back to A or B.
- Pan-off mixer path (`mix_track_route` `mlr.c:1560-1563`): add an A+B
  branch, `*mixL += s; *mixR += s;`. Every runtime mode currently sets
  `mlr_pan_enabled = true` (`main.cpp:263, 267, 276, 364`), so this path is
  legacy, but it must stay correct.
- Steady fast path: check that `mix_track_steady_integer_dual`
  (`mlr.c:1632`) and the slow path both reach `mix_track_route` for
  routing, i.e. that neither has its own L/R routing. If either does, add
  the A+B case there.

### Step 5. Monitoring (`main.cpp`)
- Grid modes, pan on (`main.cpp:950-953`): already correct through the
  latched unity gains (Step 4).
- Grid modes, pan off (`main.cpp:954-958`): add `mon_ch_last_ == MLR_CH_AB`
  → add to both outputs.
- Gridless (`main.cpp:2037-2044`): A+B → add `rec_monitor_l` to both
  `outL` and `outR`.

### Step 6. Grid UI, 16-wide (`main.cpp`)
- `handle_rec_arm_col_press` (`main.cpp:2461-2512`), D2 logic. `other` is
  `column ^ 1`, and `other_held` is
  `grid.heldRowMask(track + 1) & (1u << other)`.
  - Track armed (or being armed in this gesture) and `other_held`
    → `set_track_recorded_channel(track, MLR_CH_AB, true)`, then return
    (don't disarm).
  - Track armed, current channel is A+B, plain tap on either key → disarm
    (both keys are lit; tapping a lit key disarms).
  - Otherwise unchanged: an unlit key switches input, a lit key disarms, an
    unarmed press arms with that column's input.
  - Record-switch path (`rec_pos`, punch-in at `main.cpp:2500-2503`): the
    first key starts recording on its single input; the second key while
    held switches the live recording to A+B. Within a few ms of pressing
    the keys "together" that is inaudible. Accept this and note it in the
    docs. Check that `dry_in` switching mid-recording doesn't click
    (`mon_ch_last_` handling, `main.cpp:678-681`).
- Pan hop (`main.cpp:2477-2488`): add `recorded_channel != MLR_CH_AB` to
  `hop_ok`, so a single key on an A+B row never "hops" pan.
- Pan mode on an unarmed A+B row: allowed but inaudible. Recommended: make
  `process_pan_gesture` ignore A+B rows, so the both-key hold does nothing
  and doesn't show a pan display that doesn't apply.
- Copy gesture interaction: an arm-key hold also starts the track-copy
  gesture (`copy_source_track_`, `main.cpp:3210`, `reset_copy_gesture`).
  Make sure the second arm key on the *same row* is treated as the A+B
  gesture and not as a copy target (copy targets other rows).
- LEDs (`main.cpp:3732-3756`): for A+B, both col 0 and col 1 at the
  selected brightness. While armed or recording, both carry the arm flash.
  When not armed, show both at full instead of the pan display, since pan is
  ignored.

### Step 7. Grid UI, 8×8 (`main.cpp`)
- No second arm key. D3: in `set_armed_track` (`main.cpp:2588-2612`), when
  `small_grid_ && !channel_user_chosen && a1 && a2` → `MLR_CH_AB`.
- 16-wide keeps today's auto-detect (D5).
- LED: col 0 unchanged (`main.cpp:3730`).

### Step 8. Docs and metadata
- `panels/grid.md` §"Record arm and input" (`:138-158`): add A+B, the
  gesture, dual-mono full-level output, pan ignored, LED states, and the
  8×8 auto rule.
- `panels/gridless.md` (Inputs / outputs sections): both plugged → A+B.
- `panels/sample-manager.md`: channel field may be 2 (if D6).
- Top-of-file comments in `main.cpp:27-29`.
- `info.yaml`: version 1.3.0 (or 1.2.2), `date-updated`, new UF2 path.
  Follow the repo's release conventions in `.github/copilot-instructions.md`.

## 5. Files touched (expected)

| File | Change |
|---|---|
| `mlr.h` | constants, comments |
| `mlr.c` | ~8 mask sites, `pan_update_gains`, `mlr_pan_default`, `mlr_set_recorded_channel`, `mix_track_route` pan-off branch |
| `main.cpp` | input pick (×2), monitor (×2), arm handler, hop, pan gesture, LEDs, auto-detect, header comment |
| `device_mode.c` | info string (D6) |
| `panels/*.md`, `info.yaml` | docs, version |

Estimated size: ~120-200 changed lines. One focused PR.

## 6. Risks

- **Gesture conflicts** (arm, copy, pan mode, punch-in): the highest-risk
  part. Test every combination in §7.
- **Clipping with D1 full sum** when both inputs are hot. Mitigation: Knob X.
  Fallback: switch to `/2`.
- **Hidden `& 0x01` or `== 1` checks** missed in Step 2. Before coding, grep
  `recorded_channel` and `mon_ch_last_` across all files; every hit must be
  either A+B-aware or provably fine.
- **Web app compatibility** (D6): keep the mask until checked.
- **Old firmware / downgrade**: an A+B track plays as A (left). Acceptable;
  document it.

## 7. Test plan (on hardware)

Build: `cmake -B build && cmake --build build` (needs the Pico SDK), flash the UF2.

1. **Regression A/B**: record to A and to B on a 16-wide grid. Each records
   the correct jack and plays per pan as before. Pan gesture, hop and pan
   CV still work.
2. **A+B record**: feed different signals into In 1 and In 2, arm with the
   two-key gesture, record. Playback carries both signals, at full level on
   Out 1 and Out 2.
3. **Pan ignored**: on an A+B track, the pan gesture, hop and CV In 1 pan
   have no audible effect, and the LEDs show both keys lit.
4. **Filter, volume, speed, reverse, loop** on an A+B track: all work.
5. **Persistence**: power-cycle. The A+B track is still A+B (header). Same
   after a scene recall, pattern playback, and copy to another track.
6. **Switching a playing track** A → A+B → B: no clicks.
7. **Monitoring**: armed A+B monitors both inputs on both outputs, and Knob
   X scales it.
8. **Punch-in**: switch Up, press both keys together: the recording is A+B
   with no audible start on a single input.
9. **8×8**: both jacks plugged with no explicit choice → A+B. One jack →
   that input.
10. **Gridless**: both jacks plugged → A+B, monitor on both outputs. One jack
    → as before.
11. **Copy gesture**: holding an arm key and pressing another row still
    copies. The same row's other key gives A+B.
12. **Sample manager**: the web app lists tracks and uploads/downloads with
    an A+B track present.
13. **Clipping**: two hot signals give the expected clamp; Knob X down
    removes it.
14. **Perf** (`-DMLR_PERF_PROFILING=ON`): `mlr_perf_process_sample_max_us`,
    `..._ui_max_us` and `..._overruns` show no meaningful change against a
    1.2.1 baseline while recording A+B with 6 tracks playing.

## 8. Order of work

1. Grep audit of `recorded_channel` / `mon_ch_last_` (confirm the §4 site list).
2. Steps 1-2 (constants, masks). Builds and behaves identically, since
   nothing sets 2 yet.
3. Steps 3-5 (audio path). Test by forcing a track to A+B in code or via the
   8×8 / gridless auto rule.
4. Steps 6-7 (UI).
5. Step 8 (docs, version) → tests §7 → PR from the fork.

## 9. Part B: input monitor off

### Why
The card has no analog through-path: the monitor is the input digitized at
24 kHz/12-bit, with the 6 kHz mux notch (`ComputerCard.h:546, 711-713`). To
hear the untouched source, split it outside the card (mult or stackable
cable) to the mixer. The card must then stop adding its own dry copy.
Today Knob X is both the record gain and the monitor level
(`main.cpp:686`, `:947`), so the monitor can't be silenced without also
silencing the recording.

### Behaviour
- New global setting `mlr_monitor_enabled` (default **on** = today's
  behaviour).
- **Off**: no input is mixed into the outputs while armed, recording or with
  the switch Up. Knob X still sets the record gain. Recording, A/B/A+B
  routing and playback are unchanged.
- Applies in every mode (grid and gridless), because it lives in the shared
  scene settings.

### Code
- `mlr.h` / `mlr.c`: `extern volatile bool mlr_monitor_enabled;`.
- Grid modes, `main.cpp:927`: the monitor fade-in condition becomes
  `mlr_monitor_enabled && dry_level > 0 && (rec_armed_track >= 0 || sw == Switch::Up)`.
  When the setting is switched off mid-monitor, the existing fade-out branch
  (`:941-945`) ramps it down click-free.
- Gridless, `main.cpp:2037-2044`: add `mlr_monitor_enabled` to the
  condition. Ideally route it through a short fade as in grid mode. Today
  gridless adds `rec_monitor_l` with no fade, so toggling while recording
  could click. Acceptable, since the toggle is a grid gesture.
- Persistence: append a versioned trailing block to the scene,
  `mon_version (uint8_t) + mlr_monitor_enabled (uint8_t)`, after the cvt
  block in `scene_serialize` (`mlr.c:3026`). Load it only when `data_len`
  covers it, as the other trailing blocks do (`mlr.c:2950-2952` comment), so
  old scenes load with monitor on. Update the layout comment
  (`mlr.c:2929-2952`) and the size enum. Mark the settings dirty on toggle
  (`note_settings_changed`, `main.cpp:2896`) so it saves within ~4 s.
- Exclude it from the 5-second DELETE reset (it's a setup preference, like
  master level), and from patterns and recalls.
- Optional build default: CMake option `MLR_MONITOR_DEFAULT_OFF` (default
  OFF) sets the initial value when no scene block exists. Handy for a
  personal fork build.

### Toggle gesture (decision D7)
| Option | How | Notes |
|---|---|---|
| **Proposed: grid combo** | Hold DELETE + press the REC page key (row 0, col 0). The key flashes: 3 fast = monitor on, 1 long = off. | Checked: the row-0 handler (`main.cpp:718-726`) ignores `alt_held` on col 0, so today the combo only switches to the REC page. Branch on `alt_held` there to toggle instead (no page switch). Row-0 chrome is shared, so it works on 8×8 too. |
| Web sample manager | Checkbox in the web app | Needs a protocol + web app change in another repo. |
| Build flag only | `MLR_MONITOR_DEFAULT_OFF` | No runtime toggle. |

Gridless has no free panel gesture. It uses the saved setting (set it once
on a grid, or with the build flag).

### Tests
1. Monitor on (default): identical to 1.2.1.
2. Toggle off: arm, Switch Up, record → no dry signal on the outputs.
   Recording level still follows Knob X, playback is normal.
3. Toggle while monitoring: no click (grid mode).
4. Power-cycle: setting kept. An old scene without the block loads with
   monitor on.
5. Gridless with monitor off: recording works, no dry signal on the outputs.
6. DELETE 5-s reset doesn't change it. Patterns and recalls don't touch it.

Size: ~40-60 changed lines. Can ship before, with, or after Part A
(independent).

## 10. Later (not in this plan)

- Optional In 1/In 2 balance for A+B (e.g. Knob Y while armed).
- Stereo track pairs (simultaneous In 1 → track N, In 2 → track N+1). These
  need hardware measurement first (erase dead time on core 1); see the
  stereo/overdub feasibility report.
- Overdub.

## 11. Part C: overdub (in place, 6 tracks, no undo)

Chosen variant: **rewrite the track in place**. All 6 tracks keep their full
26.3 s. No flash re-partition, no migration, no web-app change, no undo
(the scratch-slot variant was rejected: it costs track length or a track).
Implement after Parts A/B are merged and tested on hardware.

### C.1 What the user gets
- **Record mode: Replace (default, today) / Overdub.** Toggle with **DELETE +
  CUT page key** (row 0, col 1). It mirrors the monitor toggle on col 0, with
  the same flash code (3 fast = overdub on, 1 long = off). The setting is
  saved in the scene (new versioned trailing block, like Part B). Gridless
  uses the saved value.
- With Overdub on, recording a track that **already has audio** overdubs it.
  Empty tracks record normally. Any source works (A, B, A+B).
- Arming does **not** stop the track (today arming stops it).
- Start (switch Up/Down, or Pulse In 2 gate):
  - track playing → the pass starts at the next wrap to sample 0
    (quantized to the loop start; the arm LED blinks "waiting");
  - track stopped → playback starts from 0 and the pass starts at once.
- While recording is held, passes continue back-to-back, wrap after wrap.
  Each wrap starts a new pass over the result of the previous one.
- Stop: the input is muted and the current pass runs silently to the end of
  the track, then commits. If the user cuts or stops the track before the
  end, it commits at once (see C.3, junction).
- Feedback: the old audio is scaled by `fb` before adding the input. The
  default is 0.9 (Q8 230), a build-time constant at first. An adjustable
  control comes later (C.7).

### C.2 Why it is a whole-track pass
The audio is IMA ADPCM: every nybble depends on the decoder state left by
all the previous ones, and a flash erase is 4 KB (8192 samples). A pass
therefore rewrites the stream from sample 0 to `length_samples`, in order,
with a fresh encoder (state 0,0, which is always keyframe 0). The wrap
already resets the decoder to keyframe 0 (`seek_fill_to(tr, t, wrap_start)`,
`mlr.c:3580`), so pass after pass stays bit-consistent.

Constraints on the overdub track during a pass (enforced, not just
documented):
- speed exactly 1× (`speed_frac == 256`), forward, no loop-a-section, no
  rotation/seam, not gate mode; CV1 pitch/speed modulation is ignored for
  this track;
- cuts on this track (grid, patterns, recalls, group broadcasts) are ignored
  while the record is held; after stop they end the pass (commit with
  junction);
- the track stays in its group, but group cuts skip it during the pass.
Arming for overdub is refused (the LED shows "can't") if speed ≠ 1×,
reverse, or loop is on. Alternative: auto-reset them, decided at
implementation.

### C.3 Core-0 data path (exact sample alignment)
The record section of `ProcessSample` (`main.cpp:672-711`) runs **before**
the mixer (`:899`). Encoding there from the "old" sample would be one sample
late and drift by one sample per pass. Instead:
- `main.cpp` stores the scaled input for this sample:
  `mlr_overdub_input = scaled` (`int16_t`, Knob X applied, A/B/A+B already
  picked). In overdub mode it does **not** call `mlr_record_sample`.
- In the mixer (`mix_track_steady_integer_dual`, `mlr.c:1632`), for the
  overdub track only: take the raw ring sample being consumed this output
  sample (`tr->pcm.buf[ring_pos]`: pre-volume, pre-filter, before the wrap
  crossfade), then
  `new = sat12((old * fb >> 8) + input)` → `mlr_record_sample(new)`. At 1×
  with `speed_accum == 0`, exactly one stored sample is consumed per output
  sample, so alignment is exact. If the slow path is ever taken for the
  overdub track (volume slewing, transition flags), it must apply the same
  logic. Simplest: force the fast path's conditions (no volume slew on that
  track during the pass) and abort the pass if it fails.
- The wrap crossfade (`maybe_start_wrap_preview_xfade`) mixes preview audio
  into the *output* only. The value fed to the encoder must be the raw ring
  sample, so the crossfade is never baked in.
- At each wrap (`note_track_wrap`): finish pass n (keyframes complete),
  reset the encoder to (0,0) and keyframe 0, set `rec_samples = 0` and
  `rec_flash_offset` = track audio start, and begin pass n+1 (or end the
  session).
- Cost: ~150 cycles/sample (one mult, add, clamp + the normal encoder),
  ~2% of the budget.

### C.4 Core-1 flash path: the read-before-erase rule
Playback refills read the **old** data ahead of the playhead, while the pass
writes **new** data behind it, in the same flash region.
- **Erase rule** (`flush_rec_page`, `mlr.c:4215`): before erasing sector S
  of the overdub track, the refill must already have read past it:
  `fill_byte_pos >= S_end` (on the same pass). Otherwise hold the erase. No
  fixed lag constant is needed; the check is exact. The refill runs up to
  8192 samples = 4096 B = one sector ahead, so the page ring needs
  ~16-17 pages of backlog in steady state. It has 32.
- **Don't force the erase**: the existing "urgent" override
  (`page_ring_used_pages() >= 16`, `mlr.c:4224`) and the 100 ms hold-off cap
  must not apply to the overdub read rule (erasing unread data destroys
  audio). Raise the urgent threshold for overdub to ~28 pages. If the ring
  still fills (core 1 starved), abort the pass and commit with a junction.
  Never overrun.
- **Page-ring overflow guard**: `mlr_record_sample` (`mlr.c:1297`) writes
  without checking for a full ring. Add a full check (count drops in perf)
  for both normal recording and overdub.
- **Reading back new data**: on pass n+1 the refill reads pass-n bytes from
  flash. Those bytes must already be programmed: the track must be longer
  than the refill lead + page backlog. v1 rule: **overdub needs
  `length_samples >= 2 s` (48000 samples)**, else refuse. v2 (C.7): serve
  unflushed bytes straight from the page ring, for short loops.
- **Keyframes during the pass**: within a pass the refill decodes
  sequentially and wraps via keyframe 0, so flash keyframes aren't needed.
  The session refuses cuts on that track. `rec_keyframes` (`mlr.c:328`) is
  rebuilt each pass.
- **Header**: written **once** at session end (`write_track_header` with
  merged keyframes, see junction). That's one header erase per session, not
  per pass (wear).
- **Power-loss safety**: at session start, rewrite the header once with a
  "keyframes stale" flag. Use `recorded_channel` bit 7 (old firmware masks
  `& 0x01` and ignores it; Part A's clamp must mask `& 0x7F` first). At
  boot, a track with the flag gets its keyframes rebuilt by decoding it from
  0 (≈ 630k samples × ~80 cycles ≈ 0.3 s worst case, `MLR_COLD_FUNC`). Then
  the flag is cleared. Interrupted audio gets at most one junction glitch.
- **Erase budget**: overdub erases at the same rate as a normal recording
  (1 sector / 341 ms), so the core-1 load is the same as recording today.
  This does **not** need the stereo measurement. The existing playback
  `erase_hold_off` still applies for the *other* tracks.

### C.5 Stop and the junction
- Normal stop: the input is muted and `fb` switches to 1.0 for the rest of
  the pass (a pure re-encode, so the untouched part isn't decayed; it gets
  one extra ADPCM generation). The pass ends at the wrap and the header
  commits: `num_keyframes` and keyframes come from `rec_keyframes`.
- Early commit (user cut/stop, ring starvation): the stream is new for
  `[0, x)` and old for `[x, end)`. The keyframes are merged: new ones up to
  `x`, old ones (copied from the flash header) after it. Seeks are then
  exact everywhere; only continuous playback across `x` hears a short
  ADPCM-state glitch. `write_track_header` (`mlr.c:4244`) needs a "merge"
  mode: copy the flash header, overwrite the first n keyframes.

### C.6 Quality and wear
- Each pass adds one ADPCM generation, roughly −35..−40 dB noise. With
  `fb = 0.9` the accumulated noise is bounded (~+7 dB over one generation).
  This is the main audible cost. Document it.
- Wear: one erase per sector per pass. A 4 s loop overdubbed continuously
  ≈ 110 h per track to the 100k-cycle datasheet limit; shorter loops wear
  faster. Fine for gesture-driven overdub. Document it.

### C.7 Phases
1. **C1, single-pass core**: the mode toggle and its persistence; the
   core-0 encode in the mixer; the read-before-erase rule; the overflow
   guard; whole-track passes; stop = silent finish; header at the end;
   ≥ 2 s rule; enforced constraints.
2. **C2, continuous passes** while record is held, plus the junction commit
   on cut/stop/starvation.
3. **C3, power-loss flag** and keyframe rebuild at boot.
4. **C4, adjustable feedback** (e.g. Knob Y while an overdub track is armed:
   check against the existing Knob Y roles, release/attack). Saved per
   scene.
5. **C5, short loops** (< 2 s): serve unflushed bytes from the page ring
   in `fill_pcm_ring_forward` (`mlr.c:3591`).

### C.8 Files
- `mlr.c`: record state (`:323-331`), `mlr_start_record` overdub variant (no
  `has_content = false`, no pcm reset), a mixer hook, `note_track_wrap`
  pass rollover, `flush_rec_page` read rule + urgent threshold,
  `mlr_record_sample` overflow guard, `write_track_header` merge, a boot
  keyframe rebuild, the scene block.
- `main.cpp`: the toggle (row 0 col 1 + `alt_held`), the arm path (don't
  stop the track; refuse if constraints fail), the record start/stop paths
  (`:586-651`, `:703`, `:1723/1762`, `:2501`, `:2931-2938`, `:3372`), the
  input handoff, cut-ignore checks, LEDs (waiting/overdubbing state),
  gridless.
- `mlr.h`: API + flags. Docs: `panels/grid.md`, `panels/gridless.md`.
- Estimated 400-600 lines. High risk → test with the perf build: S5
  scenario plus overdub on 1 track while 5 others play at 4× reverse.
  `pcm_underruns` must be 0, and add an `overdub_erase_holds` counter.

### C.9 Tests (hardware)
1. Overdub off: identical to before (regression).
2. Overdub a 4 s loop 1×: the pass starts at the wrap, the new input layers
   on, stop finishes silently, the result persists after a power cycle.
3. Hold record for 10 passes: no drift (a click layered at the same spot
   stays aligned), noise growth acceptable, decay with fb 0.9 audible.
4. Refusals: speed ≠ 1×, reverse, loop-a-section, length < 2 s.
5. Cut / pattern / recall / group cut during the pass: ignored; after stop →
   junction commit, seeks still clean.
6. Stress: 5 other tracks at 4× reverse while overdubbing: no underruns, no
   corrupt audio (compare a re-encode of silence with fb 1.0: should be
   near-identical).
7. Power-pull mid-pass → boot rebuilds keyframes, the track plays (one
   glitch max).
8. A+B source and monitor-off combinations.
