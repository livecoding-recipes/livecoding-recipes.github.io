---
layout: post
title:  "Implementing a MIDI player in Kotlin from scratch"
date:   2022-08-01
categories: midi kotlin tracker
---

In this series I’ll try to show you how to implement a tracker-like environment in pure Kotlin. The goal is to divide this into 3 parts:
1. Getting familiar with MIDI protocol and its abstractions in the JVM library to implement a simple MIDI player using coroutines (this post)
2. Introduce Open Sound Control (OSC), it's advantages over MIDI and use SuperCollider for precise timing, synth design and sample playback
3. Discover interactivity possibilities with Kotlin Scripting.

### Built-in Java MIDI support
Recently I discovered that the standard JVM library contains a feature-rich implementation of the MIDI protocol. We can grab any MIDI file from the web, and play it using the JVM with the following piece of code:
 
{% highlight kotlin %}
import javax.sound.midi.MidiSystem

class JvmMidiSequencer

fun main() {
    val midiStream = JvmMidiSequencer::class.java.getResourceAsStream("/GiorgiobyMoroder.mid")
    val sequencer = MidiSystem.getSequencer().apply { setSequence(midiStream) }
    sequencer.open()
    sequencer.start()
}
{% endhighlight %}

You should hear something like this:
<audio controls>
  <source src="/assets/sounds/giorgio/giorgio_jvm.ogg" type="audio/ogg"/>
  <source src="/assets/sounds/giorgio/giorgio_jvm.mp3" type="audio/mpeg"/>
</audio>

The goal of this part is to recreate this with pure Kotlin code using coroutines.

### Reverse engineering the MIDI events
First of all, let's try to reverse engineer the contents of a MIDI file. Starting with `sequencer.sequence` we can easily discover that:
- a MIDI track contains a `Sequence` of `Track`s
- each `Track` contains a sequence of `MidiEvent`s 
- each `MidiEvent` has a `tick` (a timestamp of the event) and a `MidiMessage`

So let's print it out:

