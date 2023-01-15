Pently score
============

This document describes Pently's text-based music description
language.  It should be straightforward for a user of LilyPond
notation software or MML tools such as PPMCK to pick up.

Invoking
--------
The Pently assembler takes a score file and produces an assembly
language file suitable for ca65.  Like other command-line programs,
it should be run from a Terminal or Command Prompt.  Double-clicking
it in Finder or File Explorer won't do anything useful.

Usage:

    pentlyas.py [-h] [-o OUTFILENAME] [--periods LENGTH]
                [--period-region {dendy,ntsc,pal}] [-A FREQ]
                [--segment SEGMENT] [--rehearse] [-v] [-W {error}]
                [infilename]

Positional arguments:

* `infilename`  
  Score file to process or `-` for standard input; omit for period
  table only.

Optional arguments:

* `-h`, `--help`  
  Show this help message and exit.
* `-o OUTFILENAME`, `--output OUTFILENAME`  
  Write assembly output to this file.  The default is `-`, for
  standard output.
* `--write-inc INCFILENAME`
  Write title and author metadata as an include file, made mostly of
  macros.
* `--periods NUMSEMITONES`  
  Include an equal-temperament period table in the output;
  `NUMSEMITONES` is usually 64 to 80.
* `--period-region {dendy,ntsc,pal}`  
  Make period table for this region (default: `ntsc`).
* `-A FREQ`, `--period-tuning FREQ`  
  Set the frequency of the A above middle C (`a'`) in the period
  table.  The default is "concert pitch" or 440.0 Hz; some orchestras
  (and NSD.lib) use 442 Hz.  If less than 437.0 on `ntsc`, 433.0 on
  `dendy`, or 405.9 on `pal`, the lowest possible notes may sound
  out of tune.
* `--segment SEGMENT`  
  Place output in this ca65 `.segment`.  Useful if you are stashing
  Pently in its own bank of PRG ROM.
* `--rehearse`  
  Include rehearsal mark data in output.
* `-v`, `--verbose`  
  Print tracebacks and other verbose diagnostics on standard error.
* `-W {error}`, `--warn {error}`  
  Enable warning options.  Currently the only valid warning option
  is `-Werror`, which treats warnings as errors.

Overall structure
-----------------
An **object** is a sound effect, drum, instrument, or pattern.  Each
object has a name, which must follow identifier rules: begin with a
letter and use only ASCII letters, digits, and the underscore `_`.
Objects of different types can share a name, such as a sound effect
called `snare` and a drum called `snare`.

Objects can be defined inside or outside a song.  Objects defined
outside a song are called "global" and can be used by any song in
the score.  Those defined inside a song are scoped to that song.
If an object in a song shares a name with a global object of the
same type, it will hide the global object inside the song.

**Indentation** is not important.  An object definition instead ends
at a keyword that starts another object.  A song definition ends at
a stop (`fine`) or repeat (`da capo` or `dal segno`) command.

A **comment** consists of zero or more spaces, a number sign (`#`)
or two slashes (`//`), and the rest of the line.  The parser ignores
line comments.  A comment cannot follow other things on the same line
because a number sign outside a line comment means a note is sharp,
such as `c#`.

The **`include`** command pastes another text file into a score.
You can use this to refer to a library of sound effects, drums,
instruments, or chord definitions.  The file's path is relative
to the directory containing the file where `.include` appears.

The **`title`, `author`, and `copyright`** commands define the
corresponding field of the NSF or NSFe file.  Traditionally, the
`copyright` field contains the year followed by the publisher.
You can use `artist` as a synonym for `author`.

Defining pitches
----------------
Pently works by setting the period of a tone generator to periods
associated with musical pitches.  On the pulse and triangle channels,
both sound effects and musical notes are defined on a logarithmic
scale in units of semitones above 55 Hz.

The **note names** `c d e f g a b` refer to notes in the C major
scale, or 0, 2, 4, 5, 7, 9, and 11 semitones above the octave's base.

