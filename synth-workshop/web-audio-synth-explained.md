# The Web Audio behind Synth Workshop

This document explains how `synth-workshop` makes sound. It follows the code in the file, so you can read the two side by side. Everything uses built-in Web Audio nodes. There are no libraries and no AudioWorklets.

## 1. The big picture

Web Audio works by connecting **nodes** into a graph. Sources (oscillators, noise) produce signal, processors (filters, gains, delays) change it, and everything ends at `ctx.destination`, which is your speakers.

This synth has two layers:

1. **One voice per held note.** A voice is a small graph built when you press a key and thrown away after the note ends.
2. **One shared effects chain.** Every voice feeds the same bus, so all notes share drive, chorus, delay and reverb.

```
 VOICE (one per note)                              SHARED CHAIN (built once)

 osc 1..3 (x unison) ──┐
                       ├─> filter ─> amp ─> trem ─> out ─> bus ─> drive ─┬─> dry ────────────┐
 noise ─> noise gain ──┘     (ADSR)  (ADSR)  (LFO)  (release)            ├─> chorus delay ───┤
          └─ or straight to amp (bypass)                                 ├─> echo delay ─────┼─> master ─> analyser ─> compressor ─> speakers
                                                                         └─> reverb ─────────┘
```

The graph shown on the page (the SVG) is a drawing of this structure. It is plain SVG and is not part of the audio code.

## 2. Starting the audio engine

Browsers will not play audio until the user interacts with the page. The synth creates its `AudioContext` lazily, inside the first key press or click:

```js
function init(){
  if(ctx){ctx.resume();return}
  ctx=new (window.AudioContext||window.webkitAudioContext)();
  // ...build the shared chain...
}
```

Calling `ctx.resume()` on later interactions handles the case where the browser suspended the context. The `webkitAudioContext` fallback covers older Safari.

## 3. The shared effects chain

Built once in `init()`:

| Stage | Nodes | Notes |
|---|---|---|
| Bus | `GainNode` | Every voice connects here |
| Drive | `WaveShaperNode` | Soft clipping with a `tanh` curve |
| Chorus | `DelayNode` + LFO | 22 ms delay, wobbled by a 0.7 Hz oscillator of ±4 ms |
| Echo | `DelayNode` + feedback `GainNode` | Output fed back into its own input |
| Reverb | `ConvolverNode` | Impulse response generated in code |
| Master | `GainNode` | Sums dry and effect paths |
| Meter | `AnalyserNode` | Feeds the oscilloscope, passes audio through unchanged |
| Safety | `DynamicsCompressorNode` | Stops many voices from clipping |

### Drive

A `WaveShaperNode` maps each input sample through a lookup table. The table is regenerated only when the Drive slider changes:

```js
const k=1+S.dr*25, n=1024, c=new Float32Array(n);
for(let i=0;i<n;i++){const x=i/(n-1)*2-1; c[i]=Math.tanh(k*x)/Math.tanh(k)}
sh.curve=c;
```

Dividing by `tanh(k)` keeps the loudest peaks at ±1 however hard you drive, so Drive changes the tone more than the volume.

### Chorus

A chorus is a short delay whose time is modulated, which makes the pitch of the delayed copy drift slightly against the dry sound. The LFO is a sine `OscillatorNode` running into a gain of 0.004, connected straight to the delay's `delayTime` **AudioParam**:

```js
cl.connect(clg); clg.connect(cd.delayTime);
```

Connecting a node to an AudioParam adds its output to the parameter's value. This is how all modulation works in Web Audio.

### Echo

A `DelayNode` with its output looped back through a gain node below 1 gives repeats that fade away. The Feedback slider is that gain, limited to 0.9 so it can never run away.

### Reverb

A `ConvolverNode` multiplies the input by an **impulse response**, the recording of a room clapping. This synth makes its own: 2.2 seconds of stereo white noise, faded out with a cubic curve:

```js
d[i]=(Math.random()*2-1)*Math.pow(1-i/n,3)
```

It sounds like a smooth, bright room rather than a real space, and it costs no download.

## 4. Anatomy of a voice

`start(m, vel)` builds a voice when a key goes down. The key steps:

1. Work out the frequency from the MIDI note: `f = 440 * 2^((m-69)/12)`.
2. Create the filter stage (one or two biquads).
3. Create oscillators and noise and connect them to the filter.
4. Create the LFO and connect it to its target.
5. Schedule the amplitude envelope.
6. Connect `amp -> trem -> out -> bus`.
7. Store a `release` function in the `V` table so the note can be stopped later.

All the scheduling uses the audio clock, `ctx.currentTime`, which is far more accurate than JavaScript timers.

### Oscillators and unison

Each of the three oscillators is an `OscillatorNode` with a type (sine, triangle, sawtooth, square), a frequency multiplier for the octave setting, and a fine tune in cents:

```js
o.type = s['o'+n+'w'];
o.frequency.value = f * Math.pow(2, octave);
o.detune.value = fineCents + spreadCents;
```

