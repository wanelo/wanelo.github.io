---
layout: default
title: "BORAT: Bathroom Occupancy Remote Awareness Technology with Arduino"
author: Konstantin Gredeskoul
author_username: kig
---

> Note: this is the abridged and condensed version of a much longer and more detailed technical blog post.  If you find youself looking for extra detail, [read it from the source](http://kiguino.moos.io/2014/10/12/borat-bathroom-occupancy-wireless-detection-and-notification-with-arduino.html).

One of the great things about [Wanelo](http://wanelo.com/), is that we encourage inventiveness of all sorts.  Couple that with a bi-weekly
hack-day, and you get a lot of smart people coming up with a lot of really good ideas.

The project that's the subject of this blog post, does not fit in Wanelo roadmap, does not supper our production infrastructure, and hell – does not even help with our automated tests.  What it does do, however, is make it 100% unambiguous when it is the *right* time to go the bathroom at our relatively new office.

Shortly after we moved to our current office on Clementina Street in SOMA, the fact that the office has only two single-occupancy bathrooms, each on a separate floor, started to sink in.

I would imagine others had a similar experience to me: a couple times a day, when I really needed to use the restroom, I'd go to the downstairs bathroom only to find its door locked. I'd then run upstairs and find the other one locked too... Damn! I'd come back down only find out that someone else grabbed the first bathroom while I was upstairs. Nothing against them, but Argghh!

You can likely see how this could be a frustrating and disruptive experience even just once or twice. Now multiply that by 35 employees and every work day of the year, and you end up with an actual productivity problem.

Given my foray into Arduino over the last few months, I knew I could come up with a solution. I got approved for a small budget of about $200, and started looking around.

<div class="full">
    <img src="/images/bathroom-occupied-calm-technology.jpg"/>
</div>
<br />

>The problem was very simple: __people needed to know when each bathroom was occupied, or not__.  Just like on an airplane you can see bathroom light on/off, I wanted something similar for our two bathrooms. Something everyone could see.


{{site.data.macros.continue}}
___


### Broadcasting Occupancy Status

The first question on my mind was: *"How would I let everyone know, if one or both bathrooms are occupied?"*. I decided to follow a familiar airplane concept, and introduce a display unit, which would be large enough, and bright enough, to be visible from most parts of our office. Luckily we have an open plan, so placement was not an issue.

By that time I was playing with a couple of
[Rainbowduinos](http://www.amazon.com/Rainbowduino-LED-Driver-Platform-Atmega328/dp/B0068JYK0I?_encoding=UTF8&tag=kiguino-20)
which were driving
[8x8 LED Matrix displays](http://www.amazon.com/Super-Bright-RGB-LED-matrix/dp/B0068K01QE/?_encoding=UTF8&tag=kiguino-20), and I was pretty impressed with the results. I immediately knew I wanted to use these to indicate the status of occupancy. Initially I thought the green matrix should indicate that a bathroom is available, while red would mean occupied.  I basically went with this for the final version, although I added some
[lava-lamp like animations](https://github.com/kigster/Borat/blob/master/firmware/DisplayLED/DisplayLED.ino) to make things more interesting.

<div class="full">

<a href="https://raw.githubusercontent.com/kigster/Borat/master/images/real-life-examples/borat-at-wanelo.jpg" data-lightbox="enclosures" data-title="BORAT in Action!">
    <img src="https://raw.githubusercontent.com/kigster/Borat/master/images/real-life-examples/borat-at-wanelo.jpg"
    alt="Bathroom Occupancy Wireless Notification Arduino-based System" title="Live on the wall at work"/>
</a>
</div>
<br />

### Communication

The next question was – how would the display unit receive information about each bathroom status?

This part was easy – I picked up a few [nRF24L01+ radios](http://www.amazon.com/nRF24L01-Wireless-Transceiver-Arduino-Compatible/dp/B00E594ZX0/?_encoding=UTF8&tag=kiguino-20) for very cheap. Most excellent [RF24 communications library](http://maniacbug.github.io/RF24/) has some of the best Arduino C++ code I've seen, and so it instilled confidence in this approach.  Quick test of the radios at work showed that they are more than capable of reaching through the walls.

### Detecting Occupancy Status

The last problem was about actual occupancy detection.

And *this* is where things got quite a bit more tricky.

I brainstormed with a few folks on many an option. We discussed:

* __Detecting the status of the door's lock__. This would have been great, but required changing the locks in a rented building, which was not allowed. There are additional issues with this solution, but I won't go into that here.
* I saw some projects online where people used __magnetic strips__ to detect if the door is simply closed or open.  However, this requires everyone to leave the door open when they leave – not realistic!
* Some suggested relying on __light detection__ alone. Easy and cheap? Yes. Reliable? No. Not only that, but the bathroom fan is connected to the light switch, so people often leave the light on deliberately to keep the fan going.
* Another simple option would be to use a __motion sensor__. However this alone is simply not sufficient, as we'll see in a moment.

It was clear to me that a combination system was needed. Something that not only relies on light or motion detection alone, but uses a combination of multiple sensors. Pretty soon, I settled on using just these three:

* Light sensor
* Motion sensor
* Sonar distance sensor

Shortly after, I received all the pieces in the mail, had a plan in my head, and was ready to proceed with the implementation.

## The Solution

<a href="https://raw.githubusercontent.com/kigster/Borat/master/images/module-display/DisplayUnit-0.jpg" data-lightbox="enclosures" data-title="Display Unit">
    <img src="https://raw.githubusercontent.com/kigster/Borat/master/images/module-display/DisplayUnit-0.jpg" alt="" title="" class="small-right">
</a>

Enter the project (all of which is open sourced under an MIT License), which took good month working some nights and weekends this summer to complete: [BORAT: Bathroom Occupancy Remote Awareness Technology](https://github.com/kigster/borat). As should be clear by now, BORAT is an Arduino-based toilet occupancy notification system. It uses inexpensive wireless radios (nRF24L01+) to communicate occupancy status of one or more bathrooms to the main display unit located in a highly visible area.

You may be asking – why in the hell does the observer unit (which determines occupancy) need sonar? Well, if you're like me, you like to take your time when you are ... you know. Perhaps I don't move very much during this time. Maybe I'm reading my phone. This would cause the motion sensor to eventually give up and show a green light. How many times did you have automatic light go off on you, while in a public bathroom alone?

This is where sonar comes in. Aimed directly at someone sitting on the toilet, sonar will read a different distance with a person there versus without. You can configure the threshold after installing Observer, and — voila! Near 100% accuracy! Is it creepy to have a 2-eyed robot staring at you in the bathroom? I'll leave that question to the reader.

Wanelo can't live without this technology now. It's amazing how quickly humans get used to things that are actually useful.

Here is a diagram that explains various placement options and the overall concept.

<div class="full">
<a href="https://raw.githubusercontent.com/kigster/Borat/master/images/concept/layout-diagram.png" data-lightbox="enclosures" data-title="Concept Diagram">
    <img src="https://raw.githubusercontent.com/kigster/Borat/master/images/concept/layout-diagram.png" alt="Concept Diagram" title="Concept Diagram">
</a>
</div>
<br />

### Observer Unit Logic

Observers are responsible for communicating a binary status to the display unit: either __occupied__ or __available__. The display unit also has third status: __disconnected__, for each observer unit. But Observers don't have that.

How do Observers determine if the bathroom is occupied? They do so based on the following logic:

1. If the light is off, the bathroom is available
2. If the light is on, we look at the motion sensor - if it detects movement within the last 15 seconds, the bathroom is considered occupied
3. If the motion sensor did not pick up any activity, we then look at the sonar reading.
  * If the sonar (which is meant to be pointed at the toilet) is reading a distance below a given threshold, it means someone is sitting there, and so the bathroom is occupied.
  * Otherwise, it's available.

That's it!

All settings and thresholds, including timeouts, are meant to be tweaked individually for each bathroom. This is why Observer units contain a [rotary encoder knob](http://www.amazon.com/Rotary-Encoder-Development-Arduino-Compatible/dp/B00HSWXMDK/?_encoding=UTF8&tag=kiguino-20) and a connector for external serial port. This is meant to be a Serial LCD Display used only to configure the device.  More information about this feature is available from the [original blog post ](http://kiguino.moos.io/2014/10/12/borat-bathroom-occupancy-wireless-detection-and-notification-with-arduino.html).



The diagram below shows the components used in each Observer unit.

<div class="full">
<a href="https://raw.githubusercontent.com/kigster/Borat/master/images/concept/observer-components.jpg" data-lightbox="enclosures" data-title="Observer Components">
<img src="https://raw.githubusercontent.com/kigster/Borat/master/images/concept/observer-components.jpg" alt="Observer Components" title="Observer Components">
</a>
</div>
<br />

## Conclusion

What's the moral of the story?  Who am I kidding.

__It sure is nice to know if you can or can't use the restroom, without having to get up from your desk.__

Could have this been done cheaper?  Absolutely!  Smaller?  Definitely!  Neater, prettier, faster, etc?  You bet.

But I had a lot of fun along the way, and it's been such a pleasure to work with "atoms", not "bits" for a change (although you could say this project required a good deal of bits too :-)

And now it's a relic of Wanelo culture that is frozen in time.  Forever. No, Really.

<p>Thanks for reading,<br />
&mdash; Konstantin.
</p>








