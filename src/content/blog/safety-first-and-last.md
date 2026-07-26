---
title: 'Safety First and Last'
description: 'Everybody says safety first. Building a drone that flies itself taught me it is more like safety first and last, and mostly about protecting yourself from the guy who forgets things. That guy is me.'
pubDate: 'Jul 3 2026'
---

Everybody says "safety first." It's on posters. It's the thing you nod at on your way to the fun stuff. I used to nod at it too.

Then I started building a drone that flies itself, and "safety first" stopped being a poster and started being the actual job.

Here's the thing about a drone: it's a few pounds of spinning carbon fiber and lithium that I am deliberately teaching to make its own decisions. When it works, it's magic. When it doesn't, it's a flying lawnmower looking for a face. So I've stopped thinking about safety as just "first" and started thinking about it as first AND last. It's where I start every design decision, and it's the last thing standing when everything else I built falls over.

That's not what I expected going in, so let me walk through what it actually means.

## I am going to forget things

Not "might." Will.

The single biggest shift in how I build this stuff: I stopped designing for the sharp, focused version of me and started designing for the real one. Future me is going to be tired. Distracted. Three flights deep on a hot afternoon, showing off for a friend, not reading the screen carefully. Future me is going to forget to check something.

That's not me being down on myself, it's just how people work. Aviation figured this out ages ago. Checklists don't exist because pilots are dumb. They exist because smart, experienced people forget things under load, every single time, forever.

So here's the rule I build by now: current me has to protect future me. Every guardrail I put in today is a gift to the version of me who's about to do something careless next month.

## The safe thing should be the default thing

This is the "first" half.

If I forget to change a setting, the drone should do the boring safe thing, not the exciting dumb thing. Every time it powers on, it comes up in the most conservative mode there is. If I want it to do something spicier, I have to go deliberately turn that on. And if I forget to? It just sits there being boring and safe.

The failure mode of forgetting should always land on the safe side. If "I forgot" can hurt you, you built it wrong.

## Slow computers don't get to make fast safety calls

This one I had to sit with for a while.

My drone has two brains. There's the flight controller, a dedicated little real-time computer whose only job is to fly. And there's a Linux companion computer running all my fun AI stuff, which is way smarter but also way flakier. Linux can hang. It can get stuck. It can decide to think really hard about something for four seconds at the worst possible moment.

So who gets to decide when to abort a flight and bring the drone home?

Not the smart one. The dedicated flight controller owns that call, on its own clock, no matter what my clever AI is up to. The companion computer can suggest things, it can do all the graceful housekeeping, but it never gets to override the hard safety cutoff. If my Linux box wanders off into the weeds, the flight controller still brings the drone home right on schedule.

The smart system doesn't get to be in charge of the thing that keeps you alive. Felt backwards at first. Feels obvious now.

## Good safety is invisible

This is the part that got me.

I watched a Skydio demo once where the guy just tapped a button, threw the drone in the air, and it caught itself and started following him around. Looked effortless. Looked like magic.

It is not effortless. That "magic" is sitting on top of an absurd amount of safety engineering. Obstacle avoidance watching the whole time, sensor checks that would've flat refused to launch if anything were off, return-home standing by in the background. The trick isn't that they removed the guardrails. It's that they built so many of them, so well, that you never see one until the exact half-second you need it.

That flipped the whole goal for me. I'm not choosing between "safe" and "effortless." The effortless feeling IS the safety, done well enough to disappear. Best case, nobody watching my drone will ever know how much of it is just quietly refusing to let me screw up.

## Test the scary stuff where it can't hurt anybody

Last one, and it's short.

If I'm about to change something that could make the drone misbehave, I test it in a simulator first. I do not want to learn that my failsafe logic is backwards by watching my actual drone fly confidently into a tree. Sim is free. Trees are not. My face is definitely not.

## So, first and last

Safety first, because it's where every real decision starts. What's the default, who's in charge, what happens when I forget.

And safety last, because when all my clever code has thrown up its hands and quit, the failsafe is the thing still standing between me and a genuinely bad afternoon.

It's not the fun part. Nobody's going to be wowed that my drone lands itself calmly when the radio drops. But it's the part that lets me keep doing the fun part, with a drone that still exists and a face that also still exists.

That's the deal. Build for the guy who forgets. It's me. I'm the guy.
