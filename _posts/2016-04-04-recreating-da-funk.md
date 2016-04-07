---
layout: post
title:  "Recreating Daft Punk's Da Funk"
date:   2016-04-03 21:18:00
categories: remake
---

This is the first of (I hope...) series of tutorials on using great [Overtone](http://overtone.github.io/) library for turning your computer into a musical instrument with Clojure.

Today we'll recreate Daft Punk's "Da Funk" [main theme](https://youtu.be/mmi60Bd4jSs?t=9). The track which had a huge influence for me (and many more) entering the world of electronic music. It's really simple but how huge!

### TL;DR
At the end of this tutorial we'll create this phrase:
<audio controls>
	<source src="/assets/sounds/dafunk/all.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/all.mp3" type="audio/mpeg"/>
</audio>


**EDIT:**
Based on HN/Reddit comments about some errors in my synth definitions making main theme sounding a lot different that the original I made some changes to this post. I left the corrections in the text, but code and sound samples are now updated. Thanks for all the suggestions!

### Theme

The nice thing about Daft Punk is that they "open sourced" a large amount of their music. You can find midi notes, loops, acapellas and beats for many of their songs (including Da Funk) on a site called [Daft Club](http://daft.club/daftabase). The track is also the subject of many YouTube tutorial videos, and appears in Mitchell Sigman's [Steal this sound](http://www.amazon.com/Keyboard-Presents-Steal-This-Sound/dp/1423492811) which make a perfect one to recreate using Overtone.

So from the midi file from Daft Club we can get these notes:

![notes](/assets/images/da-funk.png)

The key is ~~F major~~ G minor and BPM is 110.

To recreate the melody I'll use [Leipzig](https://github.com/ctford/leipzig) - superb library which provides a DSL for constructing melody as data structure.

{% highlight clojure %}
(require '[leipzig.melody :refer :all])
(require '[leipzig.live :as live])
(require '[leipzig.scale :as scale])

(def da-funk
  (->> (phrase [2 1/2 1/2 1/2 2.5 1/2 1/2 1/2 2.5 1/2 1/2 1/2 2.5 1 1]
               [0 -1 0 2 -3 -4 -3 -1 -5 -6 -5 -3 -7 -6 -5])
       (where :pitch (comp scale/G scale/minor))
       (all :part :da-funk)
       (all :amp 1)))
{% endhighlight %}

The base is `phrase` function which takes two sequences: durations and pitches (as degrees of a given scale). So as we're in ~~F major, F is 0, G is 1, A is 2~~ G minor, G is 0, A is 1, Bb is 2 and so on. If you want a pause, just use `nil` as pitch, but remember to handle it carefully later.

Now let's create instrument:

{% highlight clojure %}
(require '[overtone.live :refer :all])

(definst da-funk [freq 440 dur 1.0 amp 1.0]
   (let [env (env-gen (adsr 0.3 0.7 0.5 0.3)
	                    (line:kr 1.0 0.0 dur) :action FREE)
         osc (saw freq)]
     (-> osc (* env amp) pan2)))
{% endhighlight %}

So we use a saw-wave with ~~half of frequency (one octave down)~~ original frequency and add some attack and release to amplitude envelope shape.

Now we need to tell Leipzig that part `:da-funk` should use `da-funk` instrument:
{% highlight clojure %}
(defmethod live/play-note :da-funk [{hertz :pitch seconds :duration amp :amp}]
  (when hertz (da-funk :freq hert :dur seconds :amp (or amp 1))))
{% endhighlight %}

Finally, to play a melody we need to convert from midi notes to hertz and specify song tempo:
{% highlight clojure %}
(->> da-funk
    (wherever :pitch, :pitch temperament/equal)
    (tempo (bpm 110))
    live/play)		
{% endhighlight %}

You should hear something like this:
<audio controls>
	<source src="/assets/sounds/dafunk/synth1.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/synth1.mp3" type="audio/mpeg"/>
</audio>

OK, let's shape the sound a bit. Let's add second saw oscillator detuned by 5 semitones and mix them together:
{% highlight clojure %}
(definst da-funk [freq 440 dur 1.0 amp 1.0]
   (let [env (env-gen (adsr 0.3 0.7 0.5 0.3) (line:kr 1.0 0.0 dur) :action FREE)
         osc (mix [(saw freq)
                   (saw (* freq 0.7491535384383409))])]
     (-> osc
         (* env amp)
         pan2)))
{% endhighlight %}

This ~~`1.334`~~ `0.749` is a ratio of ~~adding~~ subtracting 5 semitones in hertz. This should sound like this:
<audio controls>
	<source src="/assets/sounds/dafunk/synth2.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/synth2.mp3" type="audio/mpeg"/>
</audio>

To create the "wah-wah" effect we need to apply a band-pass filter on the sound from oscillators:
{% highlight clojure %}
(definst da-funk [freq 440 dur 1.0 amp 1.0 cutoff 2200]
   (let [env (env-gen (adsr 0.3 0.7 0.5 0.3) (line:kr 1.0 0.0 dur) :action FREE)
         level (+ (* freq 0.25)
                  (env-gen (adsr 0.5 0.3 1 0.5) (line:kr 1.0 0.0 (/ dur 2)) :level-scale cutoff))
         osc (mix [(saw freq)
                   (saw (* freq 0.7491535384383409))])]
     (-> osc
         (bpf level 0.6)
         (* env amp)
         pan2)))
{% endhighlight %}

So we increase resonance a bit (0.6 vs. standard 0.5) and use center frequency of 2200 hertz. Then we apply a filter envelope: add attack and release of 0.5 and scale the envelope to half of note duration. You can of course experiment with these settings.
<audio controls>
	<source src="/assets/sounds/dafunk/synth3.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/synth3.mp3" type="audio/mpeg"/>
</audio>

And finally let's add some distortion - we could install `fx-distortion` on our instrument, but since it's integral part of the sound, let's just inline it:
{% highlight clojure %}
(definst da-funk [freq 440 dur 1.0 amp 1.0 cutoff 2200 boost 12 dist-level 0.015]
   (let [env (env-gen (adsr 0.3 0.7 0.5 0.3) (line:kr 1.0 0.0 dur) :action FREE)
         level (+ (* freq 0.25)
                  (env-gen (adsr 0.5 0.3 1 0.5) (line:kr 1.0 0.0 (/ dur 2)) :level-scale cutoff))
         osc (mix [(saw freq)
                   (saw (* freq 0.7491535384383409))])]
     (-> osc
         (bpf level 0.6)
         (* env amp)
         pan2
         (clip2 dist-level)
         (* boost)
         distort)))
{% endhighlight %}

<audio controls>
	<source src="/assets/sounds/dafunk/synth4.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/synth4.mp3" type="audio/mpeg"/>
</audio>

<br/>

### Drums

The drum pattern is a classic 4x4:

![drums](/assets/images/da-funk-beats.png)

We can also use Leipzig to construct the drums - it's actually taken from [whelmed repo](https://github.com/ctford/whelmed/blob/f937e74f150ed594c2d862834cf5ed41deb80f5a/src/whelmed/songs/love_and_fear.clj#L18) from Leipzig's author Chris Ford.

First, we need to create drum kit player:
{% highlight clojure %}
(defmethod live/play-note :beat [note]
  (when-let [fn (-> (get kit (:drum note)) :sound)]
    (fn :amp (:amp note))))
{% endhighlight %}

Where `kit` is a mapping from drum name to actual drum sound preloaded:
{% highlight clojure %}
{:close-hat {:sound #<buffer[live]: close-hat.aif 0,275313s stereo 7>, :amp 1},
 :fat-kick {:sound #<buffer[live]: fat-kick.aif 0,540542s stereo 5>, :amp 1},
 :horn {:sound #<buffer[live]: horn.aif 0,213021s stereo 8>, :amp 1},
 :kick {:sound #<buffer[live]: kick.aif 0,299833s stereo 0>, :amp 1},
 :open-hat {:sound #<buffer[live]: open-hat.aif 0,238000s stereo 2>, :amp 1},
 :snare {:sound #<buffer[live]: snare.aif 0,280875s stereo 6>, :amp 1}}
{% endhighlight %}

Again - the drum samples are taken from Daft Club resources.

We just need a helper `tap` function to program a sequence of a single drum
{% highlight clojure %}
(defn tap [drum times length & {:keys [amp] :or {amp 1}}]
  (map #(zipmap [:time :duration :drum :amp]
                [%1 (- length %1) drum amp]) times))
{% endhighlight %}

Now we are ready to construct the beat:
{% highlight clojure %}
(def da-beats
  (->>
    (reduce with
      [(tap :fat-kick (range 8) 8)
       (tap :kick (range 8) 8)
       (tap :snare (range 1 8 2) 8)
       (tap :close-hat (sort (concat [3.75 7.75] (range 1/2 8 1))) 8)])
    (all :part :beat)))
{% endhighlight %}

So we play fat-kick with kick every beat, snare every odd beat and hihats in the middle. Nice usage of Clojure collection framework.

To play only drums we need to recreate track like this:
{% highlight clojure %}
(->> da-beats
       (tempo (bpm 110))
       live/play)
{% endhighlight %}

<audio controls>
	<source src="/assets/sounds/dafunk/beat.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/beat.mp3" type="audio/mpeg"/>
</audio>

And finally, to mix the synth with drums we need to use `with` function:
{% highlight clojure %}
(->> da-funk
     (with (times 2 da-beats))
     (wherever :pitch, :pitch temperament/equal)
     (tempo (bpm 110))
     live/play)
{% endhighlight %}

<audio controls>
	<source src="/assets/sounds/dafunk/all.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/dafunk/all.mp3" type="audio/mpeg"/>
</audio>


That's it! This approach needs a little bit to understand, but it's really powerful in terms of representing simple songs. Also, it plays really well with live coding, which I plan to show in one of the next tutorials.

**EDIT:**
I'm aware that my version differs from the original song. I'm doing this blog mainly for fun and does not state that this approach could lead to a professional sounding tune.
