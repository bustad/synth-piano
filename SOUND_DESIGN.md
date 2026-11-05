# Sound design: how every sound is built

This document explains exactly how each of the 22 sounds in `piano.html` is generated. Everything is made in real time with the Web Audio API from oscillators, gains and one filter per note. There are no samples, no audio files and no libraries.

The numbers below were chosen from common synthesis practice, not measured from real instruments, so treat the sounds as approximations in the style of the instrument they are named after.

## 1. The signal path of one note

```
 partial oscillator 1 ─ gain ┐
 partial oscillator 2 ─ gain ┤
 partial oscillator 3 ─ gain ┼─► low-pass filter ─► envelope gain ─► release gain ─► bus
 ...                         ┘        ▲
 optional LFO (vibrato / wobble) ─────┘ (to pitch or to filter cutoff)

 bus ─► dry gain ──────────────────────────► master gain ─► compressor ─► speakers
 bus ─► convolver (reverb) ─► wet gain ────►
```

Every key press creates a fresh set of these nodes (a "voice"). On release the voice fades out and its oscillators are stopped, so nothing keeps running.

### 1.1 Pitch

The pitch of a note comes from its MIDI note number `n` after Transpose and Octave are applied (the key you press only decides the starting number):

```
n = key + transpose + 12 × octave
f = 440 × 2^((n − 69) / 12)        // frequency in Hz; A4 (n = 69) = 440 Hz
```

### 1.2 Partials (the oscillators)

Each sound is a list of **partials**: `[frequency multiple, waveform, level]`. A partial with multiple `2` runs at twice the note's frequency (an octave up); `0.5` runs an octave below; `1.003` runs very slightly sharp of the note.

