# PROJECT — Spectrum Advisor

## Overview
Real-time spectrum analysis and mix-coaching tool for Drum & Bass producers. Compares a live Ableton master bus against reference DnB tracks, visualizes frequency deviations, and gives actionable coaching — with a focus on surfacing sub-bass and low-mid issues that closed-back headphones cannot reveal.

## Owner
Dorian Maissin (dorian.maissin@gmail.com)

## Status
Initialized — pre-discovery.

## Versions
- **v1** (current, initialized): core real-time spectrum + reference comparison tool.

## Tech Stack
_To be decided during `/bolt:discover` and `/bolt:research`._

Leading candidates:
- Python (librosa, pyloudnorm, numpy, scipy)
- sounddevice + BlackHole (macOS virtual audio routing)
- PyQtGraph / PySide6 (real-time visualization)

## Constraints
- macOS (Apple Silicon M4)
- Works with Ableton Live 10 Suite
- Must not interfere with Ableton's exports (monitoring-only analysis)
- Target user mixes on Shure SRH440 headphones (sub-bass physically limited) — tool must compensate by visualizing what user cannot hear
