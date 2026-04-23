# PROJECT — Spectrum Advisor

## Overview
Real-time spectrum analysis and mix-coaching tool for Drum & Bass producers. Runs alongside Ableton Live 10, listens to the master bus via BlackHole, and compares the live mix against a pro reference track — visualizing frequency deviations, loudness, dynamics, and giving actionable coaching. Built specifically to compensate for closed-back headphone monitoring limitations by making sub-bass and low-mid masking **visually obvious**.

## Owner
Dorian Maissin (dorian.maissin@gmail.com)

## Versions
- **v1** (current, discovery complete): core real-time analysis + single-reference comparison + energy-gated drop detection.
- **v2** (planned):
  - Stem/element-level comparison (drums/bass/synths analyzed separately vs reference)
  - Manual section segmentation (intro / breakdown / drop1 / drop2 each with their own comparison profile)
  - **Spectral masking detector** — detects element-on-element frequency collisions (kick vs sub vs bass fighting in the same range) → direct attack on the mud/loudness problem
  - **Saturation / harmonic density comparison** — measures harmonic content vs reference, tells user when to apply saturation for perceived loudness without adding dB
  - **Sub-bass consistency tracker** — detects drift/pump in sub level across track sections (user's blind spot on SRH440)
  - Possible Max for Live port
- **v3+** (vision): personalized perceptual headphone calibrator (user-specific A/B test tones, goes beyond AutoEQ's generic measurement).

## Target User Profile
- DnB producer (deep + liquid subgenres: Alix Perez, Amoss, Minor Forms / Netsky, Satl, Calibre, Lenzman, Tokyo Prose)
- Works in Ableton Live 10 Suite on Mac M4, 16GB RAM
- Mixes on Shure SRH440 + AutoEQ correction via eqMac
- Core blind spot: sub-bass (physical driver limitation) — tool's #1 job is to visualize what he can't hear

## Scope (v1)

### Must-have features
1. **Spectrum comparison** — live master vs reference, colored hot/cold heat map (functional first, polish later)
2. **Reference library** — folder-based, sidebar picker, click-to-switch mid-session, remembers last used between sessions
3. **File formats** — WAV, AIFF, FLAC, MP3 (all via `librosa`). 320kbps MP3 fully acceptable for reference purposes.
4. **Loudness panel** — LUFS-I, LUFS-S, True Peak, with targets (**−4 LUFS-I target**, acceptable floor −6)
5. **Dynamics panel** — Crest factor, PLR, LRA (EBU R128), pump detection per band
6. **Transient punch analysis** — per-element transient comparison (kick, snare) vs reference. Combined with drum-band spectrum energy, answers "are my drums too loud / too punchy / too weak" — canned diagnoses covering: drum bus overall level, kick/snare independently, and punch vs body balance.
7. **Stereo / phase panel**:
   - Phase correlation meter (−1 to +1, live + min-hold over 30s)
   - Overall M/S balance (% side energy)
   - Per-band M/S ratio compared to reference (sub/bass/low-mid/mid/high)
   - Mono-sum LUFS vs stereo LUFS delta (shows how much loudness you'd lose collapsed to mono — critical DJ check)
   - Mono-compatibility check below 120Hz (sub must be mono)
   - Diagnoses: "low-end too wide", "phase issue @ band X", "mono loudness penalty = NdB"
8. **Diagnosis section** — what's wrong (terse, numeric)
9. **Suggestions section** — what to do (prescriptive, plain English). Covers: EQ corrections, drum balance, dynamics/compression, M/S collapse advice, phase fixes.
10. **Toggle** between Terse / Coach style
11. **Section-aware analysis** — energy-gated auto-detection of drop (top 40% loudest windows on reference, live energy gate on user audio) with manual override dropdown (`Drop` / `Intro-Breakdown` / `Full track`)
12. **Perceived Loudness Translator (Fletcher-Munson / ISO 226:2023)** — critical for headphone-only mixers:
    - Equal-loudness weighted spectrum view (toggle) — shows spectrum as ears PERCEIVE it, not raw SPL
    - Multi-level preview panel — shows perceived tonal balance at Quiet (~60 dB SPL), Normal (~75 dB SPL), Loud (~95 dB SPL) listening levels simultaneously
    - Diagnoses: "bass disappears at low volumes" / "mix harsh at loud volumes" / translation warnings
    - Specifically addresses the root cause of bad translation — mixing at one volume, shipping to listeners at many different volumes

### Explicitly out of scope for v1 (deferred to v2+)
- Snapshots / session history (all live only)
- Averaged reference library / target curves
- Subgenre-specific presets
- Stem/element-level analysis → v2
- Manual section segmentation beyond drop-gating → v2
- Spectral masking detector → v2
- Saturation / harmonic density comparison → v2
- Sub-bass consistency tracker → v2
- Max for Live port → v2+
- Polished UI — functional first, just color-coded hot/cold viz required

## Loudness Targets (DnB 2026)
- **Target**: −4 LUFS-I
- **Acceptable range**: −6 to −4
- **Warn below**: −6 ("commercially quiet")
- True Peak ceiling: −0.3 to −1.0 dBTP
- Typical PLR: 7–10 dB

## Section Detection Strategy (v1)
- **Reference processing**: analyze reference track, isolate top 40% loudest windows as the "drop" profile. Compute target spectrum from those only.
- **Live**: energy-gated. When user's master is below RMS threshold → spectrum comparison freezes (no false comparisons). When above → active. Visible state indicator in UI ("🔇 waiting" / "🔊 analyzing").
- **Manual override**: `Drop` (default) / `Intro-Breakdown` (inverse gate) / `Full track` (no gate) dropdown.

## Technical Decisions

### Stack (locked post-research)
- **Language**: Python 3.13
- **DSP**: `librosa` (FFT, onset-strength, spectral features), `pyloudnorm` (LUFS), `numpy`, `scipy` (Hilbert, filtering), `soundfile` (reference loading)
- **Audio capture**: `sounddevice` → BlackHole 2ch
- **UI**: `PyQtGraph` + `PySide6` with OpenGL backend
- **Package management**: `uv` (fast, handles Apple Silicon wheels cleanly)
- **Onset detection**: `librosa.onset.onset_strength` on rolling 2–4s buffer + custom peak-pick (no extra `aubio` dep)
- **ISO 226:2023**: hand-rolled from published lookup table, ~60 lines, log-freq/log-phon interpolation
- **Pump detection**: custom — low-band (20–200Hz) envelope → Hilbert → autocorrelation at tempo-locked lag. Reported as advisory score, not pass/fail.

### Audio routing (locked post-research)
**Confirmed viable**: BlackHole is a libASPL-based multi-client HAL plugin. Concurrent reads by eqMac + Python app work natively.

Setup:
- Ableton output → BlackHole 2ch
- eqMac `Source` = BlackHole 2ch → outputs to real headphones (disable eqMac's "Auto-switch output" option)
- Spectrum Advisor reads BlackHole 2ch concurrently
- **Pin all sample rates to 48 kHz** (Audio MIDI Setup, Ableton prefs, eqMac output, `sounddevice.default.samplerate`). UI must display a hard-fail warning if mismatch detected.
- Total latency: ~8–15ms at 128-sample buffer (acceptable for mixing)

**Fallback plan**: if BlackHole proves flaky on user's specific macOS 15 build, spend max 1 hour debugging (sample rates, permissions, `sudo killall coreaudiod`). If still broken, buy Loopback ($99) — escape hatch, not preemptive purchase.

### Analysis specs (locked)
- FFT size: 4096 with 50% overlap (Welch's method)
- Sample rate: 48 kHz (hard-fail on mismatch, no automatic resampling in v1)
- Spectrum averaging window: 10s rolling
- LUFS-I reset: 30s rolling
- Energy gate threshold: −25 LUFS-S (below = gated)
- Update rate: 10 Hz (UI refresh)
- Audio buffer: 1024 samples (~21ms)
- sounddevice latency mode: `'low'`

### Threading architecture (locked)
Producer-consumer pattern to avoid xruns in Ableton:
1. **Audio callback thread** (sounddevice): push raw frames into lock-free ring buffer. NO DSP here.
2. **DSP worker thread** (10Hz): pull from ring buffer, run all analyses (FFT, LUFS, dynamics, transients), emit Qt signal with results.
3. **Qt UI thread**: receive signal via QTimer at 100ms, call `curve.setData()` on pre-allocated numpy views. No new allocations per frame.

### PyQtGraph performance (locked)
- `useOpenGL=True` per plot
- `setDownsampling(auto=True, mode='peak')`, `setClipToView(True)`
- All pens `width=1` (width>1 kills performance)
- Preallocated ring buffers, zero-copy numpy views to `setData()`
- Target: 12 plots × 10Hz — trivial headroom on M4

## Risk Register
| # | Risk | Likelihood | Mitigation |
|---|------|-----------|------------|
| 1 | Sample rate mismatch (Ableton 44.1 vs BlackHole 48) | Medium | Hard-fail UI warning; pin 48kHz everywhere |
| 2 | Pump detection false positives from legit kick patterns | High | Advisory metric only; validate against 10–15 known references during build |
| 3 | GIL contention between audio capture + DSP + UI threads | Medium | Strict producer-consumer; ring buffer; no DSP in audio callback |
| 4 | macOS audio permissions lost after OS updates | Low | Document re-granting steps in README |
| 5 | BlackHole concurrent-reader dropouts (rare 3+ client bug) | Low | Fallback to Loopback ($99) if encountered |
| 6 | ISO 226:2023 table errors during hand-roll | Low | Unit-test against published ISO phon values |

## Constraints
- macOS 15 / Apple Silicon M4, 16GB RAM
- Must not touch Ableton's audio export (monitoring-only analysis)
- Must coexist with eqMac running on the same audio chain
- User will NOT have studio monitors — tool is his substitute for a proper monitoring environment

## Testing Approach
Manual for v1 (personal production tool). Validation = "does it correctly identify known mix problems in Dorian's existing tracks?" and "does the live reading match what a paid reference analyzer shows?"

## Success Criteria (v1 done)
1. Dorian can open the app, pick a reference, play a loop in Ableton, and within 5 seconds see a meaningful spectrum comparison
2. The sub-bass meter reliably shows energy below 60Hz that he cannot hear on SRH440
3. The suggestions section produces at least one actionable tip per mix session that Dorian acts on
4. After 2 weeks of use, he has achieved a mix measuring −5 LUFS-I or better on at least one finished track (the real-world validation metric)