The meaning of `b` can be changed with the `notenames` option.  Both
`notenames english` and `notenames deutsch` allow `c d e f g a h`
for the C major scale.

* `notenames english` (the default) treats `b` the same as `h`, 11
  semitones above `c`.
* `notenames deutsch` treats `b` as `h-` (H flat), 10 semitones
  above `c`.  (English speakers call it a B flat.)

The solfege-inspired names that LilyPond uses for Catalan, Flemish,
French, Portuguese, and Spanish are not supported because of clashes
with rest and length commands from MML.

**Accidentals** add or subtract semitones from a note's pitch:

* Sharp (add a semitone): `a# a+ as ais`
* Flat (subtract a semitone): `ab a- aes`
* Double sharp: `ax a## a++ ass aisis`
* Double flat: `abb a-- aeses`

To specify **octave** changes:

* `>` before the note name or `'` after the note name and accidental
  goes up one octave.
* `<` before the note name or `,` after the note name and accidental
  goes down one octave.

A pitched sound effect or pattern can specify the octave of each
pitch by specifying an octave mode inside the pattern:

* `absolute` means that notes `c` through `h` fall in the octave
  below middle C.  The low octave is `c,` through `h,` or `<c`
  through `<h`, and middle C is `c'` or `>c`.  The lowest note that
  works on an NES is `a,,`, and the highest depends on the size of
  the period table.
* `orelative` assumes that an octave will be in the same octave as
  the previous note.  Octave changes persist onto later notes.
  This behavior is familiar to MML users.
* `relative` guesses the octave by trying to move no more than three
  note names up or down, ignoring accidentals.  A G major scale, for
  example, is `g a h c d e fis g`.  This means you don't need to
  indicate octaves after the first note unless you're leaping a fifth
  or more.  This behavior is familiar to LilyPond users.

The reference pitch for `orelative` and `relative` modes at the start
of a pattern is `f`, the F below middle C.

The `o` command (`o0` through `o7`) changes the reference pitch to
the given octave, where octaves are numbered as in NerdTracker II
and Famitracker.  For example, `o1`, `o2`, and `o3` correspond to
pitches `f,`, `f`, and `f'`.

Sound effects
-------------
Each sound effect has a name and channel type, such as
`sfx player_jump on pulse` or `sfx closed_hihat on noise`.

### Envelopes

The pitch, volume, and timbre may vary over the course of a sound
effect.  The available changes differ based on whether an effect is
for a `pulse`, `triangle`, or `noise` channel type.

**Pulse** pitch works as above, where `c'` represents middle C.
The `timbre` controls the duty cycle of the wave, where `timbre 0`
(12.5%) sounds thin, `timbre 1` (25%) sounds full, and `timbre 2`
(50%; default) sounds hollow.  The volume of each step of the
envelope can be set between 0 (silent) and 15 (maximum).  There are
two pulse channels (`pulse1` and `pulse2`), and sound effects for
pulse channels will play on whichever one isn't already in use.

**Triangle** pitch plays one octave lower than pulse: `c''` plays
a middle C.  It has no timbre control, and the volume control is
somewhat crippled: any volume 1 through 15 produces full power,
but it still determines priority when a note and sound effect are
played at once.

**Noise** pitches work differently.  There are only 16 different
pitches, numbered 0 to 15.  The timbre can be set to `timbre 0`
(hiss; default) or `timbre 1` (buzz), and `volume` behaves the same
as pulse.  In `timbre 0`, the top three pitches (13 to 15) sound the
same as 12 but quieter on authentic NES hardware.  They may sound
more problematic on clones and emulators.

The pitch and timbre in a sound effect may loop, but the volume
cannot, as it controls the length of the sound effect.  Place a
vertical line (`|`, Shift+backslash) before the part of the pitch or
timbre that you want to loop.  Otherwise, the last step is looped.

Steps of an envelope are separated by spaces is one word.  A step
can be repeated with a colon and an integer: `8:3` means `8 8 8`.
By default, a sound effect plays one step every frame, which is 60
steps per second on NTSC or 50 on PAL.  To slow this down, use
`rate 2` through `rate 16`

