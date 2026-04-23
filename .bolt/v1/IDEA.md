# Spectrum Advisor

## The Idea
<!-- What is this project? Describe your vision in your own words. -->
A real-time spectrum analysis and mix-coaching tool for Drum & Bass production. Runs alongside Ableton Live 10, listens to the master bus via a virtual audio cable (BlackHole), and compares the live mix to reference DnB tracks. Visually shows where the mix has too much or too little energy per frequency band, with color-coded zones and actionable guidance.

## The Problem
<!-- What problem does this solve? Why does it need to exist? -->
I've been producing DnB for years in Ableton 10, and while I can write good tracks, I've never been able to get a clean, loud, commercial-quality mix. The frustration has killed my motivation repeatedly — sometimes I stop producing entirely.

The root cause (just identified): my Shure SRH440 headphones hide sub-bass by −6dB and recess the presence range. Without realizing it, for years I've been over-boosting lows and low-mids to "hear" bass, which created muddy mixes that can't reach modern DnB loudness targets (−3 to −4 LUFS) without pumping or clipping.

Even with AutoEQ correction applied (via eqMac), my closed-back drivers physically can't reproduce sub-bass accurately. I need a **visual tool** that shows me what my ears cannot reliably hear — especially below 80Hz.

## Key Features
<!-- What should it do? List the main features or capabilities. -->
- Load 1+ reference DnB tracks (I have many)
- LUFS-match reference to my live mix for fair comparison
- Real-time spectrum overlay: my live master vs reference, color-coded deviation per band (green / yellow / red)
- Named frequency zones (sub / bass / low-mid / mud / body / presence / air) to make the "where" obvious
- Loudness panel: LUFS-I (short + integrated), True Peak, PLR, Crest Factor — with DnB-specific targets
- Mono-compatibility / stereo-width check per band (critical below 120Hz)
- Actionable punch list: "You're +4dB at 280Hz — cut this range on bass/pads" (plain English, not just numbers)
- Live capture from Ableton master via BlackHole — no exports needed; tweaks show up instantly

## Stack / Tech Preferences
<!-- Any languages, frameworks, or tools you want to use? -->
- **Python** (I can run it, iterate fast)
- macOS / Apple Silicon M4, 16GB RAM
- **BlackHole** (free virtual audio cable) for live Ableton routing
- DSP libs: `librosa`, `pyloudnorm`, `numpy`, `scipy`, `sounddevice`
- UI: `PyQtGraph` + `PySide6` for real-time performance (alternative: Streamlit/Dash for simpler web UI — decide in discovery)

Does NOT need to be a Max for Live device in v1 (though keeping options open for v2).

## Notes
<!-- Anything else — constraints, inspiration, references, etc. -->
- Inspiration: a blend of Sonarworks (monitoring correction), iZotope Ozone's Match EQ (reference matching), and a mastering engineer's checklist.
- This is **Project 1** of a larger plan. **Project 2** (future): personalized perceptual headphone calibrator (going beyond AutoEQ with user-specific A/B test tones).
- Must never interfere with Ableton's audio export — tool only listens to monitoring output.
- Target workflow: I play a loop in Ableton → I see deviations from reference live → I tweak an EQ in Ableton → deviations update in real time → I stop when zones are green. Time per mix decision: seconds, not minutes.
- User context: I know basic mix techniques (sidechain, mono low-end, key matching) — the tool is for catching what my ears can't hear, not teaching me mixing 101.
