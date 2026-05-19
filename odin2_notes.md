# Odin 2 Expert Notes

Free, open-source semimodular subtractive synth by TheWaveWarden. VST3/AU/LV2. 64-bit only. No VST2.

---

## Architecture Overview

Signal flow: **OSC 1/2/3 → Filter 1/2/3 → Amp → Distortion → [Amp Envelope applied here] → FX chain**

Key gotcha: **Amp Envelope applies AFTER Distortion**, not between Amp and Distortion as GUI routing suggests.

Three independent slots each for oscillators and filters. Routing is configurable:
- Default: serial (all OSCs → F1 → F2 → Amp)
- Parallel: enable F1 Output + disable F2 Input F1 + route OSCs independently

Amplifier and Distortion are the only **polyphonic + stereo** sections in signal flow.

---

## Oscillators (11 types)

### Common params (all oscs)
- Octave / Semitones / Finetune (cents) — standard tuning
- Volume (dB) — mod with -100 = always silence; +100 = up to 0dB if below -12dB, else +12dB from current
- Reset (Rst) — restarts wave phase on note-on; useful for tight basses
- Sync (OSC 2/3 only) — hard-sync to OSC 1; uses 3x oversampling + DC-block filter internally

### Analog Osc
Waveforms: Sawtooth (fat-saw, curved not linear), Pulse (with PW control), Triangle, Sine  
- **Drift** — random pitch wander over time, emulates analog instability; subtle solo, obvious with two oscs
- PW: 0→1 shifts duty cycle; can't silence via knob but mod matrix can reach 0% or 100%

### Wavetable Osc
35 wavetables, each contains 4 waves. Position 0/0.333/0.666/1.0 = waves 1/2/3/4.  
Built-in modulation slot (Mod Env or LFO1) for fast wavetable position mod setup.

### Multi Osc
4 sub-oscillators in one slot. Detune calculated to avoid beating. Wavetable + Position + **Spread** (offsets sub-osc positions across the wavetable).

### Vector Osc
XY-pad with 4 corners (A/B/C/D), each assignable to any waveform in the synth including draw oscs. Bilinear interpolation between corners. Very powerful for morphing.

### Chiptune Osc
Emulates NES/GameBoy 4-bit sound chip. 4-bit Y-axis (16 steps). Built-in arpeggiator (2 or 3 steps, speed in Hz). Noise mode = random value each cycle (pitch-dependent, not white noise). 3x oversample on noise.

### FM Osc
Carrier + modulator setup. `f_mod = (ratio_mod / ratio_car) × f_carrier`.  
- Simple integer ratios → harmonic, bell-like
- Non-reducible fractions (e.g. 11/7) → inharmonic, wild
- Can also do FM via mod matrix for more flexibility
- Mod matrix lets ratios become non-rational (continuous sweep)

### PM Osc (Phase Modulation)
Same structure as FM but modulates phase of carrier. FM and PM produce identical frequency content when both use sine waves.

### Noise Osc
White noise source with built-in 6dB/oct HP and LP filters (virtual analog, 1st order).

### WaveDraw Osc
Draw arbitrary waveform (200 discrete steps). Must press **Apply** to take effect (button turns red when unapplied). Processed into spectral domain → wavetable.

### ChipDraw Osc
32 horizontal steps × 16 vertical values (4-bit). NES custom waveform style. Same Apply workflow. Usable in Chiptune Osc as waveform source.

### SpecDraw Osc
Additive synthesis: draw amplitude of harmonics (sine partials). Leftmost bar = fundamental. More bars = richer timbre. Can create sounds impossible in pure subtractive synthesis. Same Apply workflow.

---

## Filters (8 types)

### Common controls
- Frequency (cutoff, -3dB point)
- Resonance (peak at cutoff; **no self-oscillation** by design)
- Velocity (Vel) — MIDI velocity → filter freq; additive on top of current freq
- Envelope (Env) — Filter Envelope amount applied to freq; can be +/-
- Keyboard (Kbd) — MIDI note → filter freq; higher notes open filter more; additive
- Gain — dB control; same mod behavior as OSC volume
- Saturation — tanh waveshaping; position in signal loop varies per filter type

### LP/BP/HP (Ladder variants)
Emulates Moog ladder filter. 12dB/Oct and 24dB/Oct variants each.

### SEM-12
Oberheim SEM filter. LP → Notch → HP transition (SEM Transition knob). 12dB/Oct.