- **Waveforms** have different harmonic content: a *sine* is a single pure frequency; a *triangle* has only odd harmonics that fall off quickly (soft and hollow); a *square* has only odd harmonics that fall off slowly (buzzy, hollow); a *sawtooth* has all harmonics (bright and rich).
- **Detuned copies.** Adding a second oscillator a few cents away from the first makes the two drift in and out of phase, which gives a slow shimmer similar to a chorus or to the several strings of one piano note. At 130 Hz, a `1.003` partner (about +5 cents) beats about 0.4 times per second. Multiples used in the presets, in cents: 1.002 ≈ +3, 1.003 ≈ +5, 1.006 ≈ +10, 1.012 ≈ +21, 1.025 ≈ +43 (and the matching 0.994, 0.988, 0.975 give the same amounts flat).
- **Loudness balancing.** Each partial's gain is `level × norm × gain`, where `norm = 0.84 ÷ (sum of all the sound's levels)` and `gain` is an optional per-sound trim (default 1). This keeps the 22 sounds at roughly similar loudness, so adding a partial to a preset doesn't make it louder overall.
- **Aliasing guard.** Any partial whose frequency would be 18 kHz or higher is not created (this mostly affects the bell-like partials in very high transposed notes).

### 1.3 The filter

Every voice passes through one low-pass filter (a Web Audio `BiquadFilterNode`, 12 dB per octave). Its cutoff **follows the pitch of the note**, so high and low notes keep a similar brightness:

```
start cutoff = clamp(f × f0,  fmin0, fmax0)
end cutoff   = clamp(f × f1,  fmin1, fmax1)
```

The cutoff glides from the start value to the end value along an exponential curve over `fRamp` seconds (1.2 s unless a sound says otherwise). `fmax` caps the cutoff for high notes; the optional `fmin` floor stops it closing too far for low notes (used only by *Grand piano (brighter)*).

- A cutoff that starts high and ends low gives the "bright attack, mellow sustain" of struck and plucked sounds.
- A cutoff that starts low and ends high (Synth brass, Moog-style lead) gives a "swell" or "wah" opening.
- **Resonance.** `q` sets the resonance peak at the cutoff. For a low-pass filter Web Audio measures Q in decibels, so `q: 6` is a 6 dB peak. The default is 0.5.

### 1.4 The envelope (how loudness changes over time)

The envelope gain starts at 0.0001 (silent; an exponential curve can't start at exactly 0), rises linearly to a peak of 0.55 in `att` seconds, and then does one of two things:

| Type | Sounds | What happens after the attack |
|---|---|---|
| **Decaying** | all pianos, Synth pluck | Exponential fall to silence at `dec` seconds after the note began, where `dec = max(decMin, decBase − n/18)`. Higher notes die away faster, like a real piano, and holding the key doesn't change this. |
| **Sustaining** | all other synths, retro synths | The level settles toward `0.55 × sus` with time constant `susT` (about 95% of the way after 3 × `susT`), and then holds for as long as the key is down. `decBase` and `decMin` are not used. |

### 1.5 The release

When you let go, a separate **release gain** node (kept apart from the envelope because Web Audio can't read an envelope's current value mid-curve) fades to zero with time constant `rel` (default 0.12 s). The oscillators are stopped `max(1 s, 8 × rel)` later, once the fade is inaudible.

### 1.6 LFOs (slow modulation)

Two sounds add a low-frequency oscillator:

- **Vibrato** (Vibrato lead): a 5.5 Hz sine LFO is routed to the `detune` input of every partial. Its depth starts at 0 and ramps linearly to ±28 cents over 0.65 s (`delay 0.25 + 0.4`), so the note starts steady and the vibrato blooms in.
- **Wobble** (Wobble bass): a 3.5 Hz sine LFO is added to the filter cutoff. Its depth is 85% of the cutoff, so the cutoff swings between about 15% and 185% of its base value. For a note at 65 Hz the base cutoff is 196 Hz, so it sweeps between roughly 29 and 363 Hz.

## 2. Reading the recipe tables

- **Partials** are written `waveform ×multiple (level)`.
- **Filter** is written `start → end`, as pitch multiples with the cap (and floor) in brackets. "open" means the cutoff is so far above the note that the filter has almost no effect.
- **Envelope**: `A` = attack in seconds; for decaying sounds `decay = max(min, base − n/18)`; for sustaining sounds `sus` is the held level and `τ` the settle time constant; `R` = release time constant.
- Example for the original Grand piano on C3 (`n = 48`, 130.8 Hz): the cutoff glides from 12 × 130.8 = 1570 Hz down to 3 × 130.8 = 392 Hz over 1.2 s, and the note decays over `6.5 − 48/18 = 3.83` s.

## 3. Pianos

All the piano-type sounds are decaying: a quick attack, a filter that closes as the note fades, and a slightly detuned partner to imitate the several strings of one key.

| Sound | Partials | Filter | Envelope |
|---|---|---|---|
| **Grand piano** | triangle ×1 (1), sine ×2 (.45), sine ×3 (.2), sine ×4 (.1), sine ×1.003 (.35) | 12 (≤ 10 kHz) → 3 (≤ 4 kHz), Q .5 | A 6 ms; decay max(1.6, 6.5 − n/18) |
| **Grand piano (brighter)** | triangle ×1 (1), sine ×2 (.55), ×3 (.3), ×4 (.16), ×5 (.08), ×6 (.04), ×1.003 (.35); gain .9 | 16 (3.5–12 kHz) → 5 (1.5–7 kHz) | A 5 ms; decay max(1.6, 6.5 − n/18) |
| **Bright piano** | sawtooth ×1 (.5), sine ×2 (.5), ×3 (.3), ×4 (.2), triangle ×1.002 (.4) | 16 (≤ 12 kHz) → 5 (≤ 6 kHz) | A 4 ms; decay max(1.4, 5.5 − n/18) |
| **Mellow piano** | sine ×1 (1), ×2 (.25), ×3 (.08), ×1.002 (.3) | 6 (≤ 5 kHz) → 2 (≤ 2 kHz) | A 10 ms; decay max(1.4, 5 − n/18) |
| **Electric piano** | sine ×1 (1), ×2 (.3), ×4 (.08), ×7 (.05), ×1.005 (.25) | 8 (≤ 9 kHz) → 4 (≤ 5 kHz) | A 4 ms; decay max(1.2, 4 − n/18) |
| **Honky-tonk** | triangle ×1 (1), ×1.012 (.8), ×0.988 (.6), sine ×2 (.3), ×3 (.15) | 10 (≤ 9 kHz) → 3 (≤ 4 kHz) | A 5 ms; decay max(1.2, 4 − n/18) |
| **Music box** | sine ×1 (1), sine ×4.01 (.25), sine ×6.3 (.1) | open (20, ≤ 14 kHz) | A 2 ms; decay max(0.8, 2.6 − n/18) |

How each one is made:

- **Grand piano.** A triangle fundamental gives a soft, round core. Sine partials at ×2, ×3 and ×4 with falling levels add the harmonic series of a struck string, and the ×1.003 sine imitates the second string of the same note. The filter closing from 12× to 3× the pitch gives a bright hammer attack that quickly softens.
- **Grand piano (brighter).** Added because the original sounds dark in the low register: its filter closed to only 3× the pitch, which for a low note is a few hundred hertz. This version (a) puts a **floor** under the cutoff, so it can never start below 3.5 kHz or end below 1.5 kHz, (b) strengthens the 2nd to 4th partials and adds a 5th and 6th, and (c) trims the gain slightly to compensate. For C3 (130.8 Hz) the cutoff goes 3500 → 1500 Hz instead of 1570 → 392 Hz. It uses only triangle and sine partials; a quiet sawtooth was tried first and removed because it sounded distorted.
- **Bright piano.** Starts from a sawtooth (all harmonics) instead of a triangle, with a higher filter, a faster attack and a shorter decay. It's a different timbre from the grand pianos, more like an upright or a pop piano.
- **Mellow piano.** Almost a pure sine with a hint of ×2 and ×3, a low filter and a slower attack, which gives a felt-piano or soft-pedal sound.
- **Electric piano.** Mostly sine with a few sparse partials (×2, ×4 and a faint ×7 for a bell-like "tine" ping), a detuned partner for warmth and a shorter decay. It is Rhodes-style in character, not an FM emulation.
- **Honky-tonk.** Three triangle oscillators, one in tune and two about 21 cents away on either side, which imitates a piano whose strings are badly out of tune with each other.
- **Music box.** A sine plus two **inharmonic** partials (×4.01 and ×6.3, which are not whole-number multiples of the fundamental), as found in tuned metal bars and bells. With a very fast attack and a short decay this reads as a music box.

## 4. Synths

All of these sustain while the key is held, except **Synth pluck**.

| Sound | Partials | Filter | Envelope |
|---|---|---|---|
| **Warm pad** | sawtooth ×1 (.5), ×1.006 (.5), ×0.994 (.5), sine ×0.5 (.4) | 4 (≤ 3.5 kHz) → 2.5 (≤ 2.5 kHz) | A .35; sus .8, τ .5; R .3 |
| **Glass pad** | sine ×1 (1), ×2 (.5), ×3 (.35), ×5 (.15), ×1.003 (.5) | 10 (≤ 9 kHz) → 8 (≤ 8 kHz) | A .25; sus .85, τ .4; R .4 |
| **Square lead** | square ×1 (.6), ×1.004 (.4) | 6 (≤ 6 kHz) → 4 (≤ 4.5 kHz) | A .01; sus .7, τ .1; R .08 |
| **Synth brass** | sawtooth ×1 (1), ×1.007 (.8) | 2 (≤ 2.5 kHz) → 8 (≤ 7 kHz) over .2 s | A .05; sus .8, τ .2; R .1 |
| **Organ** | sine ×1 (1), ×2 (.8), ×3 (.5), ×4 (.3), ×0.5 (.5) | open (12, ≤ 10 kHz) | A .012; sus 1, τ .02; R .05 |
| **Synth pluck** | sawtooth ×1 (1), ×1.005 (.7) | 14 (≤ 12 kHz) → 1.5 (≤ 3 kHz) over .35 s | A 3 ms; decay max(.6, 1.8 − n/18) |
| **Vibrato lead** | triangle ×1 (1), sine ×2 (.3), sawtooth ×1.003 (.35); gain .9 | 6 (≤ 6 kHz) → 4 (≤ 4.5 kHz) | A .04; sus .85, τ .1; R .15; vibrato 5.5 Hz, ±28 cents |

- **Warm pad.** Three sawtooths, one in tune and two about 10 cents either side, plus a sine an octave below for weight. A low filter keeps it dark and soft, and a slow 0.35 s attack and 0.3 s release give the swelling pad shape.
- **Glass pad.** Pure sine partials at ×1, ×2, ×3 and ×5 (skipping ×4) and a slow attack; the open-ish filter lets the upper partials ring, which gives a clear, glassy tone with no buzz.
- **Square lead.** Two slightly detuned square waves with a quick attack and a fairly high filter. The detune keeps the usually static square wave alive.
- **Synth brass.** Two detuned sawtooths with the filter **opening** from 2× to 8× the pitch in 0.2 s after a 50 ms attack. That opening is what makes a brass note "blare" into the pitch.
- **Organ.** Sine partials at ×1, ×2, ×3, ×4 plus a sub-octave at ×0.5, like pulling drawbars. The envelope is nearly an on/off gate (12 ms attack, 50 ms release time constant, full level while held), as on an electric organ.
- **Synth pluck.** Two detuned sawtooths with a filter that **falls** from 14× to 1.5× the pitch in 0.35 s and a short decay. This is the classic filter-envelope pluck.
- **Vibrato lead.** A smooth triangle core with a little ×2 sine and a detuned sawtooth for body, plus the pitch LFO described in section 1.6.

## 5. Retro synths

| Sound | Partials | Filter | Envelope |
|---|---|---|---|
| **Chiptune square** | square ×1 (1); gain .55 | open (20, ≤ 16 kHz) | A 3 ms; sus .85, τ .005; R .03 |
| **8-bit triangle** | triangle ×1 (1); gain .9 | open (20, ≤ 16 kHz) | A 3 ms; sus 1, τ .005; R .02 |
| **Moog-style lead** | sawtooth ×1 (1), ×1.003 (.6), square ×0.5 (.6); Q 6; gain .8 | 1.5 (≤ 1.5 kHz) → 5 (≤ 5 kHz) over .25 s | A .01; sus .85, τ .15; R .1 |
| **80s poly synth** | sawtooth ×1 (.6), ×1.008 (.6), ×0.992 (.6), square ×1 (.4), ×0.5 (.4); gain .8 | 5 (≤ 4.5 kHz) → 3 (≤ 3 kHz) | A .06; sus .75, τ .3; R .25 |
| **String machine** | sawtooth ×1 (.5), ×1.01 (.5), ×0.99 (.5), ×2 (.2), ×1.005 (.3); gain .8 | 3.5 (≤ 3.5 kHz) → 3 (≤ 3 kHz) | A .5; sus .9, τ .4; R .6 |
| **Supersaw** | sawtooth ×1 (.5), ×1.012 (.5), ×0.988 (.5), ×1.025 (.4), ×0.975 (.4); gain .7 | 8 (≤ 9 kHz) → 6 (≤ 7 kHz) | A .02; sus .85, τ .2; R .15 |
| **Sync-style lead** | sawtooth ×1 (1), ×1.004 (.5); Q 9; gain .7 | 14 (≤ 9 kHz) → 3 (≤ 2.5 kHz) over .5 s | A .01; sus .8, τ .1; R .1 |
| **Wobble bass** | sawtooth ×1 (1), square ×0.5 (.8), sawtooth ×1.004 (.5); Q 5; gain .7 | 3 (≤ 2.5 kHz), fixed, plus LFO | A .01; sus .9, τ .05; R .1; wobble 3.5 Hz |

- **Chiptune square** and **8-bit triangle.** The simplest sounds in the program: a single square or triangle oscillator with the filter wide open and an almost instant attack and release, like the pulse and triangle channels of an early game console. The square is trimmed (gain .55) because a square wave is much louder than the same amplitude of sine or triangle.
- **Moog-style lead.** The classic analog mono-synth recipe: two sawtooths a few cents apart plus a square an octave below (the "sub"), through a **resonant** low-pass filter (6 dB peak) that opens from 1.5× to 5× the pitch in 0.25 s. It is inspired by that sound, not an emulation of a particular instrument.
- **80s poly synth.** Three detuned sawtooths plus two square waves (one at the pitch, one an octave down) approximate the thick, chorused sound of a vintage polyphonic synthesiser. A moderate filter and a 60 ms attack soften it.
- **String machine.** Detuned sawtooths with one an octave up and a half-detuned extra; a low filter and a long 0.5 s attack and 0.6 s release give the slow, ensemble-like swell of 1970s string synths.
- **Supersaw.** Five sawtooths spread from −43 to +43 cents give the wide, shimmering trance lead. This is a simplified version of the technique: the real thing uses 7 voices and a different mix.
- **Sync-style lead.** True oscillator hard sync isn't available in Web Audio, so this **imitates** its sweeping, vowel-like tone with a resonant (9 dB) filter that sweeps from 14× down to 3× the pitch in 0.5 s over two slightly detuned sawtooths. It sounds related to a sync sweep but is not the same as one.
- **Wobble bass.** Two sawtooths plus a sub-octave square through a resonant (5 dB) filter whose cutoff is swung up and down by the 3.5 Hz LFO described in section 1.6. It works best in low octaves; with Octave set to −1 or −2 the LFO moves the cutoff across the range where the bass has most of its energy.

## 6. The drone

The drone reuses the same voice engine. It plays Do (C) in the chosen octave, shifted by Transpose but not by the Octave setting, using one of the sustaining presets above. It starts a voice with a different instrument, a fixed pitch and a different destination: a separate "drone level" gain that feeds the same bus (and so the same reverb) as the notes. Changing Transpose, octave or sound simply releases the old drone voice and starts a new one.

## 7. The reverb

The reverb is a convolution reverb, using an impulse response (IR) generated in code rather than loaded from a file:

1. A stereo buffer is created whose length is the room size in seconds.
2. Each sample is random noise, passed through a one-pole low-pass whose coefficient falls from 1.0 to 0.15 over the length of the buffer, so the **tail gets progressively darker**, as it does in a real room.
3. The result is multiplied by `(1 − t)^2.6`, a smooth decay curve (`t` runs from 0 to 1 over the buffer), and the first 400 samples fade in to avoid a click.
4. The left and right channels use different random noise, which gives a wide stereo spread.

The Reverb mix slider sets `dry = 1 − 0.5 × mix` and `wet = 1.4 × mix`, and the master gain feeds a dynamics compressor so that loud chords don't clip.

## 8. Making your own sound

Add an entry to the `INSTRUMENTS` object near the top of the script in `piano.html`, then add an `<option>` for it in the **Sound** menu (the drone menu picks up any preset with a `sus` value automatically):

```js
mysound: {
  parts: [[1, "sawtooth", 1], [2, "sine", 0.4]],   // [multiple, waveform, level]
  f0: 10, f1: 4,            // filter start and end, as multiples of the pitch
  fmax0: 8000, fmax1: 3000, // caps on those cutoffs (Hz)
  q: 2,                     // optional resonance in dB
  gain: 0.8,                // optional level trim
  att: 0.01,                // attack (s)
  sus: 0.8, susT: 0.15,     // sustaining: held level and settle time (omit both for a decaying sound)
  rel: 0.2                  // release time constant (s)
}
```

Quick guide to what each setting does to the tone:

| To make it… | Try |
|---|---|
| Brighter | raise `f0`, `f1`, `fmax0`, `fmax1`, or add `fmin0`/`fmin1`; add upper partials |
| Darker or rounder | lower `f1` and `fmax1`; use triangle or sine instead of sawtooth |
| Thicker or warmer | add detuned copies (×1.003 to ×1.012) or a ×0.5 sub-octave |
| Hollower | use square or triangle instead of sawtooth |
| More percussive | short `att`, no `sus`, small `decBase`, a filter that falls quickly (`f0` ≫ `f1`, short `fRamp`) |
| Softer or slower | longer `att` and `rel` |
| More vocal or "wah" | raise `q`, and sweep the filter with `f0` ≠ `f1` and a short `fRamp` |