Sound effect names are exported with the `PE_` prefix, such as
`PE_closed_hihat`. Use these values with `pently_start_sound`.

Examples:

    sfx closed_hihat on noise
    volume 4 2 2 1
    timbre | 0 1
    pitch 12
    
    sfx noise_kick on noise
    volume 10 10 8 6 4 3 2 1
    pitch 10 0

    sfx tri_kick on triangle
    volume 15:3 2 2
    pitch e' c' a f# e

### Drums

Even if you are making an NSF or a music-only ROM, sound effects are
used for percussion.  Each of up to 25 drums in the drum kit plays
one or two sound effects.  It's common on the NES to build drums
out of one noise effect, which has the noise channel to itself, and
one triangle effect, which interrupts the bass line on the triangle
channel.

The following sets up two drums, called `clhat` and `kick`.  The
former plays only one sound effect, the latter two.

    drum clhat closed_hihat
    drum kick noise_kick tri_kick

Drum names must start and end with a letter or underscore (`_`) so
as not to be confused with a note duration.  Drums must not have the
same name as a pitch, which rules out things like `ass`.  Nor can
the last two characters be a digit and `g`, so as not to be confused
with a grace note command.

Instruments
-----------
Like sound effects, instruments are built out of envelopes.  They
have the same `volume` and `timbre` settings as pulse sound effects.
But their `pitch` settings differ: instead of being a list of
absolute pitches, they are a list of transpositions in semitones
relative to the note's own pitch.  For example, up a major third
is 4, while down a minor third is -3.  (They behave the same as an
"Absolute" arpeggio in FamiTracker.)  In addition, the `rate` command
is not recognized in an instrument.

The `timbre` of an instrument played on the triangle channel must be
2, or the note will cut prematurely.

On the last step of the volume envelope, the instrument enters
sustain.  (The portion of the envelope prior to sustain is called
"attack".)  A sustaining note's timbre stays constant, its pitch
returns to the note's own pitch, and its volume stays constant
or decreases linearly over time.  (This means that steps in
an instrument's `timbre` or `pitch` envelope past the attack
*will be ignored.*)  The `decay` command sets the rate of decrease
in volume units per 16 frames, from `decay 0` (no decrease; default)
through `decay 1` (a slow fade) and `decay 16` (much faster).

The `detached` attribute cuts the note half a row early, so that
notes don't run into each other.  This is especially useful with
envelopes that do not decay.

Instrument names are exported with the `PI_` prefix, such as
`PI_piano`. Use these values with `pently_play_note`.

Example:

    # Not specifying anything will make an instrument with all
    # default settings: timbre 2, pitch 0, volume 8, decay 0,
    # and no detached, suitable for e.g. triangle channel use.
    instrument bass

    instrument flute
    timbre 2
    volume 3 6 7 7 6 6 5
    
    instrument piano
    timbre 2 1
    volume 11 9 8 8 7 7 6
    decay 1
    
    instrument banjo
    timbre 0
    volume 12 8 6 5 4 4 3 3 2
    decay 1
    
    instrument tub_bass
    timbre 1 1 2
    pitch 6 3 2 1 1 0
    volume 4
    decay 2
    
    # An instrument like this is useful for the attack track
    instrument one_frame_pop
    volume 8 0

Chords
------
Arpeggio is rapid alternation among two or three pitches to create a
warbly chord on one pulse or triangle channel.  It can be specified
using a nibble pair, or two hexadecimal digits representing semitone
intervals.  For example, `37` represents a minor chord as three and
seven semitones above the lowest note.

Arpeggio can also be specified using a chord name. These nine chords
are predefined:

* `OF`: `00`, turn off arpeggio
* `M`: `47`, major (see note)
* `dom7`: `4A`, dominant 7th
* `maj7`: `4B`, major 7th
* `aug`: `48`, augmented
* `m`: `37`, minor
* `m7`: `3A`, minor 7th
* `dim`: `36`, diminished
* `dim7`: `39`, diminished 7th

