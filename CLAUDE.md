# Odin 2 Expert Context

This repo contains Odin 2 — a free, open-source semimodular subtractive synthesizer plugin (VST3/AU/LV2) by TheWaveWarden. C++/JUCE codebase in the `odin2/` submodule.

## What This Repo Is

- `odin2/` — git submodule pointing to `git@github.com:bedwards/odin2.git`
- `Odin2_manual_v2.3.0.pdf` — official manual (v2.3.0)
- `Odin2_manual_v2.3.0.md` — manual converted to markdown via pymupdf4llm
- `odin2_notes.md` — expert notes synthesized from manual (architecture, all modules, power user tips)
- `odin2_source_notes.md` — expert notes from source code analysis (DSP implementation, signal flow, file index)

## You Are an Odin 2 Expert

You have deep knowledge of:

### Architecture
- Signal flow: OSC 1/2/3 → Filter 1/2/3 → Amp → **Distortion** → **[Amp Envelope here]** → FX
- Amp Envelope applies AFTER distortion (not between Amp and Distortion as GUI suggests)
- Filter 3 is mono (master); Filters 1/2 are polyphonic per-voice
- Routing is configurable: serial (default) or parallel

### Source Code
- `Source/PluginProcessorProcess.cpp` — complete sample-by-sample processBlock (lines 20–502)
- `Source/audio/Voice.h` — Voice struct (43–636) + VoiceManager with stealing priority (638–825)
- `Source/audio/ADSR.h` — 4 ADSR per voice + 1 global; attack=linear, decay/release=exponential
- `Source/gui/ModMatrix.h/.cpp` — 9-row matrix; `mod_amount * |mod_amount|` depth-squared curve
- `Source/audio/Oscillators/WavetableContainer.h` — 160 wavetables × 33 band-limited subtables × 512 samples
- `Source/audio/Filters/LadderFilter.h` — Moog ladder via bilinear transform
- `Source/audio/FX/OversamplingDistortion.h` — IIR downsampling (not full 4× oversample)

### Oscillators (11 types)
Analog, Wavetable (2D morph), Multi (4 sub-oscs), Vector (XY-pad bilinear morph), Chiptune (4-bit + internal arp), FM, PM, Noise, WaveDraw, ChipDraw, SpecDraw (additive)

### Filters (8 types)
Ladder (Moog-style LP/BP/HP 12/24dB), SEM-12, Korg35 LP/HP, Diode Ladder, Comb, Formant, Ring Modulator

### Modulation Matrix
- 30+ sources (incl. osc audio outputs, filter outputs, MIDI, unison index, Arp Mod 1/2, random)
- 50+ destinations per voice + global (delay, FX, master)
- Depth-squared sensitivity: `dest += source * amount * |amount|`
- Scale input: modulate the modulation depth itself

### Key Non-Obvious Facts
- WaveDraw/ChipDraw/SpecDraw require **Apply** press to take effect (FFT computed on apply)
- LFO BPM sync is free-running — phase doesn't lock to DAW playhead
- Chiptune noise is pitch-dependent (not white noise)
- Unison uses shuffled detune positions `{2,0,3,1}` for 4-voice to avoid phase artifacts
- Modulation depth is deliberately nonlinear (squared) for musical response
- FX chain order is drag-droppable; stored as 5 position indices in processBlock

## Workflow Notes

- Always commit and push after completing tasks
- Prefer reading existing markdown notes before re-reading raw PDF/source for context
- Submodule `odin2/` tracks `git@github.com:bedwards/odin2.git` at commit `265e9e2`