{% highlight kotlin %}
sequence.tracks.forEachIndexed { index, track ->
    println("Track $index")
    (0 until track.size()).asSequence().map { idx ->
        val event = track[idx]
        println("Tick ${event.tick}, message: ${event.message}")
    }.take(10).toList()
{% endhighlight %}

{% highlight console %}
Track 0
Tick 0, message: javax.sound.midi.MetaMessage@22ff4249
Tick 0, message: javax.sound.midi.MetaMessage@5b12b668
Tick 0, message: com.sun.media.sound.FastShortMessage@1165b38
Tick 0, message: javax.sound.midi.MetaMessage@4c12331b
Tick 0, message: javax.sound.midi.MetaMessage@7586beff
Tick 0, message: com.sun.media.sound.FastShortMessage@3b69e7d1
Tick 240, message: com.sun.media.sound.FastShortMessage@815b41f
Tick 240, message: com.sun.media.sound.FastShortMessage@5542c4ed
Tick 480, message: com.sun.media.sound.FastShortMessage@1573f9fc
Tick 480, message: com.sun.media.sound.FastShortMessage@6150c3ec
{% endhighlight %}

Ok, so apart from discovering that `MidiMessage` doesn't have proper `toString` implementation, we can see something that is specified in `MidiMessage` [javadocs](https://docs.oracle.com/en/java/javase/17/docs/api/java.desktop/javax/sound/midi/MidiMessage.html) - that the events 
> include not only the standard MIDI messages that a synthesizer can respond to, but also `meta-events` that can be used by sequencer programs

These must be `MetaMessage`s, so for now let's focus on the standard MIDI messages - `ShortMessage`s:

{% highlight kotlin %}
    ...
    when (val message = event.message) {
        is ShortMessage -> {
            val command = message.command
            val data1 = message.data1
            val data2 = message.data2
            println("Tick ${event.tick}, command: $command, data1: $data1, data2: $data2")
        }
    }
    ...
{% endhighlight %}

{% highlight console %}
Track 0
Tick 0, command: 192, data1: 38, data2: 0
Tick 0, command: 144, data1: 57, data2: 64
Tick 240, command: 128, data1: 57, data2: 0
Tick 240, command: 144, data1: 45, data2: 64
Tick 480, command: 128, data1: 45, data2: 0
Tick 480, command: 144, data1: 60, data2: 64
Tick 720, command: 128, data1: 60, data2: 0
Tick 720, command: 144, data1: 45, data2: 64
{% endhighlight %}

OK, that's better! We can now see that each event has a `commmand` and two `data` fields. This could be a good time to look at MIDI specification - a nice brief is in [this article](https://computermusicresource.com/MIDI.Commands.html). From this table we can discover that for playing notes the `NOTE ON` and `NOTE OFF` events are used, and their `data1` is the key number (MIDI note) and `data2` is the velocity of the sound:
![midi_commands](/assets/images/midi_commands.png)

Digging into `ShortMessage` class, we can also find command codes for both `NOTE ON` and `NOTE OFF` messages:
{% highlight java %}
public class ShortMessage extends MidiMessage {
    ...
    public static final int NOTE_OFF = 128;
    public static final int NOTE_ON = 144;
    ...
}
{% endhighlight %}

Given this knowledge we can now interpet these events:
{% highlight console %}
Tick 0, command: 144, data1: 57, data2: 64
Tick 240, command: 128, data1: 57, data2: 0
Tick 240, command: 144, data1: 45, data2: 64
Tick 480, command: 128, data1: 45, data2: 0
{% endhighlight %}
as:
- at tick 0 playing the key with note 57 (`A`) with velocity 64
- at tick 240 releasing the key with note 57
- at tick 240 playing the key with note 45 (lower `A`) with velocity 64
- at tick 480 releasing the key with note 45

### Modeling the melody
OK, at this time we know how MIDI files are constructed, so it's good time to think about our own representation of melody. I think it will be good idea to "resolve" two issues with MIDI events:
1. Each `NOTE ON` event must be "terminated" with corresponding `NOTE OFF` event; this could cause problems  when the `NOTE OFF` event is missing, a better idea would be to just have a note duration, just as in sheet music notation
2. Dealing with `tick`s might be good for machines, but it would be more readable if we just represent time of the notes using `beat`s and calculate the song tempo with beats per minute (BPM).

Using these assumptions we can introduce the `Note` class as:
{% highlight kotlin %}
data class Note(
    val beat: Double,     // e.g. 0.0, 0.25, 1.5
    val midinote: Int,    // e.g. 60, 57
    val duration: Double, // in beats: 0.25, 0.5
    val amp: Float = 1.0f // 0.0f - 1.0f
)
{% endhighlight %}

To translate `tick`s to `beat`s we can just use the sequence `resolution` field, since most of the MIDI files all modeled using the [`PPQ` division type:](https://docs.oracle.com/en/java/javase/11/docs/api/java.desktop/javax/sound/midi/Sequence.html#PPQ) 
{% highlight java %}
public static final float PPQ;
 // The tempo-based timing type, for which the resolution is expressed in pulses (ticks) per quarter note.
{% endhighlight %}

So, to translate MIDI events to `List` of our `Note` objects, we can write this extension function: 
{% highlight kotlin %}
fun javax.sound.midi.Sequence.toNotes(): List<Note> {
    return tracks.flatMap { track ->
        (0 until track.size()).asSequence().map { idx ->
            val event = track[idx]
            when (val message = event.message) {
                is ShortMessage -> {
                    val command = message.command
                    val midinote = message.data1
                    val amp = message.data2
                    if (command == ShortMessage.NOTE_ON) {
                        val beat = (event.tick / resolution.toDouble())
                            .toBigDecimal().setScale(2, RoundingMode.HALF_UP)
                            .toDouble()
                        return@map Note(beat, midinote, 0.25, amp / 127.0f)
                    }
                }
            }
            null
        }.filterNotNull()
    }
}
{% endhighlight %}
To simplify, I just set each `Note` duration constantly to `0.25` instead of calculating it by finding the corresponding `NOTE OFF` event.

### Implementing the Player
We are now ready for a final part of implementation - a `Player`. The most important thing for now it the timing - so let's introduce helper `Metronome` class to correctly transpose `BPM` to milliseconds:

{% highlight kotlin %}
data class Metronome(var bpm: Int) {
    val millisPerBeat: Long
        get() = (secsPerBeat * 1000).toLong()

    private val secsPerBeat: Double
        get() = 60.0 / bpm
}
{% endhighlight %}
So for example, for a common `120 BPM` we should have 0.5 seconds per beat.

To implement a `Player` we'll use `coroutines` which allow us to write really simple code by just using the `delay` function to wait until the timestamp of the next note to play. This is really neat, as opposed to traditional multithreaded code, when you don't want to block the running thread with the `Thread.sleep` calls.

{% highlight kotlin %}
abstract class Player(
    private val notes: List<Note>, protected val metronome: Metronome,
    private val scope: CoroutineScope = CoroutineScope(Dispatchers.Default)
) {

    fun schedule(time: LocalDateTime, function: () -> Unit) {
        scope.launch {
            delay(Duration.between(LocalDateTime.now(), time).toMillis())
            function.invoke()
        }
    }
}
{% endhighlight %}
Then, to play all the notes, we just need to `schedule` until their `beat` translated to millis from some starting time:

{% highlight kotlin %}
    ... 
    private var playing = true

    abstract fun playNote(note: Note, playAt: LocalDateTime)

    fun playNotes(at: LocalDateTime) {
        notes.forEach { note ->
            val playAt = at.plus((note.beat * metronome.millisPerBeat).toLong(), ChronoUnit.MILLIS)
            schedule(playAt) {
                if (playing) {
                    playNote(note, playAt)
                }
            }
        }
    }
    ...
{% endhighlight %}
To play MIDI notes, we need to pass `javax.sound.midi.Receiver` instance, which allows us to send `MidiMessage`s. We send `NOTE ON` immidiately, and schedule `NOTE OFF` to play after note's `duration`:

{% highlight kotlin %}
class MidiPlayer(private val receiver: Receiver, notes: List<Note>, metronome: Metronome, scope: CoroutineScope) : Player(notes, metronome, scope) {

    override fun playNote(note: Note, playAt: LocalDateTime) {
        val midinote = note.midinote
        val midiVel = (127f * note.amp).toInt()
        val noteOnMsg = ShortMessage(ShortMessage.NOTE_ON, 0, midinote, midiVel)
        receiver.send(noteOnMsg, -1)
        val noteOffAt = playAt.plus((note.duration * metronome.millisPerBeat).toLong(), ChronoUnit.MILLIS)
        schedule(noteOffAt) {
            val noteOffMsg = ShortMessage(ShortMessage.NOTE_OFF, 0, midinote, midiVel)
            receiver.send(noteOffMsg, -1)
        }
    }
}
{% endhighlight %}

Summing it up, the code to play first 16 beats of `Giorgio by Moroder` looks like this:

{% highlight kotlin %}
class SimpleMidiSequencer

fun main() {
    val midiStream = SimpleMidiSequencer::class.java.getResourceAsStream("/GiorgiobyMoroder.mid")
    val sequencer = MidiSystem.getSequencer().apply { setSequence(midiStream) }
    val synthesiser = MidiSystem.getSynthesizer().apply { open() }

    val melody = sequencer.sequence.toNotes().takeWhile { it.beat < 16 }

    runBlocking {
        val metronome = Metronome(bpm = 110)
        val player = MidiPlayer(synthesiser.receiver, melody, metronome, this)
        player.playNotes(LocalDateTime.now())
    }
}
{% endhighlight %}

We can also now very simply convert it into looper, by just making these 16 notes a `bar` and playing it one after another. Do make it possible let's pass the loop lenght to the `Metronome`:

{% highlight kotlin %}
data class Metronome(val bpm: Int, val beatsPerBar: Int = 16) {
    ...
    val millisPerBar: Long
        get() = beatsPerBar * millisPerBeat
    ...
}
{% endhighlight %}

And then let's add `playBar` function to our generic `Player`:
{% highlight kotlin %}
    fun playBar(bar: Int, at: LocalDateTime) {
        if (!playing) return
        println("Playing bar $bar")
        playNotes(at)
        val nextBarAt = at.plus(metronome.millisPerBar, ChronoUnit.MILLIS)
        schedule(nextBarAt) {
            playBar(bar + 1, nextBarAt)
        }
    }
{% endhighlight %}

Then we just need to start the looper by playing the first bar:

{% highlight kotlin %}
    ...
    runBlocking {
        val metronome = Metronome(bpm = 110)
        val player = MidiPlayer(synthesiser.receiver, melody, metronome, this)
        player.playBar(1, LocalDateTime.now())
    ...
{% endhighlight %}

Now we're ready to play the final result, with additional feature of adjusting the tempo. Here is an example with 90 BPM:
<audio controls>
  <source src="/assets/sounds/giorgio/giorgio_player_90bpm.ogg" type="audio/ogg"/>
  <source src="/assets/sounds/giorgio/giorgio_player_90bpm.mp3" type="audio/mpeg"/>
</audio>

### Connecting to a real synthesiser
Finally, using a MIDI as interface gives us an opportunity to connect to various software and hardware devices. If you don't have a hardware synth you can use for example open-source [Surge XT](https://surge-synthesizer.github.io/) which sounds pretty well.

ℹ️ _For linux users: you should install Virtual MIDI kernel driver to trigger software synth events, see [this link](
https://github.com/anton-k/linux-audio-howto/blob/master/doc/os-setup/virtual-midi.md#virtual-midi-1) for detailed instructions_

To connect to given MIDI device, we have to filter out the `MidiSystem.getMidiDeviceInfo` list, in this example I'm looking for `VirMIDI`, which was created by Linux kernel driver:

{% highlight kotlin %}
    val midiDeviceInfos = MidiSystem.getMidiDeviceInfo()

    val device = midiDeviceInfos.toList()
        .map { MidiSystem.getMidiDevice(it) }
        .first { it.deviceInfo.description.startsWith("VirMIDI") }

    device.open()

    val receiver = device.receiver

{% endhighlight %}
Then we just need to pass this `receiver` to `Player` instead of `synstesiser.receiver`:

{% highlight kotlin %}
    ...
    val player = MidiPlayer(receiver, melody, metronome, this)
    ...
{% endhighlight %}

Here is a sample session with **SurgeXT**:
<iframe width="560" height="315" src="https://www.youtube.com/embed/6ov4TkH0_cQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

 It's Kotlin playing a real synth, enjoy! 😍