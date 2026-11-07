# Ear training: which field was played?

A single-file ear training page (`ear-training.html`). It has no dependencies and no server. Open the file in a modern browser and it works.

## The idea

You have named groups of short musical examples (for example "Major triad" or "Tritone up"). Two text fields, **A** and **B**, each list some of those group names.

Each round, the page secretly picks A or B at random. For every name in the chosen field, in order, it plays one randomly chosen example from that group. You listen, guess which field was played, and then check.

Fields can hold one name or several. Comparing "Major triad" with "Minor triad" trains chord quality. Comparing "Major triad, Minor triad" with "Minor triad, Major triad" trains hearing the order of things.

## Quick start

1. Open `ear-training.html` in your browser.
2. Click a row in **Starting points**, or type group names into **Field A** and **Field B** (separated by commas).
3. Press **Play** and listen.
4. Press **It was A** or **It was B** to guess. Your score appears in the status line, and the answer is shown.
5. Press **Next round** for a new random pick.

## Controls

| Control | What it does |
| --- | --- |
| **Play / Stop** | Plays the current round, or stops it. The first Play starts a new round. |
| **Loop** | Repeats the same round with no gap between repeats, until you stop it. |
| **Transpose** | When on, each new round is shifted by a random amount from -6 to +6 semitones. The whole round uses one shift, so the relationships between examples stay intact. The shift is not clamped, so very high or low notes are possible. |
| **It was A / It was B** | Records your guess, updates the score and shows the answer. Only your first guess in a round counts. |
| **Reveal** | Shows which field was played, the transposition, and which example was chosen from each group, without guessing. |
| **Next round** | Discards the current round and starts a new one. |
| **Tempo** | Speed in beats per minute. |
| **Sound** | Keys (electric-piano style), Soft pad, or Organ. Applies from the next Play. |
| **Reverb** | How much reverb you hear. You can move it while playing. |

Replaying or looping always uses the same pick until you press **Next round**. Editing the fields, the library or turning Transpose on or off also discards the current round.

## Available groups

This row lists every group in the library, with the number of examples in each. Click one to add it to whichever field (A or B) you edited last. If you haven't edited either yet, it goes into A.

Group names in the fields are matched ignoring case. An unknown or empty group name shows an error instead of playing.

## Starting points

A scrollable table of suggested A/B pairs, with a row number (`#`), a description, and the contents of Field A and Field B. Click a row to load it into the fields.

To change them, open **Edit starting points**. Write one row per line:

```
Major triad | Minor triad | Basic chord quality
```

That is `field A | field B | description`. The description is optional. Use commas for several groups in one field. Lines starting with `#` are ignored.

The `#` column counts only valid rows, so comments and blank lines don't affect the numbering.

## The library

Open **Edit library** to see and change the examples. A line like `[Major triad]` starts a group, and every line below it is one example (a "file") that the page may pick.

```
[Major triad]
C4+E4+G4/3
F3+A3+C4/3

[Minor 3rd up]
C4/1 Eb4/2
```

### Note format

| Syntax | Meaning |
| --- | --- |
| `C4` | A note. `C4` is middle C. Use `#` or `b` for accidentals (`F#3`, `Bb3`). |
| `/2` | Length in beats. The default is 1. Decimals work (`/.5`, `/1.5`). |
| `+` | Notes played together: `C4+E4+G4/3` is a chord lasting 3 beats. |
| a space | Moves on to the next note or chord. The next one starts when the previous one's length has passed. |
| `R/1` | A rest. |
| `#` at the start of a line | A comment. |

Between the examples of a round (when a field lists several groups), the page adds a one-beat gap.

### Included groups

Minor 2nd up, Major 2nd up, Minor 3rd up, Major 3rd up, Perfect 5th up, Tritone up, Major triad, Minor triad, Diminished triad, Augmented triad, Maj7 chord, Dom7 chord, Min7 chord, Major scale up, Dorian scale up.

Each has two or three examples at different pitches, so you can't rely on one fixed starting note.

### Using your own MIDI files

The page doesn't read MIDI files. If you have MIDI examples, write the notes into this text format: one line per example. Note names plus beat lengths are enough for most intervals, chords and short melodies.

## Synth settings

Open **Synth settings** to see and change how the selected sound is built. Each sound keeps its own settings.

- **Envelope:** attack, decay, sustain and release.
- **Filter:** where the low-pass filter starts and ends (as a multiple of each note's pitch), how fast it closes, and its resonance.
- **Oscillators:** four rows, each with a wave shape (off, sine, triangle, sawtooth, square), a pitch multiple, a detune in cents and a level.
- **Reverb length** in seconds. This changes the reverb straight away. All other changes apply from the next Play.

**Reset this sound to default** restores the selected sound's original values. It doesn't touch the other sounds.

## Saving

Everything you edit is saved automatically in your browser: the library, the starting points, fields A and B, the sound, the reverb amount and the synth settings.

- Saved data belongs to one browser and one file location. Moving the HTML file or opening it in another browser starts from the defaults.
- To keep or share a library, copy the text out of the editing boxes into a text file.
- **Reset library and starting points to defaults** (always visible below the library) restores the original library and starting points. Click it, then click it again within 4 seconds to confirm. It doesn't change your synth settings or the A/B fields.

## Troubleshooting

- **No sound:** browsers only allow audio after you click something on the page. Press Play and check your volume and that the tab isn't muted.
- **"Unknown or empty group":** the name in a field doesn't match a group in the library. Check spelling, or click a name in **Available groups**.
- **"Bad note" or "Bad duration":** a line in the library has a typo. Notes need a letter, an optional `#` or `b`, and an octave number, such as `Eb4`.
- **Edits aren't remembered:** some browsers block local storage for files opened from disk or from a preview. The page still works, but edits are lost on reload.
- **Reverb tail muddies small intervals:** lower the Reverb slider or shorten the reverb length.
