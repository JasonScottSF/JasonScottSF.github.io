---
title: "Ask the Hardware"
description: "Three weeks, a multiplexer, a microcontroller, a bus repeater, and a voltage regulator, all built to solve a problem that did not exist. Here's what was actually wrong, and the habit that would have found it on day one."
pubDate: 'Aug 07 2026'
---

The proximity ring works. Four little lidars, one wire, ten readings a second each, straight into the flight controller. No multiplexer. No microcontroller translating between them. No bus repeater. No voltage regulator feeding the bus repeater.

I bought all of those things. I wired all of them in. None of them were necessary, and I want to talk about why, because the answer is embarrassing and useful in roughly equal measure.

## The story so far

Two posts ago I had four sensors on a shared wire and only one of them answering. I diagnosed it as electrical contention, which is a real thing that really happens, and I bought a multiplexer.

One post ago the multiplexer turned out not to be supported by my flight software, so I put a small microcontroller in the middle to read the sensors and translate for the autopilot. That worked, sort of, with a workaround stacked on a workaround.

Then a bus repeater arrived, which was supposed to make the multiplexer and the microcontroller unnecessary. It needed its own voltage regulator, which needed its own wiring, and after all of it the flight controller still could not see a single sensor.

At that point I had built four separate solutions to one problem and the problem was completely untouched.

## Three things were wrong

**The driver was wrong.** My autopilot software has a list of supported sensor types, and I had picked one that looked correct. It wasn't. The type I'd chosen speaks to sensors using a command-and-response conversation. My sensors don't do that. They expose a set of numbered registers you read directly, like reading values out of a table.

Nobody documents this. The official wiki lists my sensor under a different connection method entirely, with a type number that turns out to be for a completely different kind of wiring. So I went and read the actual source code of the drivers, found one that reads registers, and compared its register list against the sensor manual's register list, line by line. Ten registers, the same default address, even the same rule for deciding a reading is too weak to trust. Identical. That was the right driver, and no documentation anywhere says so.

**The wiring was wrong, in a way that looks right.** This is the good one.

Two wires carry the conversation on this kind of bus. One is data, one is a clock. On my flight controller, the clock is on pin 2 and the data is on pin 3. On my sensors, the data is on pin 2 and the clock is on pin 3.

They're backwards from each other.

So the obvious cable, the one that connects pin 2 to pin 2 and pin 3 to pin 3, feeds the clock into the data line and vice versa. It looks correct at every single connector. It measures correct on a meter, because both wires are connected to something. And critically, it survives being rebuilt, because when you rebuild it you rebuild it to the same mental model that was wrong the first time.

I rebuilt that harness four times.

**And the addresses were a lie.** Each sensor needs a unique address so the flight controller can tell them apart. I'd set those weeks ago with a little script I wrote, which checked the sensor was alive, sent the address change, sent the save command, and printed "done."

It printed "done" whether or not any of that worked. It never read the address back. It verified the sensor was healthy *before* the write and reported that as the write having succeeded. Two of my four sensors weren't where I thought they were, and nothing downstream ever contradicted it, because nothing downstream worked anyway.

## Any one of these hides the other two

This is the part worth sitting with.

Fix the wiring alone: nothing works, because the driver can't speak the protocol. Fix the driver alone: nothing works, because clock and data are crossed. Fix the addresses alone: nothing, twice over.

Every individual fix produced exactly the same result as doing nothing. Which means every time I tried something and it failed, I learned that the thing I'd just tried wasn't sufficient, and I could not tell whether it was necessary. So I'd undo it and try something else. I spent three weeks generating null results and reading them as "wrong theory" when several of them were "right theory, incomplete."

I don't have a clean rule for escaping that. What I have is a symptom to watch for: if you have tried six things and every single one produced an identical non-result, stop trying things. Multiple simultaneous faults are the most likely explanation, and the only way out is to shrink the system until it works at all, then grow it back one piece at a time.

Which is eventually what I did. One sensor. Four wires. No repeater, no regulator, nothing else in the path. It worked. Then two sensors. Then three. Then four. About ten minutes, after three weeks.

## Then the optical flow sensor

Same evening, riding the momentum, I wired up a downward-facing camera that measures ground movement. The official setup guide says to connect it to a Windows PC, run the manufacturer's configuration tool, and switch the module to a particular output protocol.

I don't have a Windows PC. I was about to install an emulator, and then I stopped and thought: I could just look at what it's already saying.

So I plugged it into a serial adapter and dumped the raw bytes. Not decoded, not parsed, just the actual numbers coming out of the thing. Right there in the first line: a repeating three-byte header that identifies exactly which protocol it was speaking. Not the one the guide assumed. A different one, that my autopilot also supports, that needed no configuration tool and no Windows at all.

Twelve seconds to answer a question I'd been about to spend an evening on.

## The habit

All four of these ended the same way. Not by reading better documentation, and not by having a smarter theory. By asking the hardware directly.

The driver question got answered by comparing the sensor's own register table against the driver's source. The address question got answered by reading the address back off the sensor rather than trusting that writing it had worked. The protocol question got answered by looking at raw bytes on a wire.

Documentation tells you what a device is supposed to do. It's written by someone who wasn't holding your specific unit, and it's often describing the configuration they expected you to have rather than the one you've got. The device will tell you what it actually does, but only if you ask a question narrow enough to have one answer.

"Why doesn't my proximity ring work" has no answer. "What address does this sensor respond to" has exactly one, and you can have it in a minute.

I've spent twenty years troubleshooting networks, where this instinct is second nature. Nobody debugs a routing problem by re-reading the vendor's marketing page. You look at the actual packets. Somehow I got to a new domain, one where I know less, and immediately started trusting documentation over observation, which is precisely backwards. The less you know about a system, the more you should be measuring it and the less you should be assuming.

## What I'm keeping

The multiplexer, the microcontroller, the repeater and the regulator are all in a drawer. I'm not throwing them out, but I'm not putting any of them back on the aircraft without evidence of a problem they solve. That's a rule now: no component earns a place on the airframe because it *might* help.

The scripts I wrote along the way survived, and the one I'm proudest of is dull. It sets a sensor address, then reads it back, and exits with an error if the sensor reports anything other than what was asked for. That's it. That tiny addition is the entire difference between three weeks of confusion and none, and I only wrote it after the confusion.

The aircraft now knows how far it is from things in four directions and how far it is from the ground, and it can see the ground moving underneath it. Two calibrations away from a first hover.

Nobody's writing a book about the day everything worked. But I'll take a working ring.
