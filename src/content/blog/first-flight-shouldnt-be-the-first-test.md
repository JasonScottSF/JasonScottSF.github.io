---
title: "First Flight Shouldn't Be the First Test"
description: "A bug on my desk is a bug. The same bug in the air is a falling object. So the entire engineering side of this project is really one question asked over and over: how do I find the failure before the aircraft does? This is how I test a thing that flies, including the day my own safety net failed quietly and what caught it."
pubDate: 'Jul 27 2026'
---

Most of what I've written here is about capability. The camera learned to recognize me, to hunt for me, to notice the aircraft sharing my sky. That's the fun part. This post is about the unglamorous part that makes the fun part allowed to exist: making sure it actually works, every time, before it's holding itself up in the air over something I care about.

The stakes are the whole reason. A bug on my workbench is an annoyance. I see it, I swear, I fix it. The exact same bug once the thing is airborne is a battery and four motors deciding to become a projectile. There is no "I see it and fix it" at a hundred feet. So the engineering side of this project has quietly organized itself around a single question I keep asking in different forms: **how do I find the failure before the aircraft finds it for me?**

There are three answers I lean on, and one story about the day the answers themselves let me down.

## The failures that don't announce themselves

When people picture a bug, they picture a crash: a red error, a stack trace, something loud. Those barely worry me, because they show up the instant I run the code. The ones that keep me up are the quiet failures. The ones that don't throw anything, don't turn anything red, and just calmly do the wrong thing while looking completely fine.

A wrong byte in a command to the gimbal. The camera simply ignores it, and I'm left wondering why my zoom command "didn't take." A single flipped comparison in the pre-flight check that decides whether it's safe to fly, so a "go" that should have been a "no-go" looks exactly like a real one. A guard that's supposed to say "this data is stale, don't trust it" that quietly doesn't guard, so a frame gets labeled with information that was already ten seconds out of date.

None of those announce themselves. So those are precisely the things I write automated tests for, and almost nothing else. I don't test the camera, the radios, the network, the parts that fail loudly and immediately when they break. Testing those buys me brittle, fussy tests that mostly check whether my fake version of the world matches my other fake version of the world. Instead I test the small pieces of pure judgment: does the packet come out with the right bytes and the right checksum, does the go/no-go flip at exactly the right battery percentage, does the stale-data guard actually reject stale data.

The trick I like most is that I name each test after the failure it's there to prevent, not after the function it's testing. So the list of tests reads like a list of ways the thing could quietly hurt itself: the heartbeat went stale and it still said the link was fine, the zoom rounded past its limit and wrapped around, the traffic data was old and got trusted anyway. When one of them goes red, it isn't a cryptic assertion error. It's a sentence telling me which specific bad thing I just reintroduced. Every one of them runs automatically before any change is allowed to land, so the answer to "did I just break something quiet" arrives in about a tenth of a second instead of at altitude.

## Flying it before I fly it

Tests catch the small stuff. They don't catch "what does the whole aircraft do when the battery gets low mid-flight." For that I need to actually fly it, and I am absolutely not going to let the real aircraft's first low-battery event happen with the real aircraft.

So I fly a simulated one at my desk. The same flight-control software that will run on the real drone runs on my laptop, believing it's a real quadcopter, feeding out the same telemetry over the same protocol my dashboard already speaks. From the software's point of view there is no difference. From mine, the difference is that I can do horrible things to it and nobody has to go up on a ladder afterward.

And horrible things are exactly the point. I sit there and deliberately break it. I arm and disarm it a hundred times to watch the state track correctly. Then I cut the battery out from under it and watch whether it does the right thing, coming home or setting itself down instead of just falling. I kill the radio link and confirm it doesn't panic. I push it across a boundary I've drawn and check that it turns itself around. The first time I ever see a failsafe actually fire, I want it to be a number changing on my screen, with a simulated aircraft in no danger and me leaning back in a chair. Not a real drone in a real tree, teaching me the lesson the expensive way.

This is the same instinct as the [safety post](/blog/safety-first-and-last/), pointed inward. That one was about the boundaries the drone enforces on itself in the air. This is about rehearsing every one of those boundaries on the ground, over and over, until the real thing is boring.

## Assume I'll forget

The third piece isn't testing at all, it's a design stance, and it comes from knowing the person I most need to protect this aircraft from is a tired version of me six weeks from now who forgot something.

So the defaults are conservative, the status is loud, and everything gets logged. The system assumes the operator forgot to do the safe thing and does it anyway. Anything genuinely dangerous has to be proven safe in the simulator before it's allowed near the real hardware. I've written before about "current me protects future me," and this is where it lives in the code: I spend effort now, while I'm calm and paying attention, buying safety for a future moment when I won't be either.

## The day my safety net failed quietly

Here's the part I could leave out, and won't, because it's the most honest thing in the post.

I wrote that whole suite of tests to catch the quiet failures. I was pleased with it. And then, a little while later, I went looking for it and it wasn't there. The tests existed on my machine, they passed, I'd written them up as done, and somewhere between "done" and the real shared copy of the project, they had silently fallen on the floor. A merge that said it succeeded hadn't actually brought them along. My safety net, the thing whose entire job is catching failures that don't announce themselves, had itself failed without announcing anything.

There's a joke in there and there's also the actual lesson, which is the same lesson as the rest of the post one level up: trusting that something worked is not the same as checking that it did. "The tests are written" is not "the tests are running." "The change merged" is not "the change is there." Now I confirm the thing landed where it's supposed to live, not just that the process reported success, because a process reporting success is exactly the kind of quiet, confident wrongness I built the tests to catch in the first place.

## Why I bother, on a hobby project

None of this is required. It's a drone I'm building in my apartment. I could skip all of it, fly the thing, and probably be fine most of the time.

But "probably fine most of the time" is a description of how things end up in trees, and more importantly it's not the standard I hold myself to. The difference between someone who builds a cool drone and someone who builds aircraft is almost never the cool part. It's this part: the assumption that it will fail, the discipline of finding out how before it matters, and the honesty to admit when your own safeguards let you down. It flies, so I'm choosing to hold it to the standard of something that flies. That choice, more than any single feature, is the thing I actually want this project to say about me.