(Note: In LilyPond's chord mode, major is the default chord, and
`maj` is a confusing synonym for `maj7`.  Pently does not define
`maj`, instead defining `M` and `maj7`.)

To define an additional chord name, use the `@EN` (define chord)
command at the top level, in a song, or in a pattern.  For example,
`@ENsus4 = 57` defines `sus4` as `57`.

A nibble pair or chord name may be preceded by a minus sign.  This
causes notes to be transposed down by the highest interval in the
chord.  For example, both `c:M` and `g:-M` spell a C major chord,
the first based on the root, and the second based on the highest
note.  This may prove convenient for writing multipart harmony.

A chord may be inverted, which moves one or two notes on the bottom
up by an octave to the top.  This is specified with `/1` or `/2`
after the nibble pair or chord name.  To understand how this works,
the first inversion of a dominant 7th (`dom7/1`, `4A/1`) is `68`:

1. All three intervals of original chord: `04A`
2. Replace all `0` with `C`: `C4A`
3. Subtract the lowest interval (`4`) from all intervals: `806`
4. Rotate to left until `0` is in first position: `068`

Inversion requires both intervals to be less than an octave.
It is an error to invert a chord containing `C`, `D`, `E`, or `F`.

Patterns
--------
These are where the notes go.

A pattern contains a musical phrase that repeats until stopped.
A single pattern can be reused with different instruments or on
different channels (except noise vs. non-noise).

### Pattern header

A **pattern** starts with `pattern some_name`.  The compiler detects
whether a pattern is a pitched pattern or a drum pattern by whether
the first note looks like a pitch or a drum name.  Optionally a
pitched pattern can have a default instrument and/or a default track:
`pattern some_name on pulse2 with flute`

The `time` command sets the **time signature**, which controls
the number of beats per measure and the duration of one beat
as a fraction of a whole note, separated by a slash (`/`).  The
denominator (second number) must be a power of 2, no less than 2
and no greater than 64.  For example, `time 2/4` puts two beats in
each measure, each as long as a quarter note.

Pently recognizes two notations for compound prolation, meaning a
beat divisible by three instead of two.  One, inspired by the music
education work of _Carmina Burana_ composer Carl Orff, is a dotted
denominator.  The other is a numerator that is a multiple of 3
greater than 3, such as 6 or 9, which makes the the beat as long as
three units of the denominator.  Both `time 2/4.` and `time 6/8`
set the beat to a dotted quarter note and put two in each measure.

A few time signatures have shortcut notations:

* `time c` means `time 4/4` (common time).
* `time ¢` means `time 2/2` (cut time or alla breve).
* `time o` means `time 3/4` (perfect time).

The **`scale`** command sets what note value shall be used as a
_row_, the smallest unit of musical time in Pently.  It must be a
power of two, such as `scale 8`, which sets eighth notes as the
shortest duration, or `scale 32`, which sets thirty-seconds as the
shortest duration.  A larger `scale` will cause durations of 24 rows
or longer to use more bytes.  The default is `scale 16`.

### Notes

Each note command consists of up to six parts:

* Note name
* Accidentals (optional)
* Octave (optional)
* Duration (optional)
* Chord (optional)
* Slur (optional)

For pitched patterns, the note name, accidentals, and octave are
specified the same way as for sound effects.  For drum patterns,
one of the names defined in a `drum` command is used instead.
These commands can be used instead of a note:

* `r` (rest) or `p` (pause) cuts the current note.
* `w` (wait) does not change the pitch or restart the note.
  This represents a note tied to the previous note.
* `q` repeats the previous chord, skipping any intervening notes
  that have no chord.
* `l` (length) does not play a note or rest but sets the duration
  for subsequent notes, rests, and waits in a pattern that lack
  their own duration.
* `|` performs a bar check.  If the musical time so far in this
  pattern is not a multiple of a measure, it emits a warning.  A new
  pattern inherits starting time from the point in the song where
  it is defined, or it can be set with the `pickup` command.