### Diode Ladder
Alternative ladder circuit (worked around original ladder patent). 24dB/Oct. More aggressive with resonance than standard ladder.

### KRG-35 LP/HP
Korg MS-20 style filter. Dirty/aggressive character with high resonance. Named KRG-35 but slope is **12dB/Oct**.

### Comb Filter
Tuned delay line. `f_delay = 1 / f_cutoff`. Resonance = feedback amount. Polarity (+/-) changes frequency behavior; negative inverts signal insertion. High resonance + frequency modulation = psychedelic smearing.

### Formant Filter
Dual resonator emulating vowel formants. Select two vowels (A/E/I/O/U/Ä/Ö/Ü), transition between them. Vel and Env parameters apply to formant transition, not raw frequency.

### Ring Modulator
Multiplies input by internal sine oscillator (amplitude modulation). Ring Mod Freq = internal osc frequency. Amount = dry/wet.

---

## Amplifier & Distortion

### Amplifier
- Gain, Pan, Velocity sensitivity
- Increasing Amp Velocity lowers base gain so max velocity (127) restores original level

### Distortion
3x oversampling internally. **Amp Envelope is applied AFTER this section.**  
Types:
- **Clamp** — hard clip at threshold
- **Fold** — wavefold at threshold (more harmonics, can fold multiple times)
- **Zero** — pulls to zero above threshold (most aggressive)

Boost = pre-gain to hit threshold easier. DryWet = processed/unprocessed mix.

---

## FX Chain

5 modules: Delay, Chorus, Phaser, Flanger, Reverb.  
**Order is drag-droppable** — processed left to right.  
All can run simultaneously.

