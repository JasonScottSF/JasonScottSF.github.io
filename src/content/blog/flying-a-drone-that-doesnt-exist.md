---
title: "Flying a Drone That Doesn't Exist"
description: "The part I needed got delayed, the drone is in pieces, and I'm getting on a plane. So I built an aircraft out of software instead. Here's what's actually inside a flight simulator that isn't a flight simulator, and the two tools that lied to me on the way there."
pubDate: 'Aug 02 2026'
---

The part I need is sitting in a warehouse somewhere with a delivery date that keeps sliding right. The drone is in pieces on the bench waiting for it. And I'm getting on a plane this week.

So I spent a morning building an aircraft that doesn't exist.

## It is not a model of the aircraft

Say "flight simulator" and people picture a video game. A model of a plane, approximated well enough to be fun, with the real physics sanded down until a laptop can keep up.

This is the other way around.

The flight software I run in the simulator is the *actual* flight software. Same firmware, same state estimator, same flight mode logic, same failsafe code, compiled for my laptop instead of for the flight controller board. Nothing about it is a stand-in. When it decides the battery is too low and it's time to come home, that decision comes out of the same code that will decide it for real at four hundred feet over a field.

What's faked is everything underneath. The physics of a quadcopter. The GPS. The gyros, the accelerometers, the barometer, the compass. All synthesized and handed to the flight code, which has no way of knowing and no reason to care. It reads its sensors, runs its math, and flies.

That inversion is the whole point. A model of an aircraft tells you about the model. Real firmware with a fake world underneath tells you about the firmware, which is the part that's going to hurt you.

## The five boxes

It's less mysterious than it sounds. Five pieces, each one swappable, none of them clever.

**The physics.** A simulated airframe. Thrust, drag, mass, the ground. Something has to answer the question "if the motors are doing this, where is the aircraft now."

**The flight code.** The real firmware, believing every lie the physics tells it.

**A socket.** The flight code speaks MAVLink, the same protocol it speaks to a ground station over a radio. In the simulator it speaks it over a TCP port on localhost instead of over a 915MHz link. Same words, different pipe.

**A router.** One aircraft, several things that want to talk to it. A small program sits in the middle, takes the single connection from the flight code, and fans it out to anything that asks. My ground station on one port, a script on another, a logger on a third.

**A ground station.** The same software I use with the real drone. It connects, sees an aircraft, draws a map, and cannot tell the difference. That last part is not a nice bonus. That's the reason any of this is worth doing.

Plug the transmitter in over USB and it shows up as a joystick, and now the sticks under my thumbs are driving software that thinks it's flying. My actual switch layout. My actual mode assignments. The whole loop, minus the part that falls.

## The first tool that lied

I got it built, launched it in the background, went to check that the ground station could see an aircraft. Nothing.

Went and read the log, and found the router had written a long, calm goodbye. Unloading module param. Unloading module relay. Unloading module arm. Unloading module mode. A dozen more. One tidy line per module, then it stopped.

No error. No crash. It came up, decided it was done, put everything away neatly, and left.

The reason is dumb and completely reasonable once you see it: that program is interactive. It expects a terminal, because normally you sit in front of it and type commands at a prompt. I'd launched it detached from any terminal at all. So it started, looked for its console, didn't find one, and shut itself down in the politest way available.

There's a flag for exactly this. One word. Everything worked.

But sit with the shape of that failure for a second, because it's the interesting part. Nothing was broken. Nothing reported a problem. The log didn't say "no terminal, exiting." It said "unloading module battery," twelve times in a row, which reads like a program finishing a job rather than one giving up on it. If I hadn't known what a healthy startup looked like, I'd have gone hunting for the bug in the wrong place. Probably for a while.

## The second tool that lied

While chasing that, I checked whether the port the ground station listens on was open. My tool said closed. I believed it, and spent a stretch convinced the router wasn't forwarding anything.

Two things wrong with that, and both were mine.

The tool defaults to checking TCP. The port I asked about is UDP. So it answered a question I hadn't asked, and answered it accurately.

And even the right question was the wrong question. Nothing *listens* on that port until the ground station starts. The router just fires packets at it and doesn't care whether anyone's home. So "closed" was the correct answer, in the sense that it was true, meant nothing, and pointed me directly away from the actual state of the system.

The fix was to stop asking sideways and do the thing the ground station does. Bind the port, wait, see what shows up. Which is when the aircraft appeared: heartbeat, attitude, GPS position, battery at 12.6 volts, sitting on the ground in Canberra. It defaults to Canberra. Ten thousand miles from my desk, which was somehow the most reassuring thing I saw all morning.

I wrote a whole post a few days ago about [a diagnostic tool telling me a confident lie](/blog/four-sensors-one-bus/) and me chasing it for hours. I would like to report that I have learned. I have not learned. I have only gotten faster at recognizing the feeling.

## Why this is the thing I built while blocked

The sensor ring I've been fighting is waiting on a part. Fine. But the interesting question was never "do the sensors report numbers." It's what the aircraft *does* when they report something alarming. How hard it stops. Whether it drifts. Whether the thing I told it to do at two meters is a thing I actually want at two meters.

None of that needs real sensors. It needs real flight code and a fake world, which is precisely what's now sitting on my laptop. I can hand the simulated aircraft a simulated obstacle and watch it decide, and by the time the part shows up I'll know what "working" is supposed to look like instead of finding out with props on.

The same goes for the stuff I actually care about getting right. Cut the radio link and watch it come home. Drain the battery and watch it land itself. Push it across a boundary and see it turn around. [I've said before that first flight shouldn't be the first test](/blog/first-flight-shouldnt-be-the-first-test/), and this is the machinery under that sentence. The first time a failsafe fires, I want it to be a number changing on a screen while I'm holding coffee.

Total cost: one morning, two tools lying to me in different dialects, and about two gigabytes of disk.

The aircraft on my bench still can't fly. The one on my laptop has been flying all morning, badly, on purpose, in Australia.
