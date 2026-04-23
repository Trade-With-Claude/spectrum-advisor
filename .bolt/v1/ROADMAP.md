# ROADMAP — Spectrum Advisor v1

## Overview
6 phases, each independently plannable with `/bolt:plan`. Phases build on each other — do in order.

Target delivery: a working real-time DnB mix analyzer with spectrum/loudness/dynamics/stereo/transient/Fletcher-Munson analysis + diagnosis/suggestions engine, integrated with Ableton via BlackHole.

## Success Criteria (v1 done)
1. Dorian opens app, picks a reference, plays a loop in Ableton → meaningful analysis within 5s
2. Sub-bass meter shows energy <60Hz reliably (he can't hear it on SRH440)
3. Suggestions engine produces ≥1 actionable tip per session that Dorian acts on
4. Real-world validation: after 2 weeks of use, Dorian achieves a finished mix ≥ −5 LUFS-I

---

## Phase 1 — Foundation
**Goal**: Scaffolded Python project with a PyQtGraph window that opens, closes cleanly, and can import all stack deps.

**Deliverables**:
- `uv` project with Python 3.13 venv
- `pyproject.toml` with all locked deps: `librosa`, `pyloudnorm`, `sounddevice`, `soundfile`, `PySide6`, `pyqtgraph`, `numpy`, `scipy`
- Directory structure:
  ```
  spectrum-advisor/
  ├── src/spectrum_advisor/
  │   ├── __init__.py
  │   ├── main.py          # entry point
  │   ├── audio/           # capture, buffer, reference loading
  │   ├── analysis/        # DSP modules
  │   ├── ui/              # PyQtGraph window
  │   └── diagnosis/       # rules engine
  ├── tests/
  ├── references/          # user's reference tracks (gitignored)
  └── README.md            # BlackHole + eqMac setup guide
  ```
- Entry point script: `uv run spectrum-advisor` opens an empty PyQtGraph window with title bar
- README covers: BlackHole install, sample rate pinning, eqMac routing, how to launch

**Success check**: `uv run spectrum-advisor` opens a window on M4. All imports succeed. `pytest` runs (empty test suite OK).

**Est**: ~0.5 day

---

## Phase 2 — Audio I/O Pipeline
**Goal**: Live audio flows from Ableton → BlackHole → Python ring buffer; reference tracks load from disk.

**Deliverables**:
- `audio/capture.py`: `sounddevice.InputStream` reading BlackHole 2ch at 48kHz, blocksize 1024, `latency='low'`
- Lock-free ring buffer (30s rolling) via `collections.deque` with lock, or numpy-based circular buffer
- Producer-consumer threading:
  - Audio callback thread: pushes frames to ring buffer (NO DSP here)
  - DSP worker thread: pulls frames at 10Hz for analysis
- Sample rate detection + hard-fail warning if Ableton ≠ BlackHole ≠ 48kHz
- `audio/reference.py`: load WAV/FLAC/MP3/AIFF via `soundfile` + `librosa` resample-if-needed
- Folder scanner: recursively find audio files in user's reference folder

**Success check**:
- Play audio in any macOS app → Python prints live RMS values at 10Hz
- Load a reference track from disk, resample to 48kHz, verify waveform length matches expected
- Sample rate mismatch produces a clear warning (not silent corruption)

**Est**: ~1 day

---

## Phase 3 — Core Analysis (Spectrum + Loudness)
**Goal**: All foundational metrics compute correctly on both live buffer and loaded reference.

**Deliverables**:
- `analysis/spectrum.py`: Welch's method long-term average spectrum, FFT 4096, 50% overlap
- `analysis/loudness.py`: `pyloudnorm` integration — LUFS-I (30s rolling), LUFS-S (3s), True Peak, LRA (EBU R128), Crest factor, PLR
- `analysis/energy_gate.py`: −25 LUFS-S threshold, gates spectrum updates when quiet
- `analysis/reference_profile.py`:
  - Pre-compute reference spectrum using top 40% loudest windows only
  - Compute reference's own LUFS-I, PLR, crest (for comparison display)
  - Cache per-reference to `~/.cache/spectrum-advisor/<hash>.npz`
- LUFS-matching: align reference gain to user's current LUFS-I for fair spectrum comparison
- Deviation computation: 1/3 octave bands (31 bands from 20Hz to 20kHz), colored green/yellow/red zones

**Success check**:
- Analyzing a known reference (e.g., an Alix Perez track) produces LUFS/PLR/spectrum values within ±0.3dB of Youlean Loudness Meter 2 or Span
- Energy gate correctly freezes analysis during intro/quiet sections
- Deviation correctly flags known problem areas (compare user's bad old mix to a reference — should light up red at 250-500Hz)

**Est**: ~2 days

---

## Phase 4 — Advanced Analysis (Stereo + Transients + Pump + Fletcher-Munson)
**Goal**: All remaining DSP metrics implemented.

**Deliverables**:
- `analysis/stereo.py`:
  - Mid/Side separation (M = (L+R)/2, S = (L-R)/2)
  - Phase correlation meter (Pearson correlation of L vs R, live + 30s min-hold)
  - M/S per band ratios vs reference
  - Mono-sum LUFS vs stereo LUFS (delta = loudness penalty when summed)
  - Mono-compat check below 120Hz
- `analysis/transients.py`:
  - `librosa.onset.onset_strength` on rolling 2-4s buffer + custom peak-pick
  - Per-element windowing: kick (40-120Hz), snare body (150-300Hz), snare crack (3-7kHz), hi-hat (8-14kHz)
  - Transient punch per element vs reference
  - Drum-band total energy vs reference
- `analysis/pump.py`:
  - Low-band envelope (20-200Hz Butterworth BP + Hilbert for magnitude)
  - Autocorrelation over 4-8s window
  - Peak detection at tempo-locked lags (assume 172-176 BPM common for DnB, or tempo-detect)
  - Report as advisory score 0-10, NOT pass/fail
- `analysis/iso226.py`:
  - Hand-rolled ISO 226:2023 equal-loudness contour table (from published standard)
  - Log-freq / log-phon interpolation
  - Weight a raw spectrum to perceived-loudness spectrum at a given phon level
- `analysis/translation.py`:
  - Generate 3 preview spectrums at 60 dB, 75 dB, 95 dB SPL using ISO 226 weighting
  - Detect translation issues (bass loss at quiet, harshness at loud)

**Success check**:
- Phase correlation of a known mono file = +1.0, a known anti-phase file = −1.0
- Transient detection correctly finds kicks in an isolated DnB drum loop (onset count within ±2 of truth)
- Pump detector: clean master scores <3, intentionally-pumpy mix scores >6 (validate against 10 reference tracks)
- ISO 226 implementation matches published phon values within ±0.5 dB

**Est**: ~2-3 days

---

## Phase 5 — UI + Diagnosis/Suggestions Engine
**Goal**: Full dashboard working, rules engine translating metrics into plain-English advice.

**Deliverables**:
- `ui/dashboard.py` — PyQtGraph window matching the approved mockup:
  - Reference sidebar (folder picker, clickable list)
  - Spectrum heatmap (colored hot/cold deviation)
  - Loudness panel (LUFS-I/S, TP, target −4 with range indicator)
  - Dynamics panel (Crest, PLR, LRA, pump score)
  - Stereo panel (correlation, M/S %, mono penalty)
  - Transients panel (per-element punch)
  - Multi-level translation preview (3 curves at quiet/normal/loud)
  - Diagnosis section
  - Suggestions section (with Terse/Coach toggle)
  - Section mode dropdown (Drop / Intro-Breakdown / Full)
  - Status bar (BlackHole device, buffer fill %, LIVE / WAITING indicator)
- `diagnosis/rules.py` — rules engine:
  - Spectrum deviations → "+XdB @ YHz" diagnoses
  - Drum balance logic (punch + band-energy combined)
  - Stereo/phase warnings
  - Translation warnings (Fletcher-Munson)
  - Pump score interpretation
- `diagnosis/suggestions.py` — prescriptive text:
  - Terse mode: short actionable lines
  - Coach mode: longer explanation with causes + fixes
- Session persistence: save last reference, last section mode, last Terse/Coach preference to `~/.config/spectrum-advisor/session.json`
- Hard-fail UI warning banner for sample rate mismatch

**Success check**:
- Full dashboard renders at 10Hz without frame drops on M4
- Loading an Alix Perez reference, playing a deliberately bad mix in Ableton → top 3 diagnoses are correct (e.g., "too much 280Hz" if the mix has excess low-mids)
- Clicking between references updates target spectrum within 1s
- Session persists across app restarts

**Est**: ~2-3 days

---

## Phase 6 — Integration & Validation
**Goal**: End-to-end validation against reality. Tune defaults. Ship.

**Deliverables**:
- End-to-end test checklist:
  - [ ] Play 3 known pro DnB tracks through Ableton → BlackHole → analyzer reports them as "in range" across all metrics
  - [ ] Play 2 of Dorian's existing mixes → analyzer flags known issues (muddy low-mids, weak sub)
  - [ ] Compare loudness readings against Youlean Loudness Meter 2 (reference tool)
  - [ ] Compare spectrum readings against Voxengo Span or SPAN
  - [ ] Pump detection: 5 clean masters scored low + 5 intentionally-pumpy = scored high
- Tune energy gate threshold if 40% default doesn't match Dorian's tracks
- Tune suggestion templates based on real-world output quality
- README finalized with screenshots and troubleshooting guide
- Git tag `v1.0` and final commit
- Manual user-acceptance test with Dorian on 1 real mix session

**Success check**:
- All 4 v1 success criteria pass
- Dorian has used it on at least 3 mix sessions
- Dorian reports it helped him make at least one mix decision he wouldn't have made otherwise

**Est**: ~1-2 days

---

## Timeline
Roughly 9-12 days of focused work total. Expect 1.5-2x for real calendar time accounting for research spikes (pump detection validation) and UI polish iterations.

## Phase dependencies
```
Phase 1 (Foundation)
   ↓
Phase 2 (Audio I/O) ←——————————————┐
   ↓                                │
Phase 3 (Core Analysis)             │
   ↓                                │
Phase 4 (Advanced Analysis)         │
   ↓                                │
Phase 5 (UI + Diagnosis)  ← — — — — ┘
   ↓
Phase 6 (Integration)
```

No phase can be parallelized with an earlier one — each builds on the last.