Note durations are fractions of a whole note, whose length depends
on the `scale`.  Recognized note durations include `1`, `2`, `4`,
`8`, `16`, and `32`, so long as it isn't shorter than one row.
For example, `e4` is half as long as `e2`.  Durations may be
augmented by 50% or 75% by adding `.` or `..` after the number.

Duration is optional for each note, repeated chord, rest, or wait.
The `durations` command controls how missing durations are assigned.

* In `durations temporary` (the default), as in MML, numbers after
  a note change the duration only for that note.  Only `l` commands
  affect later notes' implicit duration.
* In `durations stick`, numbers after a note apply to later notes
  in the pattern, as in LilyPond.

Unless otherwise specified, the first notes in a pattern last one
beat as defined by `time`.

The `g` (grace note) command sets a note's duration in frames
(1/60 second) instead of rows, with the following note taking
the remainder of the row.  For example, `d4g e4 d2` produces a short
D (lasting four frames), an E taking the remainder of the quarter
note, followed by a D half note.  Grace note durations never stick.
Be aware that grace notes longer than one row have poorly specified
effects, particularly with the 20 percent longer frames of PAL.

Chord is described below in the "Pattern effects" section.

A note followed by a tilde `~` will not be retriggered but instead
will be slurred into the following note.  A note followed by a left
parenthesis `(` will be slurred into the following notes, and a note
followed by a right parenthesis `)` represents the end of such a
slurred group.  This is useful for tying notes together or producing
legato (HOPO).  Slurring into a note with the same pitch is the same
as a wait: `eb2~ eb8`, `eb2( eb8)`, and `eb2 w8` mean the same.

Notes are separated by at least one space.  Notes are added to a
pattern until it is closed by one of these commands: `title`,
`author`, `copyright`, `drum`, `song`, `segno`, `fine`, `dal segno`,
`da capo`, `at`, `attack`, `play`, `stop`, or `fallthrough`.

If a pattern ends with the `fallthrough` command, playback continues
into the following pattern in the score.  For example, if pattern
`AAA` falls through and the next pattern is `BBB`, then playing `AAA`
produces `AAA`, `BBB`, `AAA`, `BBB`, ..., while playing `BBB`
produces `BBB`, `BBB`, ...

A simple pattern might look like this:

    pattern melody with violin on pulse2
      orelative
      c'4. c g g a a g2.
      f4. f e e d d c2.
      g4. g f f e e d2.
      g4. g f f e e d2.

**TODO:** A future version of Pently may introduce a command to
modify durations in compound prolation for a swing feel.

**TODO:** A future version of Pently may introduce a command to
automatically introduce rests between notes for staccato feel.

## Pattern effects

To change the **instrument** within a pitched pattern, use `@`
followed by the instrument name, such as `@piano`.  Notes before the
first change use the instrument specified in the song's play command.
If a pattern repeats, notes before the first change use the
instrument specified in the last change.

To set the **arpeggio** for a single note in a pitched pattern,
add `:` after the note's pitch and duration followed by the chord
name or intervals of the arpeggio.
For example, `eb'2.:47` makes a dotted half note E flat major chord,
and `c:m` makes its relative minor, a C minor chord.

To set the arpeggio for all subsequent notes in a pattern, use the
`EN` command followed by a chord name or intervals, such as `EN4A`
or `ENdom7` for dominant 7th chords or `EN00` or `ENOF` to turn off
arpeggio.  (LilyPond users: This means you enter chord mode with
`ENM` and leave chord mode with `ENOF`.)  To make the pitch change
every 1 frame (fast), use `ENP1`; to make it change every 2 frames
(slow), use `ENP2`.

**Vibrato** is a subtle pitch slide up and down while a note is held.
The `MP` (modulate period) command controls vibrato: `MP1` through
`MP4` set depth between 1 (9 cents, very subtle) and 4 (75 cents,
very strong), and `MP0` or `MPOF` disables it.  Only the depth can be
controlled, not the rate (which is fixed to a musically practical
12-frame period).  Because of a 2A03 quirk, a few pitches played with
vibrato on a pulse channel may cause audible jitter: `a'`, `a`, `d`,
`a,`, `f,`, `d,`, and `h,,`.

