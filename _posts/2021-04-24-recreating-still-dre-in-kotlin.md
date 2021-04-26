---
layout: post
title:  "Recreating \"Still D.R.E.\" in Kotlin"
date:   2021-04-24 19:00:00
categories: live-coding remake
---

In this post I'll show how to recreate a classic hip-hop tune by Dr Dre using my library [punkt](https://github.com/pjagielski/punkt) and some Kotlin goodness for sequence processing. I was heavily using [this awesome post](https://coenmodder.com/still-dre-how-a-simple-pattern-turned-two-chords-into-a-classic/) to find correct chords and rhythm.

Here is short session with final result:
<iframe width="560" height="315" src="https://www.youtube.com/embed/Ch8e4vrcndw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Chords

As shown in the [Coen's video](https://www.youtube.com/watch?v=xy9ADw8NMwo&ab_channel=CoenModder-PianoCouture) the song is basically built on two chords: `Am` and `Em` (with `sus4` variant). In punkt to create this progression we must specify the key/scale (`A minor`), then select the chords using roman numerals (`I` and `V`) and then specify chord variants (sus4/inversions). So I ended up with this piece of code:

{% highlight kotlin %}
val scale = Scale(A, minor)

val (Am, Emsus4, Em) = listOf(
    Chord.I.inversion(1), // -> Am
    Chord.V.sus4().inversion(2).low(), // -> Emsus4
    Chord.V.inversion(2).low() // -> Em
)
{% endhighlight %}

When we have a `scale` and a chord progression we can create a piece of music by calling `phrase` function. It is an extension function on the `scale`, takes a list of `chords` and list of durations:

{% highlight kotlin %}
+ scale.high()
    .phrase(
        chords(
            listOf((Am to 8), (Emsus4 to 3), (Em to 5))
                .flatMap { (ch, i) -> (1..i).map { ch } }
        ),
        repeat(0.5) // durations
    )

{% endhighlight %}

In `punkt` the durations are expressed in beats (`1.0` is one beat), so we repeat `0.5` (8th notes) to create the rhythm. To repeat the chords and create the melody, we can `flatMap` over the pairs of `chord` and the repetition counter and as a result we get 8 of `Am`, 5 of `Emsus4` and 3 of `Em`, which fills the whole loop.

One more thing is this `flam`/`crushed notes` technique - the chord in not played exactly at given time but notes are slightly delayed one after another. To achieve this we can use `every` function from `punkt` - it modifies every nth note of the sequence starting from given index:

{% highlight kotlin %}
.every(3, { step -> step.copy(beat = step.beat + 0.04) }, from = 1)
.every(3, { step -> step.copy(beat = step.beat + 0.09) }, from = 2)
{% endhighlight %}

So every 2nd (index 1) note of the chord is delayed by `0.04` and every 3rd (index 2) by `0.09`.

### Bass
For the bass (or piano left hand) there is simple melody of `A`, `B` and `E`, and the pattern starts with pickup so it's like `-> A | B -> E | E ->`:

{% highlight kotlin %}
val (a, b, e) = listOf(0, 1, -3)
fun Scale.bass() =
    this.low().low().phrase(
        // -> A, B -> E, E ->
        degrees(listOf(a, null, b, null, e, null, e)),
        cycle(1.5, 1.5, 0.75, 0.25) // cycle through durations
    )
{% endhighlight %}

To pass single notes to `phrase` we use `degrees` instead of `chords`. To create a pause we use `null` and the `phrase` will just fill the pattern with empty note.

We can define this bass pattern as an extension function to play it by both `piano` and `tr808` synth to add more on the low-end:

{% highlight kotlin %}
+ scale
    .bass()
    .synth("piano")
    .amp(0.65)

+ scale
    .bass()
    .synth("tr808")
    .amp(1.25)
    .track(1)
    .djf(0.4)
{% endhighlight %}

### Beat
For the beat we can use some short samples with bass drum, snare and open hihat:
{% highlight kotlin %}
+ cycle(1.25, 0.75, 2.0).sample("bd_haus")
    .amp(1.25)

+ repeat(2.0).sample("sd", at = 1.0)
    .amp(0.85)

+ repeat(2.0).sample("snare_3", at = 1.0)
    .amp(0.3)
    .chop(config, 2)

+ repeat(2.0).sample("hho", at = 0.5)
    .amp(1.25)
{% endhighlight %}

The bass drum plays on `0.0`, `1.25`, `2.0` and `4.0` beats. The snare is layered on two samples playing every even beat starting at `1.0`.

You can find all the code of this post in [punkt-template repository](https://github.com/pjagielski/punkt-template/blob/master/src/main/kotlin/still.kts).