`detune` is measured in cents (100 cents is one semitone) and is applied on top of `frequency`, so it stays in tune across the keyboard.

**Unison** creates several copies of each oscillator and spreads their detune evenly from `-spread` to `+spread`. Slightly mistuned copies beat against each other, which is the thick "supersaw" sound. Each copy's gain is divided by the square root of the voice count, which keeps loudness roughly constant as you add voices. (Uncorrelated signals add in power, not amplitude, hence the square root.)

Web Audio oscillators always start at phase zero, so unison copies begin perfectly aligned. The detune separates them within milliseconds, but the first instant of a note can sound slightly "punchy".

### Pitch drop

For kicks and zaps, the frequency starts high and falls:

```js
o.frequency.setValueAtTime(f*mult*Math.pow(2, pe/12), t);
o.frequency.exponentialRampToValueAtTime(f*mult, t+pt);
```

An exponential ramp sounds like a smooth glide because pitch perception is logarithmic. Exponential ramps cannot start or end at zero, which is fine here because frequencies are always positive.

## 5. The noise source

Noise is a looping `AudioBufferSourceNode` reading from a buffer filled with random numbers. Three two-second buffers are made at startup:

- **White:** each sample is `Math.random()*2-1`. Equal energy at all frequencies, a bright hiss.
- **Pink:** white noise passed through a set of cascaded one-pole filters (Paul Kellet's economy method) so energy falls about 3 dB per octave. It sounds more natural, like rain or breath.
- **Brown:** white noise accumulated by a leaky integrator, so energy falls about 6 dB per octave. A deep rumble.

```js
z = (z + .02*w) / 1.02;   // brown noise: leaky integration
d[i] = z * 6;
```

Each note creates a fresh `AudioBufferSourceNode` pointing at the shared buffer. Source nodes are one-shot, but the underlying buffer is reused, so this is cheap.

Noise has its own gain node with an attack, decay and sustain envelope (section 7). A routing switch connects that gain either to the filter input or directly to `amp`, so noise can skip the filter. The release is shared with the rest of the voice.

## 6. The filter

Filtering uses `BiquadFilterNode`, a two-pole (12 dB per octave) filter. Choosing "24 dB/oct" simply puts a second one in series:

```js
for(let i=0;i<(s.f4?2:1);i++){
  const L = ctx.createBiquadFilter();
  L.type = s.ft;                      // lowpass, highpass or bandpass
  L.Q.value = s.ft==='bandpass' ? 1+s.fq : (i ? .7 : s.fq);
  ch.push(L); if(i) ch[i-1].connect(L);
}
```

Two points worth knowing:

- **Q means different things.** For low-pass and high-pass filters, `Q` is in decibels and acts as a resonance peak at the cutoff. For band-pass, it is a plain ratio controlling narrowness. The code converts for the band-pass case.
- **Cascading** puts the resonance only in the first stage (the second uses a flat 0.7) so the peak is not doubled.

### Key tracking and velocity

The base cutoff depends on the note and how hard you played it:

```js
base = fc * Math.pow(f/261.6, fk) * Math.pow(2, (vel-.7)*2);
```

- `(f/261.6)^fk` scales the cutoff with pitch. At 100% tracking, a note an octave higher gets double the cutoff, so the timbre stays the same across the keyboard.
- `2^((vel-.7)*2)` moves the cutoff up to about an octave either way depending on velocity. Harder hits are brighter, as on an acoustic instrument.

### Filter envelope

The envelope sweeps the cutoff from `base` up to `peak`, measured in **octaves** (`peak = base * 2^fe`), which sounds even across the range. It then settles to a sustain position:

```js
p.setValueAtTime(base, t);
p.linearRampToValueAtTime(pk, t + fa);
p.setTargetAtTime(sl, t + fa, fd/3);
```

The sustain slider sets how far down the sweep the cutoff stays, as a fraction of the octave amount: `sl = base * 2^(fe * fs)`.

## 7. Envelopes with AudioParam automation

Every envelope uses three scheduling methods on an `AudioParam`:

| Method | Behaviour |
|---|---|
| `setValueAtTime(v, t)` | Jump to `v` at time `t` |
| `linearRampToValueAtTime(v, t)` | Straight-line ramp, arriving at `v` at time `t` |
| `setTargetAtTime(v, t, tc)` | Exponential approach to `v`, starting at `t`, with time constant `tc` |

The amplitude envelope uses all three:

```js
amp.gain.setValueAtTime(0, t);
amp.gain.linearRampToValueAtTime(pa, t + a);              // attack
amp.gain.setTargetAtTime(pa * s.as, t + a, s.ad/3);       // decay to sustain
```

`setTargetAtTime` never quite arrives. After three time constants it has covered about 95% of the distance. Passing `decay/3` as the time constant means "roughly 95% of the way there after the decay time", which matches what the slider label suggests. A bonus is that the curve is natural-sounding, since most real decays are exponential.

The noise envelope is built the same way on its own gain node.

### Release and why there is an extra gain node

Releasing a note while the envelope is mid-flight is awkward. The natural approach, `cancelScheduledValues()`, wipes future events, but the parameter then snaps back to an earlier value. `cancelAndHoldAtTime()` solves this but is missing in some browsers.

The synth avoids the problem with an extra gain node (`out`) after the envelope:

```js
out.gain.setValueAtTime(1, w);
out.gain.setTargetAtTime(0, w, r/3);
src.forEach(o => o.stop(w + r*1.6 + .1));
```

The envelope on `amp` carries on undisturbed while `out` fades the whole voice to silence. Sources are stopped shortly after, so they free their resources. The filter's release uses `cancelScheduledValues` and a new `setTargetAtTime` back to the base cutoff, where a tiny discontinuity does not matter.

## 8. The LFO

The LFO is another `OscillatorNode` running into a gain node that sets the modulation depth. Its output goes to an AudioParam, and the target selects which one:

| Target | AudioParam | Depth |
|---|---|---|
| Filter | `detune` on each filter | depth × 3600 cents (up to ±3 octaves) |
| Pitch | `detune` on each oscillator | depth × 300 cents |
| Volume | `gain` on the `trem` node | swings between `1 - depth` and 1 |

Biquad filters expose `detune` for exactly this purpose. Modulating it in cents means the sweep is even on a musical scale, whatever the cutoff.

For tremolo, `trem.gain` is set to `1 - depth/2` and the LFO adds ±`depth/2` to it.

Each voice has its own LFO, so every note's wobble restarts from the beginning of its cycle when the key goes down. This keeps sweeps such as the wobble bass identical from note to note, but notes played together will not wobble in sync the way they would with one shared, free-running LFO.

## 9. Polyphony and note handling

Active voices live in an object `V`, keyed by MIDI note number:

```js
V[m] = { rel, t, held: true };
```

- **Retriggering** a note that is already sounding gives the old voice a very fast release (30 ms) to avoid a click, then starts a new one.
- **Voice stealing:** at 16 voices, the oldest is released quickly. This keeps CPU use bounded.
- **Sustain pedal:** when sustain is on, releasing a key only marks the voice `held: false`. When the pedal comes up, all such voices are released together.

Three input paths share the same `down(id, note, velocity)` and `up(id)` functions:

- **Pointer events** on the on-screen keyboard. `setPointerCapture` lets a finger slide across keys, and `document.elementFromPoint` finds the key under the pointer. Velocity is taken from how far down the key you press.
- **Computer keyboard** via `keydown` and `keyup`, with `e.repeat` ignored so a held key does not retrigger.
- **Web MIDI** through `navigator.requestMIDIAccess()`, handling note on, note off, and the sustain pedal (controller 64).

A source `id` such as `"p1"`, `"kz"` or `"m60"` identifies each finger or key, which is how the keyboard knows which note to release when you lift a finger.

## 10. Live controls and the settings object

Settings live in one plain object, `S`. Sliders write into it directly, and presets replace it:

```js
S = {...D, ...P[name]};
```

Voices read `S` when they start, so changes to oscillators, filter and envelopes affect only **new notes**. Effects are different: `fx()` writes the effect parameters straight onto the shared nodes, so those change immediately. The values are assigned directly (`gain.value = x`), which can produce a faint click when you drag fast. Using `setTargetAtTime` with a short time constant would smooth this.

## 11. The oscilloscope

The scope is not Web Audio itself, only a reader of it. An `AnalyserNode` sits in the chain and, every animation frame, `getByteTimeDomainData` copies the latest 512 samples into a byte array (128 is silence). A canvas then draws them as a line:

```js
an.getByteTimeDomainData(buf);
// draw buf[i]/255 * height for each x position
```

## 12. Limitations and ideas

- **No hard sync or FM.** `OscillatorNode` cannot be synced to another oscillator. FM is possible (connect one oscillator into another's `frequency` param), and an `AudioWorklet` would allow sync.
- **No glide or mono mode.** Adding glide would mean keeping a persistent oscillator and ramping its frequency.
- **Release tails end at about −42 dB.** Sources stop at `1.6 × release`, when the exponential fade has reached roughly 1% of its level. Very long releases might leave a faint cut-off. Waiting 3 time constants more would be cleaner.
- **Shared noise buffers loop every two seconds** and every note starts at the same position, so hats and snares are identical each time.
- **Per-voice nodes.** A unison of seven across three oscillators means 21 oscillators per note. That is fine on a modern machine but would limit polyphony on a phone. An `AudioWorklet` that sums the unison voices internally would be much cheaper.
- **Patch saving** could be added by serialising `S` to JSON.

## Further reading

- MDN, Web Audio API: <https://developer.mozilla.org/docs/Web/API/Web_Audio_API>
- W3C Web Audio API specification: <https://www.w3.org/TR/webaudio/>