**Portamento**, also called **pitch bend** or **pitch slide**, causes
a channel's pitch to approach a played note gradually rather than
immediately changing from the previous pitch.  The following commands
set the rate of this change:

* `EP00`, `EPOF`: Snap to target pitch (default)
* `EP01` through `EP0F`: Change by 1 to 15 semitones per frame
* `EP10` through `EP1A`: Change by an increasing fraction of a
  semitone per frame
* `EP20` through `EP27`: Change by a decreasing fraction of the
  distance to the target pitch per frame, like Roland TB-303

The same pitches that cause jitter for vibrato also cause jitter when
portamento on a pulse channel crosses them.

**Channel volume** scales the volume of the instrument's envelope.
Commands `v1`, `v2`, `v3`, and `v4` (or synonyms `pp`, `mp`, `mf`,
and `ff`) change the channel volume to 25%, 50%, 75%, and 100%
respectively.  Volume is set to 100% at the start of a piece.
Drums and other sound effects are unaffected.

Songs
-----
Like patterns, songs also have `time` and `scale`.  They are used
to interpret the `tempo` and `at` commands.

The `song` command begins and names a song.  Song names are exported
with the `PS_` suffix, such as `PS_twinkle`.  Use these values with
`pently_start_music`.

The **`title` and `author`** commands can be used within a song.
NSFe files reflect each song's title and author, and the NES file
displays a list of titles.  If a title or author is not specified,
the song's object name or the NSF's author name is used.

Patterns may be defined inside or outside a song.  A pattern defined
inside a song inherits the song's `time` and `scale`.  If a pattern
is defined outside a song, and its `scale` does not match that
of the song, it will be played with rows in all tracks the same
duration, which may or may not be what you want.

The **`tempo`** command tells how many beats are played per minute.
This can be a decimal, which will be rounded to the nearest whole
number of rows per minute.  For example, a song in `time 6/8` and
`scale 16` will have 6 rows per beat; `tempo 100.2` would then
map to 601.2 rows per minute, which is rounded to 601.  A tempo
that maps to more than 1500 rows per minute is forbidden because
it would cause a row to be shorter than two frames.

An optional note duration can be given before the tempo value.
This causes the `tempo` command to convert the tempo value from
notes of that duration per minute to beats per minute.  For example,
if the current beat is a quarter note, `tempo 8=192` converts 192
eighth notes per minute to 96 beats per minute.  Dotted durations,
such as `8.`, are valid here as well.  Because it performs the
conversion based on the current scale and time signature, the score
should specify `time` and `scale` before a duration-scaled `tempo`.

The **`at`** command waits for a `measure:beat:row` combination,
where measures and beats are numbered from 1 and rows from 0, before
processing the following command.  Row is optional; beat is also
optional if row is unspecified.  Any command may be specified on
the same line immediately following the timecor the following line.
As in Chinese films and Charles Stross novels, an `at` that goes
back in time is forbidden.

If a song begins on an upbeat, you can add a **`pickup`** measure.
The `pickup` command sets what the parser thinks is the current beat,
so that the `at` command knows how many rows to wait.  For example,
in a piece in 3/4 that starts on the third beat, use `pickup 0:3`,
where `0` means the measure preceding the first full measure, and
`3` means the third beat.

The `pickup` command also works in patterns for bar check use.
If defining a pattern within a song, make sure to specify `pickup`
outside a pattern definition, such as before the start of the first
pattern or after a command that closes a pattern.

In addition to the tracks for pitched channels (`pulse1`, `pulse2`,
and `triangle`) and the drum track, Pently has an **attack track**
that interrupts a sustaining pitched note on another channel with an
attack envelope.  Like sound effects, the attack track uses illusory
continuity to increase the apparent polyphony.  An instrument played
on an attack track must have an attack phase.  (This means its
`volume` must be longer than one step because the last step belongs
to sustain, not attack.)  To select a channel for the attack track,
use `attack on pulse1`, `attack on pulse2`, or `attack on triangle`.
(There is no channel called `titan`.)  It's not recommended to use
attack on the same channel as the sound effects that make up drums
unless all the drums have a cutout for the attack.

