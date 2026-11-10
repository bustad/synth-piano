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
| **Sound** | Which synth sound to use: Keys (electric-piano style), Soft pad, Organ and Strings, plus any sounds you create. Applies from the next Play. |
| **Reverb** | How much reverb you hear. You can move it while playing. |

Replaying or looping always uses the same pick until you press **Next round**. Editing the fields, the library or turning Transpose on or off also discards the current round.

## Identify mode

Use the switch at the top of the page to change between **Compare A/B** (above) and **Identify**. In Identify mode you hear one item and say to yourself what it is, with no clues on screen.

1. Choose a list in the **List** menu.
2. Press **Play** (or **Next round**, which starts a new item and plays it straight away).
3. Press **Reveal** to see the answer and exactly which examples were played.
4. Press **Got it** or **Missed** to keep score, then **Next round**.

Missed items come up more often, and the same item is never picked twice in a row. Loop, Transpose, Tempo, Sound and Reverb work as in Compare mode. The score covers the current session and isn't saved.

### Quiz lists

Open **Edit quiz lists** (it is closed by default so it doesn't give clues). Each list starts with `[List name]`, followed by one item per line:

```
[Chord qualities]
Major triad | Major triad
Dominant seventh | Dom7 chord
Major triad, then minor triad | Major triad, Minor triad
```

That is `description | group names`. Several group names are separated by commas and played in order, as in the A and B fields. The description is what Reveal shows. A line without `|` is read as group names only, and the names are shown instead. Quiz lists are saved with everything else and included in Export, Import and Reset.

## Keyboard shortcuts

| Key | Action |
| --- | --- |
| `Space` or `P` | Play / Stop |
| `L` | Loop on or off |
| `T` | Transpose on or off |
| `A` | It was A (Compare mode) |
| `B` | It was B (Compare mode) |
| `G` | Got it (Identify mode) |
| `M` | Missed (Identify mode) |
| `R` | Reveal |
| `N` | Next round |

The shortcuts are switched off while the cursor is in a text box, a number box or a menu, so typing group names or notes works as usual. They also don't fire when Ctrl, Cmd or Alt is held, so they don't clash with browser shortcuts. Hover over a button to see its key.

## Available groups

This row lists every group in the library, with the number of examples in each. Click one to add it to whichever field (A or B) you edited last. If you haven't edited either yet, it goes into A.

Group names in the fields are matched ignoring case. An unknown or empty group name shows an error instead of playing.

## Starting points

A scrollable table of suggested A/B pairs, with a row number (`#`), a description, and the contents of Field A and Field B. Click a row to load it into the fields.

To change them, open **Edit starting points**. Write one row per line:

```
Basic chord quality | Major triad | Minor triad
```

That is `description | field A | field B`, the same order as the columns in the table. A line with only two parts is read as `field A | field B` with no description. Use commas for several groups in one field. Lines saved in the old order (`field A | field B | description`) are converted automatically the first time you load or import them. Lines starting with `#` are ignored.

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

When a field lists several groups, the examples play back to back with no gap. To add a pause, end an example with a rest such as `R/1`.

### Included groups

Minor 2nd up, Major 2nd up, Minor 3rd up, Major 3rd up, Perfect 5th up, Tritone up, Major triad, Minor triad, Diminished triad, Augmented triad, Maj7 chord, Dom7 chord, Min7 chord, Major scale up, Dorian scale up.

Each has two or three examples at different pitches, so you can't rely on one fixed starting note.

### Using your own MIDI files

The page doesn't read MIDI files. If you have MIDI examples, write the notes into this text format: one line per example. Note names plus beat lengths are enough for most intervals, chords and short melodies.

## Synth settings

Open **Synth settings** to see and change how the selected sound is built. Each sound keeps its own settings. Hover over any setting (or a column heading in the oscillator table) to see a short explanation.

- **Envelope:** attack, decay, sustain and release.
- **Filter:** a low-pass filter with its own envelope. **Start** is the cutoff when a note begins (as a multiple of the note's pitch). With an **attack** above 0 it rises to the **peak**, then **decays** (at the speed set by the decay time) to the **sustain** level while the note is held, and a **release** above 0 moves it back to the start level after the note ends. With attack 0 and release 0, which is the default, it simply glides from start to sustain. **Resonance** boosts the frequencies around the cutoff.
- **Oscillators:** five rows, each with a wave shape (off, sine, triangle, sawtooth, square), a pitch multiple, a detune in cents, a level and a pan position from -1 (left) to 1 (right).
- **Chorus:** an ensemble effect that mixes in three slightly delayed copies whose delay wobbles, spread across the stereo field. Set its mix (0 is off), rate in hertz and depth in milliseconds. Each sound has its own chorus settings, which apply when the next Play starts. Strong chorus makes pitch harder to judge, so keep it modest for ear training.
- **Reverb length** in seconds. This changes the reverb straight away. All other changes apply from the next Play.

The sounds themselves are editable too. Use the name box and buttons at the top of the section:

- **Rename** changes the selected sound's name.
- **Save as new sound** copies the selected sound, with the name in the box, so you can experiment without losing the original.
- **Delete** removes the selected sound (click twice to confirm). At least one sound must remain.
- **Restore built-in sounds** brings back any of Keys, Soft pad, Organ and Strings that you deleted.
- **Reset this sound to default** (built-in sounds only) restores the selected sound's original values. It doesn't touch the other sounds.

## Saving

Everything you edit is saved automatically in your browser: the library, the starting points, fields A and B, the sound, the reverb amount and the synth settings.

- Saved data belongs to one browser and one file location. Moving the HTML file or opening it in another browser starts from the defaults.
- **Export settings** downloads everything above as one file (`ear-training-settings.json`). **Import settings** loads such a file, replacing your current library, starting points, fields, sound, reverb and synth settings. Use these to back up your work, move it to another browser or device, or share it.
- **Reset library, starting points and quiz lists to defaults** (always visible below the library) restores the original library and starting points. Click it, then click it again within 4 seconds to confirm. It doesn't change your synth settings or the A/B fields.

## Troubleshooting

- **No sound:** browsers only allow audio after you click something on the page. Press Play and check your volume and that the tab isn't muted.
- **"Unknown or empty group":** the name in a field doesn't match a group in the library. Check spelling, or click a name in **Available groups**.
- **"Bad note" or "Bad duration":** a line in the library has a typo. Notes need a letter, an optional `#` or `b`, and an octave number, such as `Eb4`.
- **Edits aren't remembered:** some browsers block local storage for files opened from disk or from a preview. The page still works, but edits are lost on reload.
- **Reverb tail muddies small intervals:** lower the Reverb slider or shorten the reverb length.
