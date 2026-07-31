---
title: "So I Bought the Mux"
description: "The last post about the four-sensor proximity ring ended on a very satisfying 'buy the mux, done.' Then I bought the mux. Here's the follow-up where a clean fix turned into a three-part sequel, and how I finally learned that knowing when to stop is a real engineering skill."
pubDate: 'Jul 30 2026'
---

The [last post about the four-sensor proximity ring](/blog/four-sensors-one-bus/) ended on a very satisfying note: "just buy the mux." I said it like the fix was obvious, and I meant it. Then I bought the mux, and everything I said turned out to be wrong in useful and educational ways. Here's the follow-up.

## The bad research

Between posts, I ordered a TCA9548A. Wired it into the aircraft, felt smart, opened the ArduPilot docs to find the "how to configure a mux" section I'd assumed existed. It didn't. I read closer. The feature request has been open on GitHub since 2022. There's an unmerged community fork by some helpful person that hacks it into working, if you're willing to compile ArduPilot from source and repurpose two unrelated parameters. The author of that fork explicitly calls it "not a good architectural solution" and says they don't plan to generalize it.

So the tidy plug-and-play upgrade I'd promised myself required either a full firmware build or a completely different approach.

The completely different approach won because it was a Sunday and I don't like recompiling autopilot firmware on a Sunday.

## The pivot

I decided the mux could still be useful, just not talking to the flight controller directly. Instead I'd stick a small microcontroller in the middle. The microcontroller reads all four sensors through the mux on its own private I2C bus, then sends the readings to the flight controller as MAVLink messages over UART. From the flight controller's point of view, it's just a MAVLink rangefinder source. No mux, no clever driver work required.

I had a Raspberry Pi Pico 2 in a drawer, so that's what I used. CircuitPython, a small library called `mavlite` designed exactly for this pattern, an existing example that already did the mux-plus-multiple-sensors dance for a different family of sensor. Reasonable, right?

## The rabbit that showed up next

The aggregator worked in about an hour. All four sensors reading, MAVLink flowing to the flight controller. Three of the four rangefinder slots on the flight controller consumed the data cleanly. The fourth kept saying No Data.

That fourth slot was whichever one was configured for orientation zero. If I set forward to orientation zero, forward failed. If I moved forward to orientation seven and set back to orientation zero, back failed. The failure followed the value, not the slot.

At this point I want to note something honestly. I confidently told myself, out loud, on Discord, and to my notes file, that this was a bug in ArduPilot's rangefinder MAVLink driver. It felt like a bug. The evidence was consistent: whichever slot got orientation zero didn't work.

Then I actually read the driver source. It's about six lines of substance, and it does exactly what you'd expect: compare the incoming message's orientation field to the slot's configured orientation, update the slot on a match. There is no code that treats zero specially. Nothing filters, nothing rejects, nothing does anything different at zero versus any other value.

Which means the "bug" I was confidently working around was, at best, unexplained. Something in the pipeline was dropping orientation-zero messages, but it wasn't the driver I'd blamed. Could be a routing quirk I don't see, a version-specific issue in my exact firmware build, something upstream I never inspected. I don't know. I still don't. And I'd already written the code assuming my diagnosis was right.

Note to self: don't declare a component broken until you've read its actual source. "It looks like it must be" is not the same as "I confirmed it is."

## The ghost

While all of this was going on, a separate problem kept reappearing. Every couple of hours, or on any power cycle, or seemingly whenever the humidity felt like it, CircuitPython would refuse to initialize its I2C peripheral because it couldn't detect pull-up resistors on the bus. This despite me having soldered proper external pull-ups onto the lines and measured them, repeatedly, at exactly the right voltage. The check was lying. But it was a lying check I couldn't skip.

I tried the software alternative, which doesn't do that check. It successfully initialized every time. It also read zero from every sensor, forever, because a software-bitbanged I2C doesn't play nice with this mux plus these sensors at this timing. Great.

So the choice was between a hardware I2C that sometimes refused to start, and a software I2C that started every time but couldn't actually talk. I wrote a retry loop around the hardware version. It succeeded on some boots and gave up after ten tries on others. There was no correlation with anything I could measure.

## The turn

At some point I realized I'd been at the bench for six hours, every fix was breaking something adjacent, and I'd made the aircraft slightly worse in the last two hours than it had been at hour four. I closed the laptop.

## The reframe

Here's what I decided the next morning, and it's what makes this post worth writing.

I have two aircraft in the pipeline. The little one I've been fighting with, and a much bigger one showing up in a week whose flight controller has several separate I2C buses instead of one. On the bigger aircraft, the entire problem I've been solving doesn't exist. Two sensors per bus with two buses each is well within what a single I2C peripheral can drive without help. No mux, no microcontroller in the middle, no CircuitPython, no software-emulated fallback, no mystery orientation-zero rejection, no pull-up ghost. Every workaround I built exists to solve a problem that goes away when the platform gets bigger.

Which means all of the workarounds live on the little aircraft, permanently. That's not a failure. That's what the little aircraft is now: the constrained platform where I learned things the hard way, kept flying because it was already built, and quietly still works if you don't look too closely at the sensor labeled "forward-left" that's actually pointing forward. The bigger aircraft gets a clean design that inherits none of the debt.

## What I actually learned

The lesson isn't about muxes, or MAVLink, or CircuitPython, or any specific piece of hardware. It's this:

**The instinct to keep pushing when everything is breaking is the same instinct that gets aircraft into trees.** Being able to say "this platform can't cleanly do what I'm trying to do, so I'll do it on the platform that can" is a real engineering skill, and I don't have enough hours on it. The default is to keep debugging, because debugging feels like progress. Sometimes the actual progress is closing the laptop and waiting a week for a better platform.

The second lesson is more embarrassing: don't confidently declare a component broken until you've read its actual source. I called ArduPilot's rangefinder handler bugged three separate times in as many hours before I finally opened the file and found six lines that couldn't possibly do what I was accusing them of.

## What comes next

The big drone gets the clean-sheet proximity design. The little drone keeps its Rube Goldberg aggregator, keeps flying, and does what little drones are for: getting the miles that teach you what you actually need.

If I hadn't spent all those hours fighting the small platform's sensor bus, I wouldn't have known exactly why the bigger platform's extra buses were worth waiting for. So even the failure mode was doing something useful. I just couldn't see it from inside it.

Nobody's writing a book about the day everything worked. This one was worth writing.