To **play a pattern,** use `play pattern_name`.  Pitched patterns
default to the `pulse2` track; to specify another track, add
`on pulse1`, `on triangle`, or `on attack`.  Patterns can be played
with a particular instrument or transposed up or down a number of
semitones.  For example, transposing a pattern on `triangle` up
an octave (12 semitones) counteracts the channel's inherent
transposition.

    play melody_a with fiddle
    play bass_a on triangle up 12
    play melody_b with flute on pulse1 up 7

Drum patterns may be played only on the drum track and do not take a
`with` or `on` parameter.

A `play` command immediately replaces the pattern playing on a track.
To play one pattern after the other, use an `at` command to wait for
the pattern to end.  The pattern will loop until it is stopped or
another pattern is played on the same track.

You can play a single pitch on a channel directly from the song:

    play 10 with crash_cymbal on noise
    play c#' with pad on pulse1

Unlike drum patterns, noise notes in the song use an instrument and
a pitch from 0 to 15.  They're good for crash cymbals and the like,
as they can be interrupted by other drums.  But make sure to play
single notes _after_ patterns in the same `at` block, or the
instrument may change unexpectedly.

To **stop the pattern** playing on a track, switch it to a built-in
silent pattern using `stop pulse1`, `stop pulse2`, `stop triangle`,
`stop drum`, or `stop attack`.  You can stop more than one track:

    stop pulse1 pulse2 drum

Playing a pattern made of only rests and waits produces a warning
on a pitched track or an error on a drum track.  The only known use
of a pattern made of waits, as opposed to using `stop`, is to let a
program using Pently generate and play notes at runtime.

The loop point is set with the **`segno`** (sen-yoh) command.  A song
ends with the **`fine`** (fee-neh) command, which stops playback, or
the **`dal segno`** command, which loops back to `segno` if it exists
or the beginning of the song otherwise.  When there is no `segno`,
the `da capo` command also works.

The included `musicseq.pently` file contains examples of complete songs.

### Rehearsal

The following commands primarily control playback in the NES ROM
player (`pently.nes`).  They make the edit-build-listen cycle more
convenient, but they're not quite as useful in games or NSF output.
Thus they are included in output only if `--rehearse` is specified.

To add a **rehearsal mark** for navigation within a song, use the
`mark` command after an `at`.  Each song can have up to 16 marks,
including the automatic marks for the start and loop point.
Each mark has a name of 1 to 25 ASCII (basic Latin) characters:

    at 65
    mark cadenza

The **`resume`** command starts playback in `pently.nes` from a
specific point in one of the songs in a score.  The **`mute`**
command followed by one or more track names (as with `stop`) causes
tracks to be muted when `pently.nes` starts; **`solo`** is similar
but mutes all tracks other than those specified.  A score can contain
only one `resume` and only one `mute` or `solo`.

Output file
-----------
The result of the Pently assembler is a ca65 assembly language file
that depends on macros in `pentlyseq.inc`.  It contains definitions
of all objects, as well as comments stating the size of each object,
the size of all objects of a particular type, and the size of objects
associated with each song.

If you are using the included makefile to iterate on a composition,
keep in mind that Make deletes certain intermediate files after the
ROM is built.  Thus if you're interested in the assembly language
output, you'll need to tell Make to build it and not delete it.
For example, if the `audio` folder contains `Example.pently` or
`Example.ftm`, try this:

    make obj/nes/Example.s

(Keep in mind that Make is sensitive to uppercase and lowercase
letters in filenames, even if run on a Windows system that otherwise
is not.)
  
Glossary
--------
Many of the following terms will be familiar to somebody who has
studied music theory and MIDI.

* 2A03: An integrated circuit in the Nintendo Entertainment System
  Control Deck.  It consists of a second-source version of the
  MOS 6502 CPU, a DMA unit for the sprite display list, four tone
  generators, and a sampled audio playback unit.  Pently uses the
  CPU to send commands to the tone generators.
