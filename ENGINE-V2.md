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

## Engine v3: the arrangement (the songs)

v2 fixed the sound; the songs were still static: Basic played 100 bars of one Full section, one shaker hit a bar, one
chord change in three minutes. The Audioreakt sets do the opposite: the chord bed never stops, the kick and the bass
leave for 16-bar breakdowns, layers come and go a block at a time, and one root holds for the whole track while its
voicing moves every two bars.

- **16-bar blocks.** Sections are picked only at a block's end. Motion's zones: under 25 Full only; from 25 a
  departure from Full now and then (at least 32 bars of Full between), 25% of the chances at 25 to 75% at 100; from 40
  more and more of them Breaks (none at 40, 75% from 80), each a block long, back to Full (now and then a Thin).
- **Layers turning in Full.** At each block the shaker and the open hat both, or one alone (DT4's Shakers and 909
  tracks), more often as Motion rises; never under 10, in a song's intro or outro, under Lock, or in the first 32 bars.
  Motion's own random draws, so the notes are the same at any Motion under 25.
- **The voicing breathes.** Every two bars the top voice goes a scale step up and back (m7 and m7sus4 in turn), the
  root kept: DT5's chord memory, its +5 switched every two bars. Off when Change is on Hold, at Vary 0, on a slot's
  written chord, in dub hold.
- **Recipes.** Motion: Basic 70, Chamber 55, Steppers 75, Fog 60, Concrete 80, Night Drive 68, Pressure 78, Haze 50
  (were 4-42). Hits: Basic 46 (was 8), Steppers 55, Night Drive 45, Pressure 50. `recipe_prints.h` regenerated, so a
  saved piece never changed takes the new recipe.

5 minutes of each: Basic Full 48, Break 16, Full 32, Thin 16, Full 32, Break; Pressure Full 48, Break 16, Full 32,
Break 16, Full 32, Break. RAM: a few bytes. Tests: the Motion-zone checks moved to the new zones; the sound checks
(rumble, strips, bass Low cut) set Motion 0 to measure the sound alone; the scenarios' Basic Hits starts and the host's
old-format stores (recipes_before_v3) follow the new recipes.

### v3b: tails and pulse (after the first listen: Basic downtempo, tails too short and too clean)

Measured on the side channel (the tails: the kick and the bass are mono), against the DT4 mixdown, 400 Hz-6 kHz:
Basic was 4-10 dB under the sets, its tails dark above 2.5 kHz; now within 1-3 dB.

- **Echo brighter and thinner each pass.** Its input low-pass ~4.2 kHz at Tone 50 (was ~2); the loop's low-pass 3-5
  kHz with a little peak (was 1.5-3.7 k); the loop's high-pass ~390-900 Hz (was ~220-780): the lows gone by the
  third repeat, the mids and the top going on.
- **The screech.** A band at ~1.8-2.6 kHz (Q ~1.7) added to each repeat, as much as keeps the loop at that band
  just under 1 (0.97): DT4's Filter Delay, its band-passed repeats. A held throw pushes it past 1; the soft clip
  bounds it.
- **Erosion on the stab.** The stab times a band of noise (~2-4.4 kHz), as DT4's Erosion at 4.8 kHz: hiss that
  follows the stab, the grain the echo's top repeats.
- **Echo louder.** Its return x1.45 (1.2), its driven input's ceiling 0.42 (0.3).
- **Never half-time.** A Thin block keeps four on the floor (2 and 4 ghosted at most); it was half-time, 16 bars of
  it now. Basic: Drums 86 (80), Duck 60 (50).

CPU (host, mean per block): Basic 27.1 us, Steppers 28.7, Pressure 34.5 (30.8 in v2: its Tonewheel rings long, the
erosion with it). Tests: `transition`'s hand-over strike reads 115 (the erosion's grain on the thrown stab, 17 ms
after the cue; was 94-97), its limit 125; the scenarios' Basic Drums follow the recipe.
