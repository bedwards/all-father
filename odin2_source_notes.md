# Odin 2 Source Code Notes

C++ / JUCE synthesizer plugin. 424 source files under `Source/`.

---

## Top-Level Structure

```
Source/
├── audio/
│   ├── Oscillators/     # 11 osc types + wavetable system + LFO
│   ├── Filters/         # 8 filter algorithms
│   └── FX/              # delay, chorus, phaser, flanger, reverb, distortion
├── gui/                 # JUCE UI components + ModMatrix
├── PluginProcessor.h    # OdinAudioProcessor (main class)
├── PluginProcessorProcess.cpp  # processBlock() — full signal flow
├── GlobalIncludes.h     # constants, enums, macros
├── ModMatrix.h/.cpp     # modulation routing engine
├── Voice.h              # Voice struct + VoiceManager
└── ADSR.h/.cpp          # envelope generators
libs/                    # JUCE, tuning-library, zita-rev1
assets/                  # graphics, fonts, wavetables
```

---

## Core Constants (`Source/GlobalIncludes.h`, `Source/audio/OdinConstants.h`)

```cpp
VOICES = 24                      // max polyphony
MODMATRIX_ROWS = 9               // matrix row count
SUBTABLES_PER_WAVETABLE = 33     // band-limited subtables per wavetable
WAVETABLE_LENGTH = 512           // samples per wavetable cycle
```

160 pre-designed wavetables × 33 subtables = 5,280 waveforms loaded at startup.

---

## Signal Flow (`Source/PluginProcessorProcess.cpp`)

Sample-by-sample loop (lines 20–502):

```
1. Parameter smoothing (exp smoothing, factors 0.995–0.998)
2. MIDI processing (note on/off, CC, pitch bend, arp triggers)
3. ModMatrix: zero all destinations → apply modulation
4. Per-voice (24 voices):
   a. Envelope + LFO outputs cached
   b. Oscillators (3 per voice) → osc outputs
   c. Filters (2 per voice, configurable routing) → filter outputs
   d. Amplifier + unison pan/gain → L/R per voice
   e. Amp Envelope applied HERE (after distortion)
5. Filter 3 (mono master filter, same 8 types)
6. FX chain (5 serial slots, order drag-droppable)
7. Master gain → output buffer
```

**Key gotcha**: Amp Envelope applied after distortion stage in step 4e, not between amp and distortion.

---

## Voice Management (`Source/audio/Voice.h`)

`Voice` struct (lines 43–636): all per-note state — 3 oscillators, 2 filters, 4 envelopes, 4 LFOs.

`VoiceManager` (lines 638–825): voice stealing priority:
1. Reclaim sustain-pedal-held notes first
2. Free voices (not busy)
3. Voices in release phase (oldest first, tracked in `m_voice_history`)
4. Steal oldest active voices

---

## Oscillators (`Source/audio/Oscillators/`)

Base class: `Oscillator` — handles pitch modulation (exp + linear), glide (exponential smoothing), unison detuning.

```cpp
static float pitchShiftMultiplier(float semitones) { return pow(2, semitones/12); }
```

Used everywhere for tuning, filter mod ranges, LFO shifts, pitch bend.

### Types

| Class | Notes |
|---|---|
| `AnalogOscillator` | Saw/Square/Triangle/Sine + PWM + `DriftGenerator` |
| `WavetableOsc2D` | 4-subtable morph, 35 selectable wavetables |
| `MultiOscillator` | 4 sub-oscs, detuned, separate wavetable positions + spread |
| `VectorOscillator` | XY-pad morphing between 4 user-assigned waveforms, bilinear interp |
| `ChiptuneOscillator` | 4-bit quantized, built-in 2/3-step arpeggiator, noise mode |
| `FMOscillator` | carrier + modulator, ratio control, exp and linear FM modes |
| `PMOscillator` | phase modulation variant of FM |
| `NoiseOscillator` | white noise + HP/LP filters |
| `WavedrawOsc1D` | user-drawn, 200 steps, FFT → wavetable on Apply |
| `ChipdrawOsc1D` | 32-step × 16-level (4-bit), FFT on Apply |
| `SpecdrawOsc1D` | draw harmonic amplitudes (additive), FFT on Apply |

### Wavetable Morphing (`WavetableOsc2D.h` lines 99–114)

