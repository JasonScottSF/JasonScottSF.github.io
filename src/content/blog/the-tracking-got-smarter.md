---
title: "The Tracking Got Smarter"
description: "It could recognize me. Then I spent a few evenings walking around my apartment breaking it, and every way it failed taught it something. Point-to-lock, a camera that hunts for you when you leave, and the afternoon a search behavior chased its own reflection into a wall."
pubDate: 'Jul 26 2026'
---

The last post ended with the camera able to recognize me and re-lock after I left and came back. That felt like the finish line. It wasn't. It was the point where the thing became good enough to test seriously, and testing it seriously meant walking around my apartment trying to lose it.

Every evening I did that, it failed in some specific, interesting way. And the useful thing I want to get across in this post is that I didn't sit down and design the tracking to be smarter. I found the exact spot where it was dumb, over and over, and fixed that spot. The failures were the design document. I just had to keep reading them.

## Lock by pointing at someone

The recognition worked by uploading a photo. You'd hand it a picture, it would build a fingerprint, and then it would lock onto that person when it saw them. Fine, but clunky. Half the time I just wanted to point at someone already on screen and say "that one."

So I made tapping a person do the whole thing. When you tap to lock, the system quietly starts collecting that person's appearance on its own, a few snapshots over the next several seconds as they move around. No photo upload, no ceremony. You pointed at them, which is you telling it who matters, and it takes that as permission to learn them.

That small change had a subtle trap in it that took a real failure to find, which I'll get to. But the feeling of it was exactly right: tap a person, and now the camera knows *them*, not just a box around them, from a single tap.

## The afternoon of wrong directions

I wanted to be able to touch a spot on the video and have the camera swing to look there and zoom in. Point at something across the room, camera reframes on it. Simple to describe.

It was not simple to build, and the way it went wrong is the most instructive thing in this whole post.

My first version did the obvious thing: work out the angle from where you tapped, tell the gimbal to go to that angle, done. To turn a tap into an angle you need to know the camera's field of view, how the digital zoom changes it, and which direction is which. I got all three slightly wrong, in sequence, over an afternoon. It slewed the wrong way. I flipped a sign. Now it slewed the *other* wrong way, because the first symptom had been masking a second bug. I fixed that and it went the right direction but only traveled halfway. The field-of-view number I was using came off a spec sheet and wasn't real. Every fix revealed the next wrong assumption underneath it.

At some point I stopped and realized I was trying to compute my way to an answer the camera could just *show* me. Instead of calculating an angle and trusting it blindly, I could watch the video and measure how far the image actually moved as the camera turned, and steer based on that. Close the loop on what the camera sees rather than on a pile of numbers I'd have to get all correct at once.

I threw out the angle math entirely. The new version nudges the gimbal, watches how much the scene shifted, and keeps nudging until the thing you tapped is centered. It needs no field-of-view number, no zoom math, no guessing which way is which, because it just corrects toward what it observes. It went from an afternoon of wrong directions to working on the first try, and it's the same idea as the follow controller from two posts ago: don't calculate the world, watch it and react.

The lesson I keep relearning on this project: hardware datasheets describe an ideal part, not the one on your bench. The stream is labeled one thing and is actually another. The "field of view" is a marketing number. Every time I trust a spec instead of measuring the real thing, it costs me an afternoon. Measuring costs minutes.

## Teaching it to hunt

Here's the gap that bugged me most. I'd lock onto myself, walk out a door, and the camera would sit there staring at the door I left through. I'd come back in a *different* door and it wouldn't notice until I physically walked into the narrow slice it was still watching. It had lost me and just... kept looking at the last place it saw me, like a dog watching the spot where the ball went behind the couch.

What I wanted was for it to actually *look* for me. So I built a mode I ended up calling hunt. When it loses the locked person for more than a second or so, it recenters, zooms all the way out to take in the whole room, and then watches for movement. Anything that moves, it turns toward, gets a proper look, and checks whether it's you. If it is, it re-locks and resumes. If it's the cat, it glances, dismisses it, and keeps watching. The recognition still makes the final call, so hunting can look at the wrong thing but never lock onto it.

It did not work on the first try. It worked spectacularly wrong on the first try, and it's a good enough story to be worth telling straight.

I locked on, walked out, and the camera swung all the way to the right, hit the end of its travel, and sat there drifting against the stop. It had run away completely.

The cause is a genuinely elegant bug. To detect motion, I was comparing each video frame to the one before it and looking at what changed. That works perfectly when the camera is still. But the instant the camera starts turning, the *entire image* shifts, so now everything has changed, so my motion detector sees motion everywhere, so it turns harder to chase it, which shifts the image more, which looks like more motion. The camera was chasing its own movement. It had mistaken the act of looking for the thing it was looking for, and it did that until it physically couldn't turn any further.

The fix is to only measure motion when the camera is holding still. Now it works in pulses: hold still, look for movement, make one short move toward it, stop, let the scene settle, look again. It physically cannot run away anymore, because every move is followed by a mandatory pause. Slower, deliberate, correct.

And then, while it was sitting there mid-hunt, I put my hand in front of the lens to see what it would do. It locked onto my hand. It decided my hand was me. Which, if you think about it, is not entirely wrong, and is exactly the kind of not-entirely-wrong that makes this hard. My hand up close filled the frame, got recognized as a person-sized shape, and matched my own appearance well enough, because it's my skin and my sleeve. So now there's a shape check: to be eligible for a lock, a thing has to be roughly person-shaped, taller than it is wide. A hand shoved at the lens is square, and gets rejected before recognition even weighs in. Geometry catching what appearance couldn't, which is becoming a theme.

## The trap in learning who you are

That tap-to-lock feature, the one where pointing at someone makes the system learn their appearance on its own, had a trap in it. It sprung the evening a roommate walked through.

In bad light we were both just silhouettes, similar build, both in dark clothes, and the camera flatly could not tell us apart. I've written up *why* that's unfixable in the [recognition post](/blog/who-is-this/), because the short version is that no code change creates information the image doesn't contain, and the score data proves it. But the part that belongs here is the trap, not the limit.

Because tap-to-lock keeps learning the locked person as they move, the moment it locked onto the wrong guy it started learning *him* as me. It was teaching itself the wrong answer, getting more confident about exactly the thing it had gotten wrong. A convenience feature had quietly become a feedback loop where a mistake trains the system to keep making it. The fix is to freeze what it's learned once it's captured, so a bad lock can't rewrite who you are. That doesn't make two silhouettes distinguishable, nothing could, but it stops the system from digging its own hole deeper. The lesson I took: anything that learns automatically will just as happily learn the wrong thing, so the moment you let a system teach itself, you have to ask what happens when it's wrong.

## What actually got smarter

None of this came from me being clever up front. Every improvement in this post exists because I took the thing into a real room, watched it fail in a specific way, and fixed that exact failure. Point-to-lock came from the photo upload being annoying. The visual steering came from an afternoon of wrong math. Hunt came from a camera staring at the wrong door. The pulse-and-settle came from it chasing its own motion into a wall. The shape check came from it locking onto my hand. The frozen appearance came from it teaching itself the wrong person.

That's the whole method, and it's the part I'd want an autonomy team to notice: I don't trust this system because I designed it well. I trust the specific things it does because I've watched each one fail and fixed the failure. A capability I haven't seen break yet is a capability I don't actually understand.

Next up is the half I've been circling the whole time, and the one where failing in a real room stops being cheap: getting this off the bench and onto something that flies.
