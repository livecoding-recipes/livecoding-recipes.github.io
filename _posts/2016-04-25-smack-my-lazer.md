---
layout: post
title:  "Smack My Lazer - live coding"
date:   2016-04-25 19:18:00
categories: live-coding remake
---

In this post I'll introduce my live coding music environment built on top on [Overtone](http://overtone.github.io/) and [Leipzig](https://github.com/ctford/leipzig).

### TL;DR

This is an example session we'll be discussing here:

<iframe src="http://www.youtube.com/embed/JUq-YnuFt8w?color=white&theme=light"></iframe>

I'm using lots of samples from Major Lazer & DJ Snake - Lean On (feat. MØ) and a beat from Prodigy - Smack My B**ch Up.

### Main theme
First things first. To recreate the Lean On main theme we'll use Leipzig as in [previous post](http://overtone-recipes.github.io/remake/2016/04/03/recreating-da-funk.html). The notes are here:

![notes](/assets/images/leanon-notes.png)

And can be expressed as:

{% highlight clojure %}
(def leanon-chords
  [[-9 -2 0 2 4]
   [-8 -1 -3 1 3]
   [-7 0 2 4]
   [-5 -1 2 4 5 6]
   [-5 -1 2 3 4]])

(def leanon
  (let [[ch1 ch2 ch3 ch4 ch5] leanon-chords]
    (->> (phrase (concat (take 9 (cycle [1/2 1/4]))
                         [1/2]
                         (take 9 (cycle [1/2 1/4]))
                         [1/2])
                 [ch1 nil ch1 nil ch1 nil ch2 nil ch2 nil ch3 nil ch3 nil ch3 nil ch4 nil ch4 ch5])
         (wherever :pitch, :pitch (comp scale/low scale/G scale/minor))
         (all :part :plucky)
         (all :amp 1))))
{% endhighlight %}

And I also created a simple "plucky" sound based on square wave with some reverb:
{% highlight clojure %}
(definst plucky [freq 440 dur 1 amp 1 cutoff 1500 fil-dur 0.1]
  (let [env (env-gen (asr 0 1 1) (line:kr 1.0 0.0 dur) :action FREE)
        level (+ (* 0.85 freq) (env-gen (perc 0 fil-dur) :level-scale cutoff))]
    (-> (pulse freq)
        (lpf level)
        (free-verb :room 1 :wet 0.45)
        (* env amp))))
{% endhighlight %}

And to wire it up for Leipzig we need to implement `live/play` for `:plucky`
{% highlight clojure %}
(def controls (atom {:plucky {:amp 1.0 :cutoff 900}}))

(defmethod live/play-note :plucky [{hertz :pitch seconds :duration amp :amp}]
  (when hertz
    (let [params {:freq hertz :dur seconds :volume (or amp 1)}]
      (apply i/plucky (to-args (merge (:plucky @controls) params))))))
{% endhighlight %}

This `controls` atom lets us modify live the instrument amp and cutoff frequency.

So it sounds like this:

<audio controls>
  <source src="/assets/sounds/leanon/leanon.ogg" type="audio/ogg"/>
  <source src="/assets/sounds/leanon/leanon.mp3" type="audio/mpeg"/>
</audio>

### Sampler
What I'm doing a lot in this video is using a homemade sampler. I stole some ideas from Sam Aaron's Sonic Pi [sample packs](https://github.com/samaaron/sonic-pi/blob/019cfa1d19fbd122bb1beeb3faa4642f76809d20/etc/doc/tutorial/en/03.7-Sample-Packs.md#sample-packs) concept ;)

Basically, I have a directory when I put all the samples and use special meaning of samples file name to detect some properties of the sample.
So for example, I have `120_8_smack_beat.wav` which tells my sampler that this file is in 120 BPM, is 8 beats long and is named `:smack-beat`.
Because Lean On is ~98 BPM I can compute a correct ratio to match Smack My B**ch Up beat with Lean On melody.

Original beat (120 BPM):
<audio controls>
  <source src="/assets/sounds/leanon/smack120.ogg" type="audio/ogg"/>
  <source src="/assets/sounds/leanon/smack120.mp3" type="audio/mpeg"/>
</audio>

Scaled to 100 BPM with melody:
<audio controls>
  <source src="/assets/sounds/leanon/smack-lazer.ogg" type="audio/ogg"/>
  <source src="/assets/sounds/leanon/smack-lazer.mp3" type="audio/mpeg"/>
</audio>

I can also play only first few beats of some sample by using it like this `[0 :lean-chorus 2]`, which means that on time `0` I only want first 2 beats of `:lean-chorus` sample. The beat lengths are of course computed from current BPM.

### Song state
I found it really easy to manipulate to have my current state of the song like a map of tracks:

{% highlight clojure %}
{:beat   [{:time 0 :drum :kick :amp 1} ...]
 :plucky [{:pitch 39, :time 0, :duration 1/2, :part :plucky, :amp 1} ...]}
{% endhighlight %}

This allows me to modify the current state by using this `update-track` function. It just takes a key with track name and a new value of this track. So I can have my `plucky` instrument playing while I'm modifying the beat and so on:

{% highlight clojure %}
(update-track :beat (times 2 l/lean-beat))
{% endhighlight %}

It's also very easy to remove some of the notes as I'm doing just [before the chorus](https://youtu.be/JUq-YnuFt8w?t=91) to remove last frame from beat and theme:
{% highlight clojure %}
(def last-frame (fn [n] (>= (:time n) 12)))

(update-track :beat (->> (times 2 l/lean-beat)
                         (remove last-frame))
              :plucky (->> (times 2 l/leanon)
                           (remove last-frame))
              :sampler (t/sampler [[0 :lean-verse-2]
                                   [0 :smack-beat 8 1.5]
                                   [8 :smack-beat 4 1.5]]))
{% endhighlight %}

In Leipzig I can just use `live/jam` to loop current state of the song. When I modify something it'll we activated on next loop (same as `:live-loop` in Sonic Pi).

I you want to play around with this stuff, check out my [disclojure](https://github.com/pjagielski/disclojure) repo on GitHub.
