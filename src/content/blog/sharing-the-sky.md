---
title: "Sharing the Sky"
description: "The last few posts were about the drone paying attention to things on the ground. This one is about teaching it to notice what's in the air with it, using the same signal that keeps airliners from running into each other, and why a camera that films aircraft turns out to be a machine that can label its own training data."
pubDate: 'Jul 27 2026'
---

Everything I've written so far has been about the drone looking down and out: tracking a person, recognizing them, hunting for them when they walk off. All of it is about things on the ground. But the thing I'm building is an aircraft, and an aircraft has neighbors. Somewhere above me right now there is a regional jet on approach and a Cessna poking around the coastline, and my drone, for all its cleverness about who's standing in the living room, had no idea any of them existed.

That bothered me more the longer I sat with it. The whole point of the safety thinking on this project is to assume the thing will eventually be somewhere it can hurt someone or something. "Somewhere it can hurt something" very much includes the same volume of air a crewed aircraft is using. So this post is about giving the drone a sense of the sky around it, and it turned out to be one of the more interesting afternoons of the build.

## The signal is already in the air

I didn't have to invent anything to sense other aircraft, because they're already shouting. Most crewed aircraft broadcast a little packet several times a second that says, in effect, "I am this flight, I am here, I am at this altitude, going this way." It's called ADS-B, and it's the backbone of how modern air traffic keeps itself sorted out. You can receive it with a cheap antenna, and there are public services that aggregate what thousands of hobbyist receivers pick up and hand it back as plain data.

The first time I pointed one of those services at my own coordinates, it came back with fifty-some aircraft inside thirty miles. One of them was an Alaska flight about four miles out and ten degrees up off my horizon, which is to say: a real airplane, close enough and high enough that if my camera were pointed the right way, it would be *in the shot*. That was the moment it stopped being an abstract idea and became a thing I clearly needed.

## Where am I, though

Here's the first question that turned out to be more subtle than it looked: to figure out where an aircraft is *relative to me*, I need to know where *I* am. And the obvious answers are all wrong.

The laptop can't tell me. It has no real GPS, it just guesses from wifi, which is fine for showing you a weather forecast and useless for aviation geometry. The tablet I fly with has a real GPS, but it's sitting on the ground next to me, and the camera isn't on the ground next to me, it's up on the drone. When the aircraft is a few hundred feet away and a hundred feet up, the difference between "where I'm standing" and "where the camera is" is the whole ballgame.

So the position has to come from the drone's own GPS, the one already wired into its flight controller. That sensor knows where the camera actually is and, just as importantly, how high it is, which I need for the vertical angle. On the bench, with no flight controller talking yet, it falls back to my fixed home coordinates. In the air, it uses the aircraft's own fix. It's a small thing, but getting it right is the difference between the system knowing where a plane is in *its* sky versus in *mine*, and those diverge the instant the drone leaves my hand.

## Turning a map into a sky

Once I know both positions, the rest is geometry I half-remembered and re-derived badly a couple of times. Each aircraft's broadcast is a latitude, a longitude, and an altitude. What I actually want is human, and camera-shaped: how far away is it, what compass bearing do I turn to face it, and how high do I tilt up to see it. Distance and bearing come off the usual great-circle math. The tilt, the elevation angle, is just the altitude difference over the horizontal distance, run through an arctangent.

Sorted by that elevation angle, the list suddenly reads like something a person would say. The thing highest in my sky floats to the top. The airliner grinding along at thirty-five thousand feet, technically overhead but a barely-there speck, sinks down where it belongs. What comes out the other end is a live, ranked answer to "what's up there, and where do I look," refreshed every several seconds, sitting right on the pre-flight page next to the battery and GPS readouts.

## The part I didn't expect

I built this for safety and awareness. What I didn't see coming until it was half working is that it quietly solves a completely different problem I'd been dreading.

I want the drone to get better at recognizing aircraft on sight, and getting better means training on examples, and training on examples means *labeling* them: someone, usually me, drawing a box around every plane in a pile of images and typing "airplane." It is the tedious, thankless part of every machine-learning project, and I'd been putting it off.

But think about what I now have. When my camera is recording and its own vision spots a shape it thinks is an aircraft, I can ask ADS-B, at that exact instant, what was actually up there. If the vision says "one airplane" and ADS-B says "yes, exactly one aircraft above your horizon right now," then that frame just labeled itself. The truth data and the image line up on their own. Every recording session pointed at the sky becomes a small, self-annotating training set. The safety feature and the data-collection chore turned out to be the same feature wearing two hats, and I genuinely did not plan that.

## What it can't do, and why I'm saying so

I want to be careful not to oversell this, because the honest limits are the interesting part.

Right now it can tell me *what's in my sky* accurately. What it can't yet do is reliably paint the exact callsign onto the exact pixel, because to project a specific aircraft to a specific spot in the frame I also need to know precisely which way the camera is pointing, its compass heading and its tilt, fused with the zoom. I have pieces of that and not all of it, so the on-screen labeling is honest about its uncertainty: when there's one aircraft in view and one contact overhead, it names it; when it can't be sure which is which, it says so and lists the candidates instead of confidently lying. A wrong label that looks certain is worse than an honest "one of these two."

And a deeper limit, the one that actually matters for safety: ADS-B only shows me aircraft that are *cooperating*. A glider with no transponder, another hobby drone, a bird, my own reflection in a window, none of them broadcast anything. So this is one layer of situational awareness, not a force field. It's the layer that catches the airliners and the Cessnas, which is a lot, and it's exactly the kind of thing that real detect-and-avoid systems treat as *one* input among several, never the whole answer. Knowing what it doesn't cover is the point, not a footnote.

## Why this one felt different

Most of what I've built on this drone has been about making it capable. This was the first piece that was about making it a good neighbor. It doesn't help the drone do anything to the world. It helps the drone understand that it's sharing airspace with people whose day depends on nobody doing anything stupid near them.

That framing is the one I keep coming back to as this project drifts from "cool camera that follows me" toward "small autonomous aircraft that has to earn its place in the sky." The tracking was table stakes. Knowing what else is up there, and being honest about the parts I can't see yet, feels like the first thing on this build that a real aviation engineer would nod at.