* 2A07: Variant of 2A03 used in the PAL NES sold in Europe.
* 6527P: Variant of 2A03 used in PAL famiclones, such as Micro Genius
  and Dendy.
* Attack: The beginning of an envelope.  It consists of all volume
  envelope steps except the last.
* Bar: The line in musical notation that separates one measure from
  the next.  Can also mean a measure itself.
* Cent: 1/100 of a semitone.  An octave is 1200 cents, and a pitch
  difference of less than six cents is considered inaudible.
* Channel: An output device capable of playing one tone at once.
  The 2A03 contains four channels: `pulse1`, `pulse2`, `triangle`,
  and `noise`.
* Channel type: A set of channels with the same behavior.  The 2A03
  has three channel types: `pulse`, `triangle`, and `noise`.
* Downbeat: The first beat of a measure.
* DPCM: Delta pulse code modulation, representing a waveform as a
  series of samples increasing or decreasing by 1/63 of full scale
  from the previous sample.  Used for drums in some NES games.
* Drum: A sound effect played to express rhythm.  Usually represents
  unpitched percussion.
* Envelope: The change in pitch, volume, or timbre over the course
  of a single note or sound effect.
* Frame: The fundamental unit of time, on a scale comparable to the
  progress through an envelope, too fast for rhythmic significance.
  Like other NES music engines, Pently counts frames based on the
  vertical retrace of the picture generator.  An NTSC PPU produces
  60.1 frames per second, and a PAL PPU produces 50.0 frames per
  second.  This usually ends up assigning three to fifteen frames
  per row depending on the tempo and scale.
* HOPO: Instantaneous change in a note's pitch.  (After guitar
  techniques called "hammer-on" and "pull-off" that produce this.)
* Illusory continuity: The tendency of the human auditory system to
  fill in gaps in a continuous tone when these gaps coincide with
  another sufficiently loud tone or noise.
* Instrument: A set of pitch, volume, and timbre envelopes that is
  used to play notes.
* Note: A musical event with a pitch and a duration.
* Note value: A duration expressed as a binary fraction of a
  whole note.
* Octave: A pitch difference corresponding to a frequency ratio of
  2 to 1.  In Western music, note names repeat at the octave.
* Pattern: A musical phrase, consisting of a list of notes and rests.
* Pickup measure: A partial measure at the start of a piece of music,
  which may begin on an upbeat or a fractional beat.  Also called
  "anacrusis".
* Pitch: The frequency of a tone expressed using a logarithmic scale.
* Pitched: Relating to a channel with a `pulse` or `triangle` type,
  which plays pitches rather than noise.
* Polyphony: Playing more than one note at once.
* Prolation: The division of a beat into two parts (simple) or
  three parts (compound).
* Rest: A musical event consisting of silence for a duration.
* Row: The shortest rhythmically significant duration in a piece
  of sequenced music.  Also called a subdivision or tatum (after
  American jazz pianist Art Tatum).
* Semitone: A pitch difference of one-twelfth of an octave,
  corresponding to the frequency ratio 1.0595 (the twelfth root of 2)
  to 1.
* Song: A piece of music, which plays patterns at various times.
* Sound effect: A set of pitch, volume, and timbre envelopes
  without necessarily a definite pitch.
* Tempo: The speed at which music is played back, expressed in beats
  per minute.
* Timbre: The quality of a sound independent of its volume, pitch,
  or duration, and determined by its harmonic structure.
* Time signature: A fraction determining the number of beats in a
  measure and the note value corresponding to one beat.
* Track: A logical structure on which notes can be played.  Pently
  has five tracks: one for each pitched channel, one more that can
  replace the attack on a pitched channel's track, and a drum track.
* Upbeat: A beat other than a downbeat.
* Whole note: The name in American English, German, Greek, Japanese,
  and other languages for a note whose duration is that of a measure
  of common (4/4) time.  Also called "semibreve" in Italian and
  British English, or words meaning "round" in Catalan, French,
  and Spanish.