```cpp
// Position 0–1 across 4 subtables
if (pos < 0.333)      { left=0, right=1, t=pos*3 }
else if (pos < 0.667) { left=1, right=2, t=(pos-0.333)*3 }
else                  { left=2, right=3, t=(pos-0.667)*3 }
// Linear interp between adjacent subtables
```

### Band-Limiting

33 subtables per wavetable, each pre-computed with harmonics removed above Nyquist for that frequency range. `WavetableContainer.h` selects correct subtable based on played pitch → no aliasing without runtime filtering.

---

## Filters (`Source/audio/Filters/`)

Base: `OdinFilterBase` — provides modulation framework via float pointers.

| Class | Algorithm | Slope | Notes |
|---|---|---|---|
| `LadderFilter` | Moog ladder, bilinear transform | 12/24dB | LP/BP/HP variants; resonance feedback; virtual analog |
| `SEMFilter12` | Steiner-Parker | 12dB | LP/notch/HP transition via SEM Transition param |
| `Korg35Filter` | Korg Monotron topology | 12dB | LP + HP variants |
| `DiodeFilter` | Diode bridge ladder | 24dB | Nonlinear; more aggressive than Moog under resonance |
| `FormantFilter` | Dual resonator | — | Vowel formants; Vel/Env apply to transition not freq |
| `CombFilter` | Tuned delay line | — | `f_delay = 1/f_cutoff`; feedback = resonance |
| `RingModulator` | AM via internal sine osc | — | Freq + dry/wet controls |
| `VAOnePoleFilter` | VA bilinear single-pole | 6dB | Building block used internally |

Filter modulation inputs (per filter): keyboard tracking, velocity, envelope amount, direct mod matrix signal.

Manual says no self-oscillation, but `LadderFilter` source has resonance feedback (`OdinFilterBase.h` lines 103–110 has Oberheim variation topology).

---

## Envelopes (`Source/audio/ADSR.h`)

4 ADSRs per voice + 1 global. State machine: ATTACK(0) → DECAY(1) → SUSTAIN(2) → RELEASE(3) → FINISHED(4).

- Attack: linear
- Decay + Release: exponential (`calcDecayFactor()`, `calcReleaseFactor()`)
- Loop mode: restarts attack after decay → LFO-like modulator
- Legato: attack starts from last envelope value, same slope
- Retrig: attack soft-resets from current value on overlapping note

Amp Envelope (ADSR 0) signals end-of-voice processing when FINISHED.

---

## LFOs (`Source/audio/Oscillators/LFO.h`)

Inherits `WavetableOsc1D` — uses low-res subtables for efficiency.

3 per-voice + 1 global. 13 waveform shapes + Sample & Hold mode.

BPM sync: `setFreqBPM(bpm, numerator, denominator)` → converts to Hz. Free-running (not DAW-position-locked).

Phase modulation range: ±48 semitones for extreme LFO effects.

---

## Modulation Matrix (`Source/gui/ModMatrix.h/.cpp`)

9 rows. Each row: 1 source → 2 independent destinations + optional scale source.

### Modulation Application (`ModMatrix.cpp` lines 29–42)

```cpp
// Depth-squared sensitivity curve:
dest += source * mod_amount * abs(mod_amount)

// With scaling:
if (scale_amount >= 0)
  dest += source * mod_amount * abs(mod_amount) * (1 + (scale_value - 1) * scale_amount)
else
  dest += source * mod_amount * abs(mod_amount) * (1 + abs(scale_value) * scale_amount)
```

`mod_amount * |mod_amount|` = nonlinear sensitivity (small amounts have less effect, large amounts more).

### Sources (30+)

Per-voice (poly): 3 osc outputs, 2 filter outputs, 3 ADSRs, 3 LFOs, MIDI note, velocity, random-per-voice, unison index  
Global (mono): Global LFO, Global ADSR, mod wheel, pitch bend, XY X/Y, channel pressure, breath, sustain pedal, soft pedal, constant 1.0, Arp Mod 1/2

### Destinations (50+)

Per-voice: osc pitch (exp/lin), volume, PW, FM/PM amount, carrier/modulator ratio, vector X/Y, wavetable position, multi detune/spread, chip arp speed; filter freq/res/gain/saturation/env/vel/kbd amount; ADSR A/D/S/R; LFO freq  
Global: delay (time/feedback/HP/dry/wet), chorus/phaser/flanger params, reverb, arp speed/gate, XY pad X/Y, master gain, glide

