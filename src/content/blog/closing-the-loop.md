---
title: "Closing the Loop: A Gimbal That Follows You"
description: "The camera half of the Skydio dream: get a drone to see a person and physically keep them in frame. Here's the pipeline, the control loop, and the afternoon the camera decided to run away from me."
pubDate: 'Jul 26 2026'
---

The dream, the one everybody has the first time they see a Skydio, is a drone that just follows you. You walk, it flies along with you, and the camera stays locked on you the whole time. It looks like magic.

That whole trick is really two loops stacked on top of each other. There's the aircraft flying itself to stay near you, and there's the camera moving to keep you framed. Different problems, different stakes. I decided to build the easier, lower-stakes one first: the camera. Get a gimbal to watch a person and physically track them as they move. No flying involved yet, just seeing and pointing.

Here's how that came together, including the part where it worked backwards and drove the camera away from me.

## The chain

The whole thing is a pipeline, and every link is its own small headache:

Camera to Jetson to detection to tracking to gimbal, and back to the camera.

The camera is a SIYI A8 mini gimbal. It streams video over Ethernet to an NVIDIA Jetson Orin Nano, the little edge-AI computer doing all the thinking. The Jetson finds the people in the frame, decides which one is the target, works out how far off-center they are, and tells the gimbal which way to move to fix it. Now the camera's pointing at a slightly different spot, so the whole thing runs again. Thirty times a second, ideally.

Let me walk the interesting links.

## Seeing

The detection is YOLOv8, running right on the Jetson. Feed it a frame, it hands back boxes: "person here, 86% sure." On top of that runs ByteTrack, which does the boring-but-critical job of remembering that the person in this frame is the same person from the last frame, and tagging them with a stable ID. A box plus a persistent ID is enough to lock onto one specific target and ignore everybody else.

Running this on a little Orin Nano, at the edge, on battery, is the whole game. It's easy to do computer vision on a desktop with a fat GPU. Doing it on something you can bolt to an aircraft is where the real constraints show up. Which brings me to the detour that ate an afternoon.

## The decode detour

The first time I wired the real camera into the pipeline, I got about 1.5 frames per second. Slideshow, not video. The tracking technically worked, it was just useless.

The culprit was video decode. The A8 mini streams H.265, and I was decoding it in software, on the CPU, which is a terrible use of a CPU. The Jetson has a dedicated hardware video decoder sitting right there, idle, built for exactly this. Once I routed the stream through the hardware decoder instead, the same pipeline jumped to a stable 25 frames per second and the CPU stopped sweating.

That's the kind of thing that never shows up in a demo but absolutely decides whether an edge system is viable. The model was never the bottleneck. Feeding it was.

(There was also a genuinely stupid half-hour where the stream is named "main.264" but is actually H.265, so I was trying to decode it with the wrong decoder and getting nothing out. The filename lied. Always trust the stream, not the label.)

## Closing the loop

Here's the part I actually enjoy.

Once the Jetson knows where the locked target is, it knows how far off-center they are, in pixels, in both directions. That offset is an error signal, and driving an error signal to zero is just a control loop. Target's off to the right? Pan right, proportional to how far off they are. Drifting up? Tilt up. The further off-center they get, the faster the gimbal moves to catch up, and as they come back toward the middle, it eases off. There's a small dead zone in the center so it doesn't jitter and buzz when you're basically centered already.

That's a proportional controller, the plainest, most honest control loop there is. And watching it work for the first time, watching a physical camera swing to keep me centered as I walked across the room, was one of those genuinely great moments in a build. The math left the screen and moved something in the real world.

## The afternoon it ran away

It did not work on the first try. It worked backwards.

I locked onto myself, turned on follow, and the camera immediately swung the other way and drove me straight out of frame. Classic sign error: I'd built positive feedback instead of negative. Instead of chasing the target toward center, it was shoving it toward the edge as fast as it could.

Two things made that a funny story instead of a scary one. First, the fix is a single flipped sign. Second, and this is the part I actually care about, it couldn't run away. The instant it pushed me out of frame, it lost the target, and losing the target makes it stop. The failure was self-limiting by design. A backwards control loop that happily spins forever is a very different afternoon than one that shoves the target off-screen and then politely gives up.

That's not luck. That's the [safety-first-and-last](/blog/safety-first-and-last/) habit showing up in a small way: build it so the dumb failure mode stops itself.

## Where it falls short (on purpose, for now)

Here's the honest limitation, and it's the whole reason there's a next post coming.

Right now the lock is tied to that ByteTrack ID. It follows *a track*, not *a person*. So the second you walk out of frame, or something briefly blocks you, the track dies. Walk back in a few seconds later and you're a brand-new ID as far as the system is concerned. A total stranger. It has no idea you're the same person it was just glued to.

That's fine for "keep the camera on the guy who's already in frame." It's useless for "find that specific person again after they've been gone." And the second one is the actually-interesting problem.

The fix is a second, slower loop running alongside the fast one. The fast loop keeps the camera locked on whatever it's following, frame to frame. The slow loop does something harder and more human: it looks at people and asks *who is this*, by appearance, so it can recognize you and re-lock even after you've left and come back. That's re-identification, and it's what turns "follow the box" into "follow *you*."

That's what I'm building next.
