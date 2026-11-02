# DUB-1 engine v2: the sound redesign

Same instrument, same pages, knobs, pieces and save format. Only the sound engine (`firmware/src/dsp/amb.c`)
changed, measured against Audioreakt's Dub Techno 4 and 5 Live sets (their device chains, MIDI and the DT4
mixdown), the Riemann Dub Techno templates 11-13, and B. Koçer's *Dub Techno: The Orphic Experience of Sound*
(the dub echo chapter).

## What was wrong, measured

Full mix against the Audioreakt DT4 mixdown (bands relative to the whole):

| | 110-300 Hz | 6-12 kHz | mid side/mid | kick at 2 ms |
|---|---|---|---|---|
| Audioreakt DT4 | -10 / -15 dB | -31 dB | -3 dB (wide) | 180-300 Hz |
| DUB-1 1.4 (Basic) | -16 / -23 dB | -43 dB | -9 dB | ~90 Hz |
| engine v2 (Basic) | -12 / -19 dB | -44 dB* | -9 dB | ~170 Hz |

(*Basic plays few hats; Steppers is at -31.) The 1.4 chord was voiced C3-B3 and high-passed at 160 Hz, so the low-mid
wash the dub sets live on was missing; every stab opened the filter the same amount (velocity was level only); the
kick had no knock.

## The changes

**Chord stab**
- Voiced F#2-F3 (was C3-B3): the dub sets voice F1-G2 roots.
- Filter 24 dB (a second pole pair at 1.3x the cutoff), base an octave lower (62 Hz x knob, was 110), envelope
  deeper (Saws 4 octaves, was 2.5; the other voices likewise), floor 55 Hz (was 90).
- Velocity drives the filter: envelope depth 0.3-1.0 and cutoff -1.1..+0.25 octave. The bar's first stab
  0.9-1.0, the others 0.5-0.88 (were 0.65-0.9). A soft stab under a ringing one adds to it instead of cutting it.
- Ghost stabs: soft (0.2-0.45), dark stabs on empty off-beat sixteenths, mostly heard in the echo (the "chord
  rain" of DT4's Chord 1 layer). None at Vary 0 / Hold, ~0.6 a bar at Vary 50, ~1.5 from 62.
- Overdrive after the filter (above ~250 Hz driven, tone ~5 kHz, mixed under): DT5's Drift > Overdrive chain.
- High-pass ~90 Hz (was ~160).

**Echo**
- Right channel reads its own time, 3:2 against the left (1/8. against 1/8, 1/4 against 1/4.): DT4's Echo,
  L 3/16 R 2/16.
- Loop low-pass two poles (Butterworth, no resonance so a near-100% loop can't ring), wandering +-0.4 octave at
  ~0.11 Hz with Drift (Koçer: the echo's LPF on a slow LFO).
- Input driven into a soft clip (the sets' Echo at +5..+13 dB input with its clipper): ghosts come back nearly as
  loud as accents.

**Bass (Sub voice)**: a saw through two poles at ~2.4x the note under the sine, opened to ~7x on each note,
closing over ~70 ms (Bite deepens it): DT4's driven saw bass at 160 Hz. Warmth at 80-200 Hz.

**Kick**: starts at ~4.5x its note (Tight; each kit its own), body held 20-70 ms before the decay. Measured
170 Hz at 2 ms, -1 dB at 60 ms (refs: 180-300 Hz, -0..-1 dB at 60 ms). Rumble -3 dB to keep its tail as before.

**Hats**: low-pass per kit (0.65-0.9, was 0.65 for all: ~6 kHz), +1 dB.

**Master**: glue compression, 2:1 above -15 dBFS, ~10 ms / 150 ms, linked, +0.5 dB.

## Costs and checks

- Level: ~+1.5 dB RMS over 1.4, peaks -6..-8 dBFS.
- CPU (host, per 256-sample block, mean): Basic 26.6 > 27.7 us, Steppers 29.1 > 32.3, Night Drive 29.2 > 31.6,
  Pressure 31.2 > 30.8; p99 up to +16% (Steppers). Check the Setup page's CPU on the FM-1.
- RAM: a few dozen floats; no new buffers.
- Tests: `amb_test` ok; all 34 scenarios ok. Two expectations changed: the Saws' Env depth (4 octaves) and
  `introoutro`'s Pressure intro (-5.5 LU under its main, was -4.5; the ghosts make the main denser).