---

## Distortion (`Source/audio/FX/OversamplingDistortion.h`)

Not full 4× oversampling — uses **IIR downsampling filter** (York University coefficients, 10-tap buffer).

5 algorithms: hard clip (Clamp), wavefold (Fold), zero-crossing (Zero), sine lookup, cubic.

Applied per-voice on L/R channels. Amp Envelope multiplied after this stage.

---

## Unison (`Source/PluginProcessor.h` lines 312–331)

Predefined pan distributions: 1/2/3/4/6/12 voice counts.  
Detune positions shuffled to avoid phase artifacts (4-voice order: {2,0,3,1}).  
Gain reduction: `0.45×` for 12-voice, scaling up to `1.0×` for 1-voice.

```cpp
unison_detune_factor = 2^(position * detune_amount / 12)  // semitone-space detuning
```

---

## Parameter Management

`AudioProcessorValueTreeState` (m_value_tree) — all automatable params.  
Separate ValueTree for non-automatable state (play mode, LFO sync, routing).  
Patch save/load via ValueTree XML serialization.

All continuous controls smoothed sample-by-sample (exponential, factor ~0.997) to prevent clicks.

---

## Tuning System

`Tunings::Tuning` object from tuning-library (libs/). Per-voice pointer `MIDINoteToFreq()`.  
.kbm + .scl file support for arbitrary microtuning (added v2.3).

---

## Key File Index

| File | What's There |
|---|---|
| `PluginProcessorProcess.cpp` | Full processBlock(), lines 20–502 |
| `audio/Voice.h` | Voice struct (43–636), VoiceManager (638–825) |
| `audio/ADSR.h` | Envelope state machine (37–101) |
| `audio/Oscillators/Oscillator.h` | Base osc + pitch math (29–124) |
| `audio/Oscillators/WavetableOsc2D.h` | 2D morph logic (99–114) |
| `audio/Oscillators/WavetableContainer.h` | Wavetable registry (26–128) |
| `audio/Oscillators/FMOscillator.h` | FM/ratio implementation (23–128) |
| `audio/Oscillators/LFO.h` | LFO + S&H + BPM sync (20–90) |
| `audio/Filters/LadderFilter.h` | Moog ladder + resonance (25–218) |
| `audio/Filters/OdinFilterBase.h` | Filter base + Oberheim topology (103–110) |
| `audio/FX/OversamplingDistortion.h` | Distortion algorithms (72–75) |
| `audio/OdinConstants.h` | SUBTABLES=33, WT_LENGTH=512 (18–29) |
| `gui/ModMatrix.h` | Routing struct (17–240) |
| `gui/ModMatrix.cpp` | Depth-squared application (29–42) |
| `GlobalIncludes.h` | VOICES=24, all type enums (67, 90–115) |
| `PluginProcessor.h` | Unison pan tables, processor class (312–331) |

---

## Non-Obvious Implementation Details

**Modulation depth is squared**: `mod_amount * |mod_amount|` means the sensitivity curve is nonlinear. Small knob values have disproportionately less effect, large values more. Design choice for musicality.

**Wavetable position breaks at 0.333/0.667**: Not a bug — 4 subtables, 3 linear segments. Expect slight discontinuities in morph if driven fast.

**LFO sync is free-running**: BPM sets frequency but phase doesn't track DAW playhead. Two instances will drift after stop/start.

**FX order is drag-droppable at runtime**: internally stored as 5 position indices (`m_delay_position`, etc.) and the serial chain reorders accordingly in processBlock lines 446–472.

**Draw oscs require Apply**: `WavedrawTable`, `ChipdrawTable`, `SpecdrawTable` are recomputed via FFT only on Apply press, not on every draw stroke. Apply button turns red on unapplied changes.

**Noise in Chiptune**: pitch-dependent (random value output per cycle, so frequency of note = rate of random values). Very different from white noise — perceived pitch exists.

**Distortion uses IIR not FIR oversampling**: the downsampling filter is a 10-tap IIR with preset coefficients, not a proper 4× oversample chain. Adequate for the zero/fold/clamp algorithms but not artifact-free for all inputs.

**Filter 3 is always mono**: the master filter (slot 3) collapses polyphonic voices to mono before filtering. Poly filters (slots 1–2) process each voice independently.