### Delay
- Time: Hz or BPM-synced (arbitrary fractions, e.g. 5/16)
- Feedback: 0 = single echo, 1 = infinite
- PingPong: L/R crossfeed; mono input → L line → feeds R
- HP filter on wet signal (post-feedback loop, so echo chain doesn't keep getting filtered)
- Ducking: attenuates delay output when input signal is present (declutter)
- Separate Dry and Wet controls (unique among FX modules)

### Chorus
Dual delay-line read positions modulated by internal LFO. L/R LFOs 90° phase-offset for stereo spread. Rate synced or Hz.

### Phaser
Series of allpass filters modulated by internal LFO (L/R 90° offset). Phase-shifted signal added back to original → frequency-selective cancellation/boosting. Freq knob shifts base frequency of allpass filters.

### Flanger
Modulated comb filter. Short delay (<50ms) + LFO. Feedback can be positive or negative (positive/negative comb behavior). Metallic smearing at high feedback.

### Reverb
Based on Zita-rev1. Pre-delay, RT60 Decay, HF-Damp, built-in EQ (Gain + Freq, wet-only). DryWet control.

---

## Modulators

### ADSR Envelopes (4 total)
- **Amp Envelope** — hardwired to voice volume (post-distortion); also free mod source
- **Filter Envelope** — hardwired to filter freq when Filter Env enabled; +/- direction
- **Mod Envelope** — fully free; per-voice (poly)
- **Global Envelope** — exists once for all voices (mono); useful for unifying modulation across voices

Attack: linear curve. Decay + Release: exponential curves.  
Loop mode: restarts Attack after Decay → LFO-like behavior.  
Legato: Attack starts from last envelope value, same slope.

### LFOs (4 total)
- LFO 1/2/3: per-voice (poly)
- Global LFO: one for all voices (mono)
- Rate: Hz or BPM-synced (free-running but freq derived from BPM — not DAW position-locked)
- Reset: retrigger to wave start on note-on
- Sample & Hold mode: random bipolar value held per cycle
- Can use WaveDraw waveforms from osc slots 1/2/3

### XY-Pad
Two independent mono/unipolar mod sources (X and Y). Mouse-driven. Also automatable.

### Modwheel & Pitch Bend
Both auto-track MIDI CC. Pitchbend range configurable in semitones (default ±12). Pitchbend range can be set to 0 to use purely as mod source.

---

## Modulation Matrix

9 rows. Each row: Source → Destination 1 (with Amount) + Destination 2 (with Amount) + optional Scale source (with Scale Amount).

**Modulation Scaling**: Scale source multiplies the modulation depth. Scale Amount 100 = full multiply. Useful for Modwheel controlling vibrato depth.

Mono/Poly rules:
| | Mono Dest | Poly Dest |
|---|---|---|
| Mono Source | single mod | one source modulates all voices |
| Poly Source | only most recent voice | each voice independently |

### Key Modulation Sources
- OSC outputs (bipolar, poly) — use audio signal as mod source
- Filter outputs (bipolar, poly)
- Envelopes: Amp/Filter/Mod (unipolar, poly); Global (unipolar, mono)
- LFOs 1/2/3 (bipolar, poly); Global LFO (bipolar, mono)
- XY-Pad X/Y (unipolar, mono)
- MIDI Note, Velocity (unipolar, poly)
- Modwheel, PitchBend, Breath, Channel Pressure, Sustain/Soft Pedals (mono)
- Unison Index (bipolar, poly) — unique per-voice value in unison stack
- Arp Mod 1/2 (bipolar, poly) — per-step modulation from arpeggiator
- Random (bipolar, poly) — new value per voice trigger
- Constant (unipolar, mono) — always 1; useful for testing/offsets

### Notable Destination Ranges
- Osc Pitch Exp ±100 = ±2 octaves
- Osc Pitch Lin ±100 = ±2× current frequency (can reverse oscillator direction at -100)
- Filter Freq ±100 = ±64 semitones (factor ~40 or ~0.025)
- Env times ±100 = ×8 +0.3s or ÷8 -0.3s
- LFO Freq ±100 = ±4 octaves (factor 16 or 0.0625)

---

## Arpeggiator & Step Sequencer

Overrides incoming MIDI, generates sequences from held notes.  
Up to 32 steps. Step Active buttons toggle each step on/off for rhythmic patterns.

Directions: Up, Down, Up Down, Down Up, Crawl Up, Crawl Down.

Arp Mod 1 and Arp Mod 2: per-step knobs that send values to triggered note via mod matrix sources "Arp Mod 1/2". Use for per-step timbre variation.

Arp Transpose: per-step semitone offset. With octaves=1 and single key held → custom melodic sequence.

One-Shot: plays through steps once then stops.

---

## Global Settings

### Play Modes
- **Poly** — up to 24 voices; oldest held note stolen when limit hit
- **Legato** — 1 voice; overlapping notes don't retrigger envelopes; pitch glides
- **Retrig** — 1 voice; overlapping notes do soft-reset (Attack starts from current value, not zero)

### Unison
Stack multiple voices per keypress. Detune + Width controls. CPU scales with voice count — bounce to audio if needed.  
**Unison Index** mod source lets you differentiate behavior per voice in the stack.

### Microtuning (v2.3+)
Supports .kbm (keyboard mapping) and .scl (Scala) files. Export current tuning from GUI to edit manually. Scale archive: huygens-fokker.org/docs/scales.zip

### Glide
Exponential pitch slide from previous note freq to current. Modulatable.

### Master
Output gain. Same modulation behavior as Amp Gain.

---

## Power User Notes

**OSC as mod source**: Audio-rate OSC signals can be routed into the mod matrix as sources — enables audio-rate modulation (AM, ring-mod-style effects without the ring mod filter).

**ADSR Loop**: Loop mode on any envelope = per-voice LFO. Useful when you want envelope-shaped LFOs that stay poly.

**WaveDraw → LFO**: Draw a waveform in WaveDraw OSC slot, then set LFO waveform to that slot. Now LFO has custom shape.

**Vector Osc + Draw Oscs**: Use WaveDraw/ChipDraw/SpecDraw waves in Vector Osc corners for unique morphing.

**FM via mod matrix**: Route OSC 2 output → Osc Pitch Exp/Lin of OSC 1 for manual FM with continuous ratio control.

**Formant + Envelope**: Filter Env Amount on Formant Filter modulates the formant *transition* (not raw frequency) — creates vowel movement effects.

**Comb + high resonance + LFO on freq**: Psychedelic smearing effect.

**Arp Mod → timbre**: Per-step Arp Mod values routed to filter cutoff, distortion, or oscillator pitch → sequences that change sound character per step.

**Modulation Scaling with slow LFO**: LFO1 → filter cutoff, scaled by Global LFO → amplitude modulation of the LFO effect itself. Creates evolving, compound motion.

**Diode Ladder aggressive resonance**: At high resonance push into saturation stage for gritty, self-reinforcing character.

**Unison Index → detune/pan**: Besides built-in detune/width, use Unison Index in mod matrix to modulate any parameter (filter freq, wavetable pos) differently per voice — adds depth beyond simple pitch/pan spread.
