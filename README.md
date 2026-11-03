# Synth Piano for Sight Singing

A single-file, browser-based synth piano with a built-in sight-singing trainer. Everything is generated in code with the Web Audio API: there are no audio files, libraries or build steps.

## Getting started

1. Save `piano.html` anywhere on your computer.
2. Open it in a modern desktop or mobile browser (it uses the Web Audio API, which current versions of Chrome, Edge, Firefox and Safari support).
3. Click or tap a key. Browsers only allow audio after a user gesture, so sound starts on your first interaction.

It works offline and has no dependencies.

## Features

### Keyboard
- Three octaves, C3 to B5 (21 white and 15 black keys), scaling to the window width.
- Mouse and touch input, with glissando (drag across keys) and multitouch chords.
- Computer-keyboard play: the `Z`–`M` and `Q`–`U` rows play upward from C3. `S D G H J` and `2 3 5 6 7` are the black keys.
- Middle C is marked **Do** on the keyboard.

### Sounds (22)
Choose from the **Sound** menu. Piano sounds decay naturally; synth and retro sounds hold while the key is down and fade on release.

| Group | Sounds |
|---|---|
| Pianos | Grand piano, Grand piano (brighter), Bright piano, Mellow piano, Electric piano, Honky-tonk, Music box |
| Synths | Warm pad, Glass pad, Square lead, Synth brass, Organ, Synth pluck, Vibrato lead |
| Retro synths | Chiptune square, 8-bit triangle, Moog-style lead, 80s poly synth, String machine, Supersaw, Sync-style lead, Wobble bass |

### Reverb and volume
- Convolution reverb using an impulse response generated in code (decaying, progressively darker stereo noise).
- **Reverb mix** sets the wet/dry balance (default 35).
- **Room size** sets the tail length from 0.5 to 6 seconds (default 2.8). The reverb is rebuilt when you release the slider.
- **Volume** is the master level.

### Transpose and octave
- **Transpose** shifts the pitch by semitones, from −24 to +24. **Random** picks a new shift between −12 and +12 (always different from the current one), and **Reset** returns to 0.
- **Octave** shifts by whole octaves, from −2 to +2 (default −1). It is separate from Transpose, so Random and Reset never change it.
- The keys on screen never move; only the sounding pitch changes, and only for notes played afterwards.

### Sight-singing trainer
1. Press **Select** and click the keys you want to practise. Each gets a blue dot, and you hear a short preview. Click again to remove a note, or press **Clear**.
2. Set the **Tempo** from 10 to 100 BPM (default 20). The slider is logarithmic, so the slow end is finely adjustable.
3. Press **Play**. On every beat a random selected note lights up green and stays lit until the next beat. The same note is never chosen twice in a row unless it is the only one selected. No jump is larger than the **Max leap** (see below). Press **Stop** to end.
4. **Max leap** sets the largest jump, in semitones, allowed between one note and the next (1 to 35, default 12 = one octave). It is measured between the keys, so Transpose and Octave don't affect it. If no other selected note is within reach, the same note repeats and the status line says so. You can change it while playing.
5. **Play sound** lets you hear the note as well. **Sound delay** sets how long after the light the sound plays, as a fraction of a beat: none, 1/8, 1/4, 1/3, 1/2, 2/3 or 3/4 (default 3/4). This gives you time to sing before you hear the answer. Turn Play sound off to train from sight alone.

### Solfege ladder
- Shows all 12 chromatic tones, counted from middle C = Do = 1, with scale degree, solfege name and key name for each row (for example, `♭3 · Me · D♯/E♭`).
- Bold rows are the major scale; the second solfege name on chromatic rows is the alternate spelling (Di = raised Do = Ra).
- Click and hold a row to hear that pitch.
- Tick **Show the lit note on the ladder** to highlight the current trainer note's row, so you can check your answer. It is off by default.

## Suggested way to practise

- Treat **middle C as home** (1 or Do) whatever the Transpose and Octave settings are. Those settings change the pitch you hear, not which key is home.
- Sing the **name of each note** as it lights up, either as a scale degree (1, 2, 3, ♭3, 4, ♯4 …) or in solfege.
- Start with two or three notes, a slow tempo and a long delay, then add notes and speed up.
- Use **Random** transpose to practise relative pitch: the pattern stays the same while the pitch you hear changes.
- For matching pitch, steady sounds such as Mellow piano, Organ, Glass pad or 8-bit triangle are the easiest to sing along with. Vibrato lead and Wobble bass move in pitch or tone on purpose.

## How it works

The signal path for each note is:

```
oscillators (partials) → low-pass filter → attack/decay envelope → release gain → bus
bus → dry gain ─────────────────────────────────────────────────→ master
bus → convolver (reverb) → wet gain ────────────────────────────→ master
master → compressor → speakers
```

Each sound is a preset in the `INSTRUMENTS` object near the top of the script: a list of partials `[frequency multiple, waveform, level]` plus filter, envelope and optional LFO settings (pitch vibrato or filter wobble). To add or tweak a sound, edit or copy a preset and add an `<option>` for it in the **Sound** menu.

## Limitations

- The sounds are synthesised approximations, not sampled instruments.
- The **Sync-style lead** imitates the sweep of a hard-synced oscillator with a resonant filter; the Web Audio API has no true oscillator sync.
- Changing Transpose, Octave or Sound affects only notes played afterwards; notes already sounding keep their pitch.
- Timing in the trainer uses JavaScript timers, which is accurate enough for practice but not for sample-exact rhythm.
