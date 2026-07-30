---
title: "Four Sensors, One Bus, One Bad Day"
description: "I lost a day to four tiny lidars and a bus that wouldn't carry them. The flight controller was telling me exactly what was wrong the whole time. I just hadn't bothered to learn what it was saying."
pubDate: 'Jul 30 2026'
---

I lost most of a day to four little lidar sensors and a bus that didn't want to carry them.

The whole time, the flight controller was telling me exactly what was wrong. I wasn't listening. The pre-arm messages ArduPilot kept throwing at me weren't noise, they were a diagnostic language, and each variation meant something totally different. I was reading all of them as "it won't arm." Which, technically, was correct, but not useful.

The setup, briefly. The drone has four little distance sensors bolted around its middle. Front, back, left, right. They tell the flight controller what's near the aircraft so it can refuse to fly into it. All four talk to the FC over one shared wire, because that's how sensor buses have worked for the last thirty years.

Trouble started the moment there were four of them.

One at a time on the bench, each one worked fine. Two together, fine. Four together? Three of them went silent and the whole system started behaving like it had opinions. Every time I thought I had it fixed, it would either fail to arm, or arm and complain, or arm and refuse to move, or complain about a completely different sensor than it was complaining about ten minutes earlier.

That last bit is what saved me. The complaints kept changing. I just had to notice that they were changing.

## The pre-arm as a language

ArduPilot won't arm the motors if a pre-flight check fails. It tells you why in a little text banner. I'd been treating those banners as "the thing that won't let me fly." Turns out they're a proper diagnostic language, and each variation is pointing at a very different failure. Here's the four I collected on the way to enlightenment.

**"Rangefinder 3: Not Detected."** The FC went looking for the aft sensor at boot and did not find it. Whatever was supposed to answer at that address, didn't. Doesn't mean the sensor is broken. Means the FC never saw it during the roll call at startup. Check addressing. Check wiring. Check that pin 5 is actually grounded like you thought it was.

**"Rangefinder 3: No Data."** Completely different failure, and this is the one that tricked me for a while. The FC *did* find the sensor at boot. Roll call passed. Then, at some point during operation, that specific sensor just stopped answering. Cold solder joint. Bad crimp. Wire under tension. Or, in my case, four devices sharing a bus that could really only drive three.

**"PRX1: No Data."** Now we've escalated. It's not one sensor anymore, it's the whole proximity system, silent. The individual rangefinders may still technically be there, but the aggregated picture ArduPilot's avoidance logic wants has stopped arriving. When you see this, the fault is upstream of any single sensor. Usually the bus itself has degraded to the point where nothing's getting through cleanly.

**"Proximity 90 deg, 0.27m (want > 0.6m)."** This one looks like a failure. It's actually a party. It means the sensors are working perfectly. So perfectly that they can see the edge of my workbench sitting 27cm from the starboard sensor, and ArduPilot is refusing to arm because obviously you shouldn't spin motors up with an obstacle that close. First time I saw this message I knew the ring was alive. The check wasn't broken. The check was doing its job. I moved the drone.

Four messages, four failure modes, four different fixes. I'd been treating them as one message and throwing solutions at the wall until one worked. Once I started reading the words, I could match the fix to the failure.

## The mistake I chased for hours

The thing that ate the most time wasn't even a real problem. It was a lie the ground station was telling me.

In QGroundControl there's a MAVLink inspector, which shows you the messages flying past. I could see distance readings from one sensor arriving. Never the other three. Silence.

I spent hours convinced three of my sensors had bad addresses, or the wrong firmware, or were somehow secretly a different model. Reprogrammed them. Reflashed them. Considered blaming the manufacturer. Wrote a Python script to talk to each one directly over UART, confirmed they were all fine and had exactly the right addresses. Plugged them back on the bus. Watched three of them go "silent" again.

Except they weren't silent. They were transmitting. All four sensors send messages of the same type, the inspector shows you the *latest* message it received, and the display refreshes about twice a second. The four sensors were sending round-robin at a rate that just happened to put the same one in the display slot right before each refresh. Every other sensor's message was there, arriving normally, and being immediately overwritten by the next arrival before the screen could paint.

To me, watching the screen, it looked like only one sensor was ever transmitting. In reality, all four were, and I was reading the tool wrong. Confidently. For hours.

The lesson isn't "MAVLink inspector bad." It's a fine tool. The lesson is your diagnostic tool can lie to you without being broken, and if you don't understand what it's actually showing you, you'll invent problems that don't exist and go chase them. Which I did.

## What was actually wrong

After all that theater, the real problem was something I could have spotted in fifteen minutes if I'd been looking for it. Four sensors on one shared I2C bus is right at the edge of what the bus can drive. The signal edges get lazy, the FC gets what it can at boot, and after that the sensors drift in and out depending on wire length and temperature and vibration and whether it's a leap year.

Clean fix is a five-dollar chip called an I2C multiplexer. It gives each sensor its own private bus and lets the FC talk to them one at a time, so contention just... goes away.

I own several of these. I could not remember which drawer I put them in. So the mux install is queued for the next bench session, and in the meantime the aircraft has three-sensor coverage. Which is fine, because I'm not planning to fly backward on the first hover anyway. Anyone who watches me try to hover in Loiter mode on a maiden flight and thinks "yeah, but is the *back* covered" is not the audience I'm optimizing for.

I *did* try smaller pull-up resistors first, which give the bus more current to drive its own signals with. That bought me about ten minutes before everything fell over again. Which is diagnostic in its own right: if stronger pull-ups buy you time but not stability, the bus itself is the wrong architecture for the number of devices you're asking it to carry. Fix the architecture, not the resistors. I know this now. Yesterday I did not.

## Why I'm writing this down

I almost didn't post this. It felt embarrassingly basic. Anyone who's built a couple of these knows about I2C bus contention. They know to reach for a mux when the device count gets past three. They know to read pre-arm text as an actual sentence instead of a red light. Table stakes.

Except it isn't. It's table stakes for someone who's done it ten times. If you're doing it once, it's a day of confused debugging, a small pile of unnecessarily reprogrammed sensors, and a growing suspicion your soldering iron has personally wronged you. The gap between "I've seen this pattern before" and "I've never seen this pattern before" is exactly one instance of it, and the only way to shorten that gap is to have somebody else's day-of-flailing already written down for you.

So, here you go. If you're building a proximity ring on a hobby drone and it isn't working, read the pre-arm text word for word. "Not Detected" and "No Data" and "PRX1: No Data" and "Proximity 90 deg, 0.27m" are four different problems asking for four different fixes. Don't treat them as one message. And if you're already past three sensors on a shared bus, just buy the mux. Present you doesn't get to talk you out of five dollars when future you is looking at a whole day.

The aircraft is closer to flying now than it was before all this happened. Not because the ring is done, but because I finally understand what it will tell me when it isn't. Given that a safety system that doesn't fail loudly isn't much of a safety system, that's arguably the more valuable result. It's also the more expensive one, in units of afternoon.
