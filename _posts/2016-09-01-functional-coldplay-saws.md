---
layout: post
title:  "Functional Coldplay - remaking \"supersaw\" synth from \"Sky Full of Stars\""
date:   2016-09-01 20:00:00
categories: remake
---

Today I'm going to show you how to recreate "supersaw" synth with an example of Coldplay's <a href="https://youtu.be/zp7NtW_hKJI?t=68">"Sky Full of Stars" chorus</a>.

Full sound:
<audio controls>
	<source src="/assets/sounds/coldplay/all.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/coldplay/all.mp3" type="audio/mpeg"/>
</audio>

I'm not a big fan of Coldplay, but must admit that they know how to follow the trends, and using a "supersaw"-like sound is really popular these days in both pop and EDM songs.

### Synth design

*The main ideas of this design are taken from this excellent <a href="http://www.attackmagazine.com/technique/synth-secrets/detuned-pad/">Attack Magazine tutorial</a> and original <a href="http://sccode.org/1-4YS">Supercollider implementation by coreyker</a> from sccode.org*

My Overtone implementation of this synth looks like this:
{% highlight clojure %}
(definst supersaw [freq 440 dur 0.2 release 0.5 amp 0.6 cutoff 3500 env-amount 0.5]
         (let [snd-fn (fn [freq]
                        (let [tune (ranged-rand 0.99 1.01)]
                          (-> (lf-saw (* freq tune))
                              (delay-c 0.005 (ranged-rand 0.0001 0.01)))))
               hi-saws (splay (repeatedly 5 #(snd-fn freq)))
               lo-saws (splay (repeatedly 5 #(snd-fn (/ freq 2))))
               noise (pink-noise)
               snd (+ (* 0.65 hi-saws) (* 0.85 lo-saws) (* 0.12 noise))
               env (env-gen (adsr 0.001 0.7 0.2 0.1) (line:kr 1 0 (+ dur release)) :action FREE)]
           (-> snd
               (clip2 0.45)
               (rlpf (+ freq (env-gen (adsr 0.001) (line:kr 1 0 dur) :level-scale cutoff)) 0.75)
               (free-verb :room 1.8 :mix 0.45)
               (* env amp)
               pan2)))
{% endhighlight %}

Here'a how it sounds played solo:

<audio controls>
	<source src="/assets/sounds/coldplay/saws.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/coldplay/saws.mp3" type="audio/mpeg"/>
</audio>

I'm aware that it's not a perfect one, but for purposes of this blog it should be fine and it's design shows some cool features of Overtone and functional programming.

The idea here is to take many saw oscillator sources and combine them together with slightly detuned pitch and a little bit of delay. The cool things we can use here in contrast to traditional synths are: **randomization** and **functional programming**. With randomization we can take a random value of both the pitch detune and the delay from a very small range (`ranged-rand`). This makes our synth a little different each time we register it in Supercollider. With functional programming we can define a function (`snd-fn`) which creates one saw synth for us, and then `repeatedly` invoke it a given number of times to create many detuned synths. In above case we create 5 `hi-saws` with original frequency and 5 `lo-saws` with a half of original frequency (one octave below). Next, we just add some `pink-noise` to enrich the sound a bit and just combine the saws and noise together. The rest is just an envelope (`env-gen`), filter (`rlpf`) and adding some reverb (`free-verb`).

## Layering ##
As you may know from <a href="/remake/2016/04/03/recreating-da-funk.html">previous posts</a> I use <a href="https://github.com/ctford/leipzig">Leipzig<a> DSL to define melodies playable with Overtone. The cool thing we can do with approach is "layering": use the same notes with different instruments and mix the sound together.

Here is "Sky Full of Stars" melody defined by Leipzig (taken from <a href="http://www.nonstop2k.com/midi-files/9734-coldplay-sky-full-of-stars-midi.html">this midi file</a>)
{% highlight clojure %}
(def stars
  (let [time-pattern [3/4 3/4 1 1 1/2]
        ch1 [-2 2 4 7] ch2 [-3 1 4 8] ch3 [-4 0 2 4 9]
        ch4 [-7 0 2 4 9] ch5 [-5 -1 1 8]]
    (->> (phrase (cycle time-pattern)
                 (concat [ch1 ch1 ch1 ch2 ch2]
                         (repeat 5 ch3) (repeat 5 ch4) (repeat 5 ch5)))
         (wherever :pitch, :pitch (comp scale/F scale/sharp scale/major))
         (all :part :supersaw)
         (all :amp 1))))
{% endhighlight %}

To use another instrument we have to change the `:part` value:
{% highlight clojure %}
(->> (times 2 stars)
     (all :part :stab))
{% endhighlight %}

I defined `:stab` and `:plucky` instruments to make this background for the lead synth:
<audio controls>
	<source src="/assets/sounds/coldplay/layer.ogg" type="audio/ogg"/>
	<source src="/assets/sounds/coldplay/layer.mp3" type="audio/mpeg"/>
</audio>

I also used this nice <a href="https://www.freesound.org/people/oceanictrancer/sounds/237075/">kick drum</a> with a lot of sidechain compression which makes it EDM-ish a bit.

So the whole track from the beginning of the post is defined like this:
{% highlight clojure %}
(def stars-track
  {
   :beat     (times 2 (->> (tap :bd (range 8) 8 :amp 0.75)
                           (all :part :beat)))
   :stab     (->> stars
                  (all :part :stab)
                  (all :amp 0.05)
                  (all :cutoff 5000))
   :supersaw (->> (times 2 stars)
                  (all :amp 0.6)
                  (all :cutoff 4000))
   :plucky   (->> (times 2 stars)
                  (all :part :plucky)
                  (all :amp 0.15)
                  (all :cutoff 6000))
   })
{% endhighlight %}
