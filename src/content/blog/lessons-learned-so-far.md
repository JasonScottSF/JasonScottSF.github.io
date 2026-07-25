---
title: 'Lessons Learned (So Far)'
description: "A running list of the small, sharp lessons this build keeps teaching me. Work in progress - I'll keep adding to it as the drone keeps finding new ways to humble me."
pubDate: 'Jul 26 2026'
---

**This one's a living document, not a finished post.** Every build throws off little lessons that are too small to justify their own article but too good to lose. This is where those go. I'll keep adding to it as I keep making new mistakes, which, given the pace so far, should be often.

---

## Don't trust the filename, trust the stream

The SIYI gimbal's RTSP URL is literally named `main.264`. It's H.265. I burned real time trying to decode it as H.264 before checking the actual SDP response. The label lied and I believed it instead of checking.

Lesson: verify what a stream, file, or API actually is before building against what it's called.

## Software decode will lie to you about whether your design works

First pass at the video pipeline used software H.265 decode. Technically worked, capped out around 1.5 frames a second, felt broken. Swapping to the Jetson's hardware decoder jumped it to a stable 25 fps on the exact same hardware. The model was never the bottleneck, I was starving it.

Lesson: benchmark the boring plumbing, decode and I/O, before assuming your compute budget is the constraint.

## Build failure modes that stop themselves

First test of gimbal auto-follow had the control-loop sign backwards. It drove the camera away from the target instead of toward it. Scary in theory. In practice it was a non-event: losing the target stops the gimbal, so a backwards loop just pushed me out of frame once and then quit. The fix was a one-line sign flip.

Lesson: when you're building a feedback loop, design the failure mode on purpose. A bug that runs away forever and a bug that self-limits are very different afternoons, and which one you get is a design choice, not luck.

## "Set it once" isn't the same as "configured"

Manually assigning a static IP to the Jetson's Ethernet link worked right up until the gimbal power-cycled and the link flapped, and the network manager quietly flushed the manual address back to DHCP-seeking every time. Had to make it a real persistent connection profile, not a one-off command.

Lesson: anything that needs to survive a reboot or a link bounce needs to be actual config, not a command you happened to run once while it was working.

## Infrastructure that isn't there when you need it most

The Jetson's dashboard target was originally a hostname that only resolved on my home network. The first time it needed to work away from that network, DNS resolution just failed and the whole video pipeline silently stopped reaching the dashboard.

Lesson: a system that's supposed to work away from home shouldn't depend on infrastructure that only exists at home. Match your dependencies to where the thing actually has to run, not where you happened to build it.

## Don't assume specs transfer between similar-sounding products

Almost mounted a gimbal on the wrong assumption because I was using clearance numbers from a frame I didn't actually end up with, while building on its shorter-legged sibling. Similar names, similar look, different landing gear height. Caught it before it mattered, but it was a real near-miss.

Lesson: when two products get talked about in the same breath, verify which one you actually have before trusting numbers you found for its cousin.

## Pin your toolchain in CI, don't trust the runner's defaults

First automated deploy failed outright because the CI runner defaulted to an older Node version than the framework needed. Worked fine locally the whole time, since my local install was already current.

Lesson: CI runs on somebody else's defaults, not yours. Pin what you need explicitly instead of finding out the hard way.

---

*More as they happen. If you've got your own version of any of these, I'd bet money you learned it the same way I did.*
