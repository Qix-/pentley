<!DOCTYPE HTML><html lang="en"><head><title>Pently</title><meta charset="utf-8"><style type="text/css">
html { background: #888; }
body { margin: 0 auto; width: 35em; padding: 1em 2em; background: #FFF; color: #333; border-radius: 1em; line-height: 1.5em }
h1,h2,h3,h4,h5,h6 { line-height: 1.2em }
h1 { margin-top: 0 }
td.numalign { text-align: right }
table.datatable { border: 1px solid #888; border-collapse: collapse; margin: 1em auto }
table.datatable tr { vertical-align: top }
table.datatable th { border: 1px solid #CCC; padding: 0.2em; background: #EEE }
table.datatable td { border: 1px solid #CCC; padding: 0.2em }
tr.odd { background: #EEE }

ul#toc { margin-left: 0; padding-left: 0; line-height: 1.8em }
ul#toc li { display: inline; list-style-type: none; white-space: nowrap }
ul#toc li a { padding: 0.2em 0.5em; border-radius: 0.5em }
ul#toc li a:link, ul#toc li a:visited { color: #00C; quotext-shadow: 0 1px 2px #FFF; box-shadow: 0 1px 2px rgba(0,0,0,.4) }
ul#toc li a:hover, ul#toc li a:focus { background: #EEE; box-shadow: 0 1px 2px rgba(0,0,0,.6) }
</style></head><body>
<h1>Pently</h1>
<p>
This document describes Pently, the audio engine used in Pin&nbsp;Eight NES games since 2009.
</p>
<ul id="toc">
<li><a href="#Introduction">Introduction</a></li>
<li><a href="#API">API</a></li>
<li><a href="#Pitch">Pitch</a></li>
<li><a href="#Sound_effects">Sound effects</a></li>
<li><a href="#Instruments">Instruments</a></li>
<li><a href="#Conductor_track">Conductor track</a></li>
<li><a href="#Patterns">Patterns</a></li>
<li><a href="#Bugs_and_limits">Bugs and limits</a></li>
</ul>

<h2 id="Introduction">Introduction</h2>
<p>
Pently is a music and sound effect player code library for use in games for the Nintendo Entertainment System written in assembly language with ca65. It has seen use in NES games dating back to 2009, including <em>Concentration Room</em>, <em>Thwaite</em>, <em>Zap Ruder</em>, the menu of <em>Action 53</em>, <em>Double Action Blaster Guys</em>, <em>RHDE: Furniture Fight</em>, and <em>Sliding Blaster</em>.
</p><p>
The name comes from Polish <em lang="pl">pętla</em> meaning a loop. It also reminds one of Greek <em lang="el">πέντε (pénte)</em> meaning "five", as it supports five tracks (pulse 1, pulse 2, triangle, drums, and attack injection) mapped onto the NES audio circuit's four tone generator channels.
</p>

<h2 id="API">API</h2>
<p>
The following methods, declared in the assembly language include file <code>pently.inc</code>, make up the public <abbr title="application programming interface">API</a>:
</p><dl>
<dt><code>pently_init</code></dt>
<dd>Initializes all sound channels. Call this at the start of a program or as a "panic button" before entering a long stretch of code where you don't call <code>pently_update</code>.</dd>
<dt><code>pently_start_sound</code></dt>
<dd>Plays a sound effect, element A from the <code>psg_sound_table</code>. Watch out: this trashes RAM <code>$0000</code> through <code>$0004</code>, so be careful to save this data if you are calling this method from game logic.</dd>
<dt><code>pently_update</code></dt>
<dd>Updates the sound channels. Call this once each frame.</dd>
<dt><code>pently_start_music</code></dt>
<dd>Starts to a song, element A from the <code>songTable</code>.</dd>
<dt><code>pently_stop_music</code></dt>
<dd>Stops the song, allowing sound effects to continue.</dd>
<dt><code>pently_resume_music</code></dt>
<dd>Resumes the playing song. Calling <code>pently_resume_music</code> without having first called <code>pently_start_music</code> or after a <a href="#fine"><code>fine</code></a> results in undefined behavior.</dd>
<dt><code>pently_play_note</code></dt>
<dd>Plays note A (see <a href="#Pitch">pitch table</a>) on channel X (0, 4, 8, 12, or 16) with instrument Y from <code>pently_instruments</code>.</dd>
<dt><code>getTVSystem</code></dt>
<dd>Waits for two NMIs, counting the time between them. Returns 0 in A for NTSC systems, 1 for PAL NES, or 2 for <a href="https://en.wikipedia.org/wiki/Dendy_(console)" target="wikipedia">Dendy</a>-style PAL famiclones. Make sure your NMI handler finishes within 1500 or so cycles (not taking the whole NMI or waiting for sprite 0) while calling this, or the result in A will be wrong.</dd>
<dt><code>pently_get_beat_fraction</code></dt>
<dd>Reads the fraction of the current beat. Returns a value from 0 to 95 in A.</dd>
</dl><p>
Your makefile will need to assemble <code>pentlysound.s</code>,
<code>pentlymusic.s</code>, and <code>musicseq.s</code>, and link them into
your program.  If using <code>getTVSystem</code>, additionally assemble
<code>paldetect.s</code>.  If using <code>pently_get_beat_fraction</code>,
additionally assemble <code>math.s</code> and <code>bpmmath.s</code>.
If not generating into the period table into <code>musicseq.s</code>,
additionally assemble <code>ntscPeriods.s</code>.
</p><p>
The file <code>musicseq.s</code> contains the sound effects,
instruments, songs, and patterns that you define. It should
<code>.include "pentlyseq.inc"</code> to use the macros described
below. For those familiar with
<a href="http://www.nullsleep.com/treasure/mck_guide/">Music Macro
Language (MML)</a> or <a href="http://lilypond.org/">LilyPond</a>,
the distribution includes a processor for an MML-like music description
language. (For more information, see <code>pentlyas.md</code>.)
</p>
<h3>Configuration</h3>
<p>
The file <code>pentlyconfig.inc</code> contains symbol definitions that
enable or disable certain features of Pently that take more ROM space or
require particular support from the host program. A project using a feature
can enable it by setting the symbol associated with the feature to a nonzero
number (<code>PENTLY_USE_this = 1</code>). A project not using a feature,
especially an NROM-128 project in which ROM space is at a premium, can turn
it off by setting its symbol to zero (<code>PENTLY_USE_this = 0</code>).
</p><p>
If <code>PENTLY_USE_ROW_CALLBACK</code> is enabled, your code must provide and
<code>.export</code> two callback functions: <code>pently_row_callback</code>
and <code>pently_dalsegno_callback</code>. These are called before each
row is processed and when a <a href="#dalSegno"><code>dalSegno</code></a>
or <a href="#fine"><code>fine</code></a> is processed, respectively.
They can be useful for synchronizing animations to music.
For <code>pently_dalsegno_callback</code>, carry is clear at the end
of a track or set if looping.
</p><p>
If <code>PENTLY_USE_PAL_ADJUST</code> is enabled, the main program must
<code>.export</code> a 1-byte RAM variable called <code>tvSystem</code>.
Pently includes a subroutine to detect the TV system by measuring the
length of a frame in cycles. An NES program that includes the file
<code>paldetect.s</code> can call <code>getTVSystem</code> and store
the result into <code>tvSystem</code> before calling Pently subroutines.
In NSF, the NSF shell stores the TV system in <code>tvSystem</code>.
</p><p>
Disabling <code>PENTLY_USE_ARPEGGIO</code> saves about 60 bytes, and
disabling <code>PENTLY_USE_VIBRATO</code> saves about 150.
</p>

<h2 id="Pitch">Pitch</h2>
<p>
Pently expresses pitch in terms of a built-in table of wave periods in <a href="https://en.wikipedia.org/wiki/Equal_temperament" target="wikipedia">equal temperament</a> (12edo). The following values are valid for the square wave channels; the triangle wave channel always plays one octave lower. The player automatically compensates for different APU speeds based on bit 0 of the <code>tvSystem</code> variable (0: NTSC NES or Dendy famiclone; 1: PAL NES).
</p><p>
Because of the NES's limited precision for wave period values, note frequencies become less precise at high pitches. These frequencies apply to NTSC playback:
</p><table class="datatable">
<caption>Pitch values</caption>
<tr><th>Value</th><th>Name</th><th>Frequency (Hz)</th></tr>
<tr><td class="numalign">0</td><td>A1</td><td class="numalign">55.0</td></tr>
<tr class="odd"><td class="numalign">1</td><td>A#1/B&#9837;1</td><td class="numalign">58.3</td></tr>
<tr><td class="numalign">2</td><td>B1</td><td class="numalign">61.7</td></tr>
<tr><td class="numalign">3</td><td>C2</td><td class="numalign">65.4</td></tr>
<tr class="odd"><td class="numalign">4</td><td>C#2/D&#9837;2</td><td class="numalign">69.3</td></tr>
<tr><td class="numalign">5</td><td>D2</td><td class="numalign">73.4</td></tr>
<tr class="odd"><td class="numalign">6</td><td>D#2/E&#9837;2</td><td class="numalign">77.8</td></tr>
<tr><td class="numalign">7</td><td>E2</td><td class="numalign">82.4</td></tr>
<tr><td class="numalign">8</td><td>F2</td><td class="numalign">87.3</td></tr>
<tr class="odd"><td class="numalign">9</td><td>F#2/G&#9837;2</td><td class="numalign">92.5</td></tr>
<tr><td class="numalign">10</td><td>G2</td><td class="numalign">98.0</td></tr>
<tr class="odd"><td class="numalign">11</td><td>G#2/A&#9837;2</td><td class="numalign">103.9</td></tr>
<tr><td class="numalign">12</td><td>A2</td><td class="numalign">110.0</td></tr>
<tr class="odd"><td class="numalign">13</td><td>A#2/B&#9837;2</td><td class="numalign">116.5</td></tr>
<tr><td class="numalign">14</td><td>B2</td><td class="numalign">123.5</td></tr>
<tr><td class="numalign">15</td><td>C3</td><td class="numalign">130.8</td></tr>
<tr class="odd"><td class="numalign">16</td><td>C#3/D&#9837;3</td><td class="numalign">138.6</td></tr>
<tr><td class="numalign">17</td><td>D3</td><td class="numalign">146.8</td></tr>
<tr class="odd"><td class="numalign">18</td><td>D#3/E&#9837;3</td><td class="numalign">155.6</td></tr>
<tr><td class="numalign">19</td><td>E3</td><td class="numalign">164.7</td></tr>
<tr><td class="numalign">20</td><td>F3</td><td class="numalign">174.5</td></tr>
<tr class="odd"><td class="numalign">21</td><td>F#3/G&#9837;3</td><td class="numalign">184.9</td></tr>
<tr><td class="numalign">22</td><td>G3</td><td class="numalign">195.9</td></tr>
<tr class="odd"><td class="numalign">23</td><td>G#3/A&#9837;3</td><td class="numalign">207.5</td></tr>
<tr><td class="numalign">24</td><td>A3</td><td class="numalign">220.2</td></tr>
<tr class="odd"><td class="numalign">25</td><td>A#3/B&#9837;3</td><td class="numalign">233.0</td></tr>
<tr><td class="numalign">26</td><td>B3</td><td class="numalign">246.9</td></tr>
<tr><td class="numalign">27</td><td>C4 (middle C)</td><td class="numalign">261.4</td></tr>
<tr class="odd"><td class="numalign">28</td><td>C#4/D&#9837;4</td><td class="numalign">276.9</td></tr>
<tr><td class="numalign">29</td><td>D4</td><td class="numalign">293.6</td></tr>
<tr class="odd"><td class="numalign">30</td><td>D#4/E&#9837;4</td><td class="numalign">310.7</td></tr>
<tr><td class="numalign">31</td><td>E4</td><td class="numalign">330.0</td></tr>
<tr><td class="numalign">32</td><td>F4</td><td class="numalign">349.6</td></tr>
<tr class="odd"><td class="numalign">33</td><td>F#4/G&#9837;4</td><td class="numalign">370.4</td></tr>
<tr><td class="numalign">34</td><td>G4</td><td class="numalign">392.5</td></tr>
<tr class="odd"><td class="numalign">35</td><td>G#4/A&#9837;4</td><td class="numalign">415.8</td></tr>
<tr><td class="numalign">36</td><td>A4</td><td class="numalign">440.4</td></tr>
<tr class="odd"><td class="numalign">37</td><td>A#4/B&#9837;4</td><td class="numalign">466.1</td></tr>
<tr><td class="numalign">38</td><td>B4</td><td class="numalign">495.0</td></tr>
<tr><td class="numalign">39</td><td>C5</td><td class="numalign">522.7</td></tr>
<tr class="odd"><td class="numalign">40</td><td>C#5/D&#9837;5</td><td class="numalign">553.8</td></tr>
<tr><td class="numalign">41</td><td>D5</td><td class="numalign">588.7</td></tr>
<tr class="odd"><td class="numalign">42</td><td>D#5/E&#9837;5</td><td class="numalign">621.4</td></tr>
<tr><td class="numalign">43</td><td>E5</td><td class="numalign">658.0</td></tr>
<tr><td class="numalign">44</td><td>F5</td><td class="numalign">699.1</td></tr>
<tr class="odd"><td class="numalign">45</td><td>F#5/G&#9837;5</td><td class="numalign">740.8</td></tr>
<tr><td class="numalign">46</td><td>G5</td><td class="numalign">782.2</td></tr>
<tr class="odd"><td class="numalign">47</td><td>G#5/A&#9837;5</td><td class="numalign">828.6</td></tr>
<tr><td class="numalign">48</td><td>A5</td><td class="numalign">880.8</td></tr>
<tr class="odd"><td class="numalign">49</td><td>A#5/B&#9837;5</td><td class="numalign">932.2</td></tr>
<tr><td class="numalign">50</td><td>B5</td><td class="numalign">989.9</td></tr>
<tr><td class="numalign">51</td><td>C6</td><td class="numalign">1045.4</td></tr>
<tr class="odd"><td class="numalign">52</td><td>C#6/D&#9837;6</td><td class="numalign">1107.5</td></tr>
<tr><td class="numalign">53</td><td>D6</td><td class="numalign">1177.5</td></tr>
<tr class="odd"><td class="numalign">54</td><td>D#6/E&#9837;6</td><td class="numalign">1242.9</td></tr>
<tr><td class="numalign">55</td><td>E6</td><td class="numalign">1316.0</td></tr>
<tr><td class="numalign">56</td><td>F6</td><td class="numalign">1398.3</td></tr>
<tr class="odd"><td class="numalign">57</td><td>F#6/G&#9837;6</td><td class="numalign">1471.9</td></tr>
<tr><td class="numalign">58</td><td>G6</td><td class="numalign">1575.5</td></tr>
<tr class="odd"><td class="numalign">59</td><td>G#6/A&#9837;6</td><td class="numalign">1669.6</td></tr>
<tr><td class="numalign">60</td><td>A6</td><td class="numalign">1747.8</td></tr>
<tr class="odd"><td class="numalign">61</td><td>A#6/B&#9837;6</td><td class="numalign">1864.3</td></tr>
<tr><td class="numalign">62</td><td>B6</td><td class="numalign">1962.5</td></tr>
<tr><td class="numalign">63</td><td>C7</td><td class="numalign">2110.6</td></tr>
</table><p>
The pitch table <code>ntscPeriods.s</code> is generated by
<code>pentlyas.py --periods 64</code>. The 64 can be changed to 76 to make
another octave above these notes (64 through 75) available, though that range
begins to fall out of tune due to the 2A03's limited period precision.
</p>

<h2 id="Sound_effects">Sound effects</h2>
<p>
At any moment, the mixer chooses to play either the music or the sound effect based on whatever is louder on each channel. If there is already a sound effect playing on the first square wave channel, another sound effect played at the same time will automatically be moved to the second, but a sound effect for the triangle or noise channel will not be moved. A sound effect will never interrupt another sound effect that has more frames remaining.
</p><p>
Sound effects are defined in <code>pently_sfx_table</code> in <code>musicseq.s</code>. Each is a <code>sfxdef</code> line giving a pointer to the sound effect's data, the length in steps, how much to slow it down, and which channel to play it on.
</p><pre>
sfxdef name, baseaddr, length, period, channel
</pre><dl>
<dt><code>name</code></dt>
<dd>The name of the sound effect, used for <code>pently_start_sound</code> and <code>drumdef</code>. This value is exported.</dd>
<dt><code>baseaddr</code></dt>
<dd>Starting address of sound effect data.</dd>
<dt><code>length</code></dt>
<dd>Length in steps of sound effect data.</dd>
<dt><code>period</code></dt>
<dd>Time in frames (1 to 16) to play each step of sound effect data.</dd>
<dt><code>channel</code></dt>
<dd>Which channel to play this sound effect on (0: pulse, 2: triangle, or 3: noise).</dd>
<p>
Sound effect data consists of a stream of two-byte steps, each consisting of a duty/volume and a pitch value. It may be played at one entry per frame or more slowly for longer sound effects. Volume is in the range <code>$01</code> through <code>$0F</code>, and for square wave channels, it can be OR'd with <code>$00</code> (1/8 duty, sharp), <code>$40</code> (1/4 duty, smooth), or <code>$80</code> (1/2 duty, hollow). For sound effects used on the triangle wave channel, always use <code>$80</code> to keep the note from stopping early due to interaction with the linear counter. The noise channel ignores duty, instead using the upper bit of pitch to determine the type of sound.
</p><p>
Pitch on the square and triangle channels is specified in semitone offsets from the lowest possible pitch (0, a low A). C is 3, 15, 27, 39, 51, or 63. Triangle waves are always played an octave below square waves; middle C is 27 on a square wave channel or 39 on a triangle wave channel. Pitch on a noise channel is <code>$03</code> (highest) to <code>$0F</code> (lowest) for ordinary noise or <code>$80</code> (highest) to <code>$8F</code> (lowest) for metallic tones. Values $00 through $02 are also valid, but they sound identical to quieter versions of $03.
</p>

<h2 id="Instruments">Instruments</h2>
<p>
There can be up to 51 different instrument definitions in a soundtrack.
</p><p>
The envelope determines the volume and timbre of an instrument over time. We take a cue from the <a href="https://en.wikipedia.org/wiki/Roland_D-50" target="wikipedia">Roland D-50 and D-550 synthesizers</a> that a note's attack is the hardest thing to synthesize. An instrument for the D-50 can play a PCM sample to sweeten the attack and leave the decay, sustain, and release to a subtractive synthesizer. Likewise in Pently, an envelope has two parts: attack and sustain.
</p><p>
An attack is like a short sound effect that specifies the duty, volume, and pitch for the first few frames of a note. It's analogous to the arpeggio, volume, and duty envelopes in FamiTracker, but in a compact format almost identical to that of sound effects with one difference: instead of specifying an absolute pitch (as in FamiTracker's "Fixed" envelope), they specify an offset in semitones from the note's own pitch (as in FamiTracker "Absolute" envelope).
</p><p>
After the attack finishes, the channel continues into the sustain. The duty and initial volume of the channel are set, and then the volume gradually decreases if desired. Each instrument is defined by one line in <code>pently_instruments</code> in <code>musicseq.s</code>:
</p><pre>
instdef name, duty, volume, decayrate, earlycut, attackptr, attacklen
</pre><dl>
<dt><code>name</code></dt>
<dd>The name of the instrument, used for <code>pently_start_sound</code> and <code>drumdef</code>.</dd>
<dt><code>duty</code></dt>
<dd>Width of pulse waves. Options are <code>0</code> for 12.5% (sharp); <code>1</code> for 25% (smooth), or <code>2</code> for 50% (hollow). Instruments for the triangle channel MUST use <code>2</code>.</dd>
<dt><code>volume</code></dt>
<dd>Starting volume of the sustain phase, from 0 to 15. Volume for the triangle channel is either off (0) or on (nonzero), but instrument volume is compared with sound effect volume.</dd>
<dt><code>decayrate</code></dt>
<dd>Rate of volume decrease in the sustain phase, in volume units per 16 frames. Optional; defaults to 0.</dd>
<dt><code>earlycut</code></dt>
<dd>If nonzero, the note shall be cut half a row before the next note. This allows leaving space between notes if there is no attack or decay, especially on triangle. Optional; defaults to 0.</dd>
<dt><code>attackptr</code></dt>
<dd>Pointer to attack data. Optional; used only if <code>attacklen</code> is larger than 0.</dd>
<dt><code>attacklen</code></dt>
<dd>Length in steps of attack data.</dd>
</dl><p>
Instruments for the noise channel are defined differently from instruments for the tone channels. A table <code>pently_drums</code> maps up to 25 note codes to pairs of sound effects. A common pattern is for a kick or snare drum to have a triangle component and a noise component, each represented as its own sound effect. Entries are specified as follows:
</p><pre>
drumdef name, sfx1, sfx2
</pre><dl>
<dt><code>name</code></dt>
<dd>The name of the drum, used in sound effects.</dd>
<dt><code>sfx1</code></dt>
<dd>An entry in <code>pently_sfx_table</code> to play when this drum is triggered.</dd>
<dt><code>sfx2</code></dt>
<dd>An optional second entry in <code>pently_sfx_table</code> to play when this drum is triggered.</dd>
</dl><p>
The fifth channel can <em>only</em> play attacks, and it plays them on top of the pulse 1, pulse 2, or triangle channel, replacing the attack phase of that channel's instrument (if any). This is useful for playing staccato notes on top of something else, interrupting the notes much like sound effects do.
</p>

<h2 id="Conductor_track">Conductor track</h2>
<p>
The conductor track determines which patterns are played when, how fast to play them, and how much of the song to repeat when reaching the end. This is the rough equivalent of an "order table" in a tracker. Each <code>songdef</code> line in the list of songs at <code>pently_songs</code> in <code>musicseq.s</code> names the song ID (to pass to <code>pently_start_music</code>) and points to a conductor track. 
</p><pre>
songdef name, conductor_addr
</pre><dl>
<dt><code>name</code></dt>
<dd>An identifier to pass to <code>pently_start_music</code>. Exported.</dd>
<dt><code>conductor_addr</code></dt>
<dd>The address of the start of this song's conductor data.</dd>
</dl><p>
Some examples of conductor patterns:
</p><dl>
<dt><code>setTempo 288</code></dt><dd>Sets the playback speed to 288 rows per minute. For example, this can represent 96 beats per minute where a beat is three rows, or 144 beats per minute where a beat is two rows. The speed defaults to 300 rows per minute and can be up to 2047 rows per minute, enough for thirty-second-note resolution at up to 255 quarter notes per minute. The player automatically adjusts the playback speed based on the value of the <code>tvSystem</code> variable (zero: 60.1 Hz, nonzero: 50 Hz).</dd>
<dt><code>playPatSq2 4, 27, FLUTE</code></dt><dd>Plays pattern 4 on the second square wave channel (<code>Sq2</code>), transposed up 15 semitones (base middle C), with instrument <code>FLUTE</code>.</dd>
<dt><code>playPatTri 5, 15, 0</code></dt><dd>Plays pattern 4 on the triangle wave channel (<code>Tri</code>), transposed up 15 semitones (base C3), with instrument <code>BASS</code>.</dd>
<dt><code>noteOnNoise $05, 4</code></dt><dd>Plays note $05 on the noise channel (<code>Noise</code>), with instrument 4. Conductor notes always use the instrument system, not the sound effect system, even on the noise channel. This might be useful for, say, a crash cymbal.</dd>
<dt><code>waitRows 48</code></dt><dd>Waits 48 rows before processing the next command. Use this to allow patterns to play through.</dd>
<dt id="fine"><code>fine</code></dt><dd>Stops music playback. Use this at the
end of a piece. (<em lang="it">Fine</em> is Italian for "end". In sheet music,
it directs the musician to stop playing in a piece of ternary (A-B-A) form.
More generally, sheet music uses a "final barline" symbol &#x1D102; to denote
where a piece stops.)
<dt><code>segno</code></dt><dd><em lang="it">Segno</em> (pronounced sen-yo) is Italian for "sign". In sheet music, it refers to <a href="https://en.wikipedia.org/wiki/Dal_Segno" target="wikipedia">the symbol &#x1D10B;</a> that marks the end of an introduction and the start of a large portion of a piece that should be repeated. This command has a similar function: setting the loop point in the conductor track.</dt>
<dt id="dalSegno"><code>dalSegno</code></dt><dd><em lang="it">Dal segno</em> (D.S.) is Italian for "from the sign". It directs the musician to go back to the loop point. This command moves the current position in the conductor track to the most recent <code>segno</code>. If no <code>segno</code> was seen, the position moves to the start of the piece; in music, this is called <a href="https://en.wikipedia.org/wiki/Da_capo" target="wikipedia"><em lang="it">da capo</em></a> (from the head).</code>
<dt><code>stopPatSq2</code></dt><dd>Stops the pattern playing on the second square wave channel. Patterns ordinarily loop when they reach the end, so you'll need to stop the pattern if you're not going to start another.</dd>
<dt><code>attackOnSq1</code></dt><dd>Plays the attack track on the first square wave channel.</dd>
<dt><code>setBeatDuration D_D8</code></dt><dd>Sets the duration of one beat to a dotted eighth note (three rows). The default is <code>D_4</code>, a quarter note (four rows). This has no audible effect, but <code>pently_row_callback</code> can see <code>rowBeatPart</code> and <code>rowsPerBeat</code> as a convenience to synchronize animations or DPCM samples to the music.</dd>
</dl><p>
The transpose values are in semitones. Pitch values such that the value <code>N_C</code> in pattern code produces a C are 3, 15, 27, and 39. For example, with transpose 15 on a square wave channel or 27 on a triangle wave channel, <code>N_CH</code> produces a middle C and <code>N_C</code> produces the C an octave below it. Other values produce transpositions that can prove useful for fitting a melody into the two-octave range of a single pattern. The noise channel ignores both transpose and instrument.
</p><p>
The list of all conductor commands defined in <code>pentlyseq.inc</code> follows; the meaning should ideally be self-explanatory given the above descriptions.
</p><ul>
<li>Play pattern: <code>playPatSq1, playPatSq2, playPatTri, playPatNoise, playPatAttack</code></li>
<li>Stop pattern: <code>stopPatSq1, stopPatSq2, stopPatTri, stopPatNoise, stopPatAttack</code></li>
<li>Play note on pattern: <code>noteOnSq1, noteOnSq2, noteOnTri, noteOnNoise</code></li>
<li>Set channel for attack track: <code>attackOnSq1, attackOnSq2, attackOnTri</code></li>
<li>Loop control: <code>fine, segno, dalSegno</code></li>
<li>Timing control: <code>setTempo, setBeatDuration, waitRows</code></li>
</ul>

<h2 id="Patterns">Patterns</h2>
<p>
A pattern represents a musical phrase as a sequence of notes with durations. Unlike in traditional trackers, patterns can be any length, with a shorter pattern on one track looping while a longer pattern on another track plays. Patterns are listed below <code>pently_patterns</code> in <code>musicseq.s</code>:
</p></p><pre>
patdef name, patdata_addr
</pre><dl>
<dt><code>name</code></dt>
<dd>An identifier to pass to <code>playPat</code> commands.</dd>
<dt><code>patdata_addr</code></dt>
<dd>The address of the start of this pattern's data.</dd>
</dl><p>
Each note's pitch is relative to the transposition base in the <code>playPat</code> command in the conductor track:
</p><table class="datatable"><caption>Note values in pattern data</caption>
<tr><th>Code</th><th>Note if base is C</th><th>Name of interval</th><th>Interval in semitones</th></tr>
<tr><td><code>N_C</code></td><td>C</td><td>Unison</td><td>0</td></tr>
<tr><td><code>N_CS</code> or <code>N_DB</code></td><td>C#/D&#9837;</td><td>Minor second</td><td>1</td></tr>
<tr><td><code>N_D</code></td><td>D</td><td>Major second</td><td>2</td></tr>
<tr><td><code>N_DS</code> or <code>N_EB</code></td><td>D#/E&#9837;</td><td>Minor third</td><td>3</td></tr>
<tr><td><code>N_E</code></td><td>E</td><td>Major third</td><td>4</td></tr>
<tr><td><code>N_F</code></td><td>F</td><td>Perfect fourth</td><td>5</td></tr>
<tr><td><code>N_FS</code> or <code>N_GB</code></td><td>F#/G&#9837;</td><td>Tritone</td><td>6</td></tr>
<tr><td><code>N_G</code></td><td>G</td><td>Perfect fifth</td><td>7</td></tr>
<tr><td><code>N_GS</code> or <code>N_AB</code></td><td>G#/A&#9837;</td><td>Minor sixth</td><td>8</td></tr>
<tr><td><code>N_A</code></td><td>A</td><td>Major sixth</td><td>9</td></tr>
<tr><td><code>N_AS</code> or <code>N_BB</code></td><td>A#/B&#9837;</td><td>Minor seventh</td><td>10</td></tr>
<tr><td><code>N_B</code></td><td>B</td><td>Major seventh</td><td>11</td></tr>
<tr><td><code>N_CH</code></td><td>High C</td><td>Octave</td><td>12</td></tr>
<tr><td><code>N_CSH</code> or <code>N_DBH</code></td><td>High C#/D&#9837;</td><td></td><td>13</td></tr>
<tr><td><code>N_DH</code></td><td>High D</td><td></td><td>14</td></tr>
<tr><td><code>N_DSH</code> or <code>N_EBH</code></td><td>High D#/E&#9837;</td><td></td><td>15</td></tr>
<tr><td><code>N_EH</code></td><td>High E</td><td></td><td>16</td></tr>
<tr><td><code>N_FH</code></td><td>High F</td><td></td><td>17</td></tr>
<tr><td><code>N_FSH</code> or <code>N_GBH</code></td><td>High F#/G&#9837;</td><td></td><td>18</td></tr>
<tr><td><code>N_GH</code></td><td>High G</td><td></td><td>19</td></tr>
<tr><td><code>N_GSH</code> or <code>N_ABH</code></td><td>High G#/A&#9837;</td><td></td><td>20</td></tr>
<tr><td><code>N_AH</code></td><td>High A</td><td></td><td>21</td></tr>
<tr><td><code>N_ASH</code> or <code>N_BBH</code></td><td>High A#/B&#9837;</td><td></td><td>22</td></tr>
<tr><td><code>N_BH</code></td><td>High B</td><td></td><td>23</td></tr>
<tr><td><code>N_CHH</code></td><td>Top C</td><td>Two octaves</td><td>24</td></tr>
</table><p>
Only one note can be played on a single track at once; playing a note cuts the one already playing. To stop a note without playing another, use a <code>REST</code>.
</p><p>
Each note or rest is OR'd with a duration, or the number of rows to wait after the note is played. The durations are in fractions of a 16-row "<a href="https://en.wikipedia.org/wiki/Whole_note" target="wikipedia">whole note</a>", following standard practice for describing durations in U.S. and Canadian English, most other Germanic languages, Chinese, and Greek. Available durations are &#x1D161; sixteenth (default, 1 row), &#x1D160; eighth (<code>|D_8</code>, 2 rows), &#x1D15F; quarter (<code>|D_4</code>, 4 rows), &#x1D15E; half (<code>|D_2</code>, 8 rows), and &#x1D15D; whole (<code>|D_1</code>, 2 rows). Augmented (or "dotted") versions of eighth, quarter, and half notes are 50 percent longer: &#x1D160;&#x1D16D; dotted eighth (<code>|D_D8</code>, 3 rows), &#x1D15F;&#x1D16D; dotted quarter (<code>|D_D4</code>, 6 rows), and &#x1D15E;&#x1D16D; dotted half (<code>|D_D2</code>, 12 rows). Not all durations can be expressed with one row, but anything up to 20 rows can be made from two <a href="https://en.wikipedia.org/wiki/Tie_(music)" target="wikipedia">tied</a> notes: a note with <code>D_4</code>, <code>D_2</code>, or <code>D_D2</code> followed by <code>N_TIE</code>, <code>N_TIE|D_8</code>, or <code>N_TIE|D_D8</code>.
</p><table class="datatable">
<caption>Note G played with each of 16 durations</caption>
<tr><th>Code</th><th>Duration name</th><th>Length in rows</th></tr>
<tr><td><code>N_G</code></td><td>Sixteenth</td><td class="numalign">1</td></tr>
<tr><td><code>N_G|D_8</code></td><td>Eighth</td><td class="numalign">2</td></tr>
<tr><td><code>N_G|D_D8</code></td><td>Dotted eighth</td><td class="numalign">3</td></tr>
<tr><td><code>N_G|D_4</code></td><td>Quarter</td><td class="numalign">4</td></tr>
<tr><td><code>N_G|D_4, N_TIE</code></td><td>Quarter + sixteenth</td><td class="numalign">5</td></tr>
<tr><td><code>N_G|D_D4</code></td><td>Dotted quarter</td><td class="numalign">6</td></tr>
<tr><td><code>N_G|D_4, N_TIE|D_D8</code></td><td>Quarter + dotted eighth</td><td class="numalign">7</td></tr>
<tr><td><code>N_G|D_2</code></td><td>Half</td><td class="numalign">8</td></tr>
<tr><td><code>N_G|D_2, N_TIE</code></td><td>Half + sixteenth</td><td class="numalign">9</td></tr>
<tr><td><code>N_G|D_2, N_TIE|D_8</code></td><td>Half + eighth</td><td class="numalign">10</td></tr>
<tr><td><code>N_G|D_2, N_TIE|D_D8</code></td><td>Half + dotted eighth</td><td class="numalign">11</td></tr>
<tr><td><code>N_G|D_D2</code></td><td>Dotted half</td><td class="numalign">12</td></tr>
<tr><td><code>N_G|D_D2, N_TIE</code></td><td>Dotted half + sixteenth</td><td class="numalign">13</td></tr>
<tr><td><code>N_G|D_D2, N_TIE|D_8</code></td><td>Dotted half + eighth</td><td class="numalign">14</td></tr>
<tr><td><code>N_G|D_D2, N_TIE|D_D8</code></td><td>Dotted half + dotted eighth</td><td class="numalign">15</td></tr>
<tr><td><code>N_G|D_1</code></td><td>Whole</td><td class="numalign">16</td></tr>
</table><p>
A pattern can force a particular instrument to be used, such as when a pattern alternates between instruments. For this, use <code>INSTRUMENT</code> followed by the instrument's name.
</p><p>
Legato, also called slur or <a href="https://en.wikipedia.org/wiki/Hammer-on" target="wikipedia">HOPO</a>, is an effect that skips the ordinary note-on process. A note played legato is played by changing the pitch of the existing note on a channel without restarting its envelope, and instruments set to note-off a half row early will not do so when legato is on. To slur a set of notes, put <code>LEGATO_ON</code> after the first and <code>LEGATO_OFF</code> after the last. Legato doesn't make sense in the attack track, which is played staccato by definition.
</p><p>
Arpeggio rapidly cycles a note among two or three different pitches, which produces the warbly chords heard in <a href="https://en.wikipedia.org/wiki/MOS_Technology_SID#Software_emulation" target="wikipedia">SIDs</a> and <a href="https://en.wikipedia.org/wiki/NES_Sound_Format" target="wikipedia">NSFs</a> by European composers. The arpeggio is specified as a hexadecimal number, similar to that used with the <code>J47</code> effect in <a href="https://en.wikipedia.org/wiki/S3M_(file_format)" target="wikipedia">S3M</a> or <a href="https://en.wikipedia.org/wiki/Impulse_Tracker" target="wikipedia">IT</a> or the <code>047</code> effect in <a href="https://en.wikipedia.org/wiki/MOD_(file_format)" target="wikipedia">MOD</a>, <a href="https://en.wikipedia.org/wiki/FastTracker_2" target="wikipedia">XM</a>, or FTM. with a first and second nibble representing intervals in semitones. If the second nibble is 0, only two steps are used; otherwise, three steps are used. For example, <code>ARPEGGIO,$47</code> makes a major chord in root position including 4 semitones (a major third) and 7 semitones (a perfect fifth) above the root note. There are three ways to make an interval, depending on how much the lower or higher note should dominate. For example, with an octave <code>ARPEGGIO,$0C</code> is three steps low, low, and high; <code>ARPEGGIO,$C0</code> is two steps low and high; and <code>ARPEGGIO,$CC</code> is three steps low, high, and high. Arpeggio doesn't work in the attack track, and an arpeggio involving both a base note below middle C and an interval below an octave tends to sound muddy.
</p><table class="datatable">
<caption>Some possible arpeggio values</code>
<tr><th>Code</th><th>Result</th></tr>
<tr><td><code>ARPEGGIO,$00</code></td><td>Turn off arpeggio</td></tr>
<tr><td><code>ARPEGGIO,$30</code></td><td>Minor third</td></tr>
<tr><td><code>ARPEGGIO,$40</code></td><td>Major third</td></tr>
<tr><td><code>ARPEGGIO,$50</code></td><td>Perfect fourth</td></tr>
<tr><td><code>ARPEGGIO,$60</code></td><td>Tritone</td></tr>
<tr><td><code>ARPEGGIO,$70</code></td><td>Perfect fifth</td></tr>
<tr><td><code>ARPEGGIO,$C0</code></td><td>Octave</td></tr>
<tr><td><code>ARPEGGIO,$37</code></td><td>Minor chord, root</td></tr>
<tr><td><code>ARPEGGIO,$38</code></td><td>Major chord, first inversion</td></tr>
<tr><td><code>ARPEGGIO,$47</code></td><td>Major chord, root</td></tr>
<tr><td><code>ARPEGGIO,$49</code></td><td>Minor chord, first inversion</td></tr>
<tr><td><code>ARPEGGIO,$57</code></td><td>Sus4 chord</td></tr>
<tr><td><code>ARPEGGIO,$58</code></td><td>Minor chord, second inversion</td></tr>
<tr><td><code>ARPEGGIO,$59</code></td><td>Major chord, second inversion</td></tr>
</table><p>
Vibrato is a subtle pitch slide up and down while a note is held. First it waits 12 frames, then it adds a fraction of the note's period to its period, which fraction is controlled by a sinusoid with period 12 frames. The depth can be set from off (<code>VIBRATO,0</code>) to subtle (<code>VIBRATO,1</code>) through very strong (<code>VIBRATO,4</code>).
</p><p>
The transpose command changes the pitch of the rest of a pattern by a given number of semitones. For example, <code>TRANSPOSE,5</code> moves the rest of the pattern up a perfect fourth. <code>TRANSPOSE,&lt;-12</code> moves down an octave, with the <code>-</code> denoting negative and the <code>&lt;</code> working around ca65's lack of support for signed bytes. It's most often used to include notes more than two octaves apart in one pattern.
</p><p>
The grace command shortens the next two rows to one row's length. The next byte specifies the length in frames of the first note in the pair. Like the <code>EDx</code> command in MOD/XM or the <code>SDx</code> command in S3M/IT, it's designed for making an <a href="https://en.wikipedia.org/wiki/Ornament_%28music%29#Acciaccatura">acciaccatura (grace note)</a> or a set of triplets (3 notes in the time of 4). For example, to play a short C note for 4 frames followed by a B flat that is as long as a quarter note minus 4 frames, do <code>GRACE,4,N_CH, N_BB|D_Q4.</code>
</p><p>
Finally, to end the pattern, use <code>PATEND</code>. This isn't strictly necessary if a pattern is always interrupted at its end, but if it isn't present, playback will <a href="https://en.wikipedia.org/wiki/Switch_statement#Fallthrough">fall through</a> into the following pattern.
</p><p>
The following are all the symbols that are valid in pattern code:
</p><ul>
<li>Notes, low octave: <code>N_C, N_CS, N_D, N_DS, N_E, N_F, N_FS, N_G, N_GS, N_A, N_AS, N_B</code></li>
<li>Notes, high octave: <code>N_CH, N_CSH, N_DH, N_DSH, N_EH, N_FH, N_FSH, N_GH, N_GSH, N_AH, N_ASH, N_BH</code></li>
<li>Note, top of range: <code>N_CHH</code></li>
<li>Notes, <a href="https://en.wikipedia.org/wiki/Enharmonic" target="wikipedia">enharmonic</a> synonyms: <code>N_DB, N_EB, N_GB, N_AB, N_BB, N_DBH, N_EBH, N_GBH, N_ABH, N_BBH</code>
<li>Duration carriers that are not notes: <code>N_TIE, REST</code></li>
<li>Durations: <code>D_8</code> (2 rows), <code>D_D8</code> (3 rows), <code>D_4</code> (4 rows), <code>D_D4</code> (6 rows), <code>D_2</code> (8 rows), <code>D_D2</code> (12 rows), <code>D_1</code> (16 rows)</code></li>
<li>Control: <code>INSTRUMENT, ARPEGGIO, LEGATO_ON, LEGATO_OFF, VIBRATO, TRANSPOSE, PATEND</code></li>
</ul>

<h2 id="Bugs_and_limits">Bugs and limits</h2>
<p>
No music engine is perfect. These problems exist:
</p><ul>
<li>Though it's only 1.6 KiB and thus much smaller than the FamiTracker or
NerdTracker II player, it may still take up too much space in a very tight
NROM-128 game because it is not modularized to be built without support for
some effects.</li>
<li>There is currently no way to split sequence data across multiple PRG ROM banks or stash it in CHR ROM (like in <em>Galaxian</em>).</li>
<li>No pitch bends.</li>
<li>No echo buffer effect.</li>
<li>No support for DPCM drums. This is a low priority because Pently is used in games that depend on controllers or raster effects incompatible with DPCM. However, it won't interfere with your own sample player, which can be triggered from <code>pently_row_callback</code>.</li>
<li>No support for Famicom expansion synths, such as Nintendo MMC5, Sunsoft
5B, and Konami VRC6 and VRC7. This is a low priority for two reasons: the NES
sold in English-speaking regions did not support expansion synths without
modification, and no expansion synth has a CPLD replica as of 2016.</li>
<li>Envelopes have no release phase; a note-off kills the note abruptly.</li>
<li>No error checking for certain combinations that cause undefined behavior.</li>
<li>No graphical editor, unless you count using FamiTracker and then converting it with <a href="https://github.com/NovaSquirrel/ft2pently">NovaSquirrel's ft2pently</a>.</li>
<li>Limit of 51 instruments, 64 sound effects, 25 different drums, 128 patterns, and 128 songs.</li>
<li>The bottom octave of the 88-key piano is missing from the pulse channel
and the top octave from the triangle channel, reflecting an NES limit.</li>
<li>The row grid cannot be swung.</li>
<li>Pently does not compose music for you, but you could write your own <a href="https://en.wikipedia.org/wiki/Algorithmic_composition" target="wikipedia">algorithmic composition</a> engine in terms of <a href="#API"><code>pently_play_note</code></a>.</li>
</ul>

<h2 id="License">License</h2>
<p>
The Pently audio engine is distributed under the MIT License (Expat
variant):
</p><pre>
Copyright 2010-2016 Damian Yerrick

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
</pre><p>
This means that yes, you may use Pently in games that you are selling on cartridge. And no, you do not have to make your game <a href="https://en.wikipedia.org/wiki/Free_software" target="wikipedia">free software</a>; this is not a copyleft. If a game is distributed with a manual, you may place the full notice in the manual so long as the author is credited within the game.
</p>
</body></html>