---
title: "Who Is This: Teaching the Camera to Recognize Me"
description: "Following a box is easy. Following a specific person, even after they walk out of frame and come back, is a different problem. Here's re-identification on a Jetson, the weights that silently sabotaged it, and the bug that made a 100ms job take two minutes."
pubDate: 'Jul 26 2026'
---

Last post ended on an honest limitation. The gimbal would happily follow a person around the room, but it wasn't following *a person*. It was following a numbered box. Walk out of frame, walk back in, and you're a stranger with a new number. The camera had no idea you were the same person it had been glued to ten seconds earlier.

That's the difference between tracking and recognition, and it turns out to be most of the difficulty.

This post is how I closed that gap: upload a photo of somebody, and the camera finds them, locks on, and re-locks if they leave and come back. It mostly works. It also taught me more than anything else in this build, including one bug that had me convinced the whole thing had deadlocked.

## What re-identification actually is

The trick is to stop thinking about pixels and start thinking about a fingerprint.

You run a crop of a person through a neural network, and instead of a label, it hands back a list of numbers. A few hundred of them. That list is an *embedding*, and the whole point is that the numbers describe what makes this person look like this person: the shape of them, the color of their clothes, how they carry themselves. Two photos of me should produce two similar lists. A photo of me and a photo of you should produce two different lists.

Once you have that, recognition becomes arithmetic. Store the numbers from a reference photo. When a new person shows up in frame, compute their numbers and measure how close the two lists are. Close enough, same person. That closeness measure is cosine similarity, and it comes out as a score between 0 and 1.

So "do I recognize you" reduces to "is this number big enough." Which is satisfying, and also exactly where the problems start.

## The weights that quietly ruined everything

The model I'm using is OSNet, which was designed specifically for person re-identification. I pulled it in through a library called torchreid, wrote the pipeline, ran it, and got scores that were complete garbage. Everybody matched everybody. The numbers hovered in the same narrow band no matter who was in frame. I could hold up a photo of myself and a chair and get similar answers.

The cause is genuinely nasty, and I want to write it down because I lost real time to it.

When you ask torchreid for an OSNet model, it will happily hand you one, pretrained, no complaints. But by default those weights are trained on ImageNet, which is a general object-classification dataset. That network learned to tell a dog from a truck from a coffee cup. It never learned to tell one person from another person, because that was never the task. So it produces embeddings that describe "this is a human-shaped thing," which every single person in frame satisfies equally well. Turns out a gym bag on a couch also resembles a human in the eyes of the OSNet model.  

What you actually need is the same architecture trained on a person re-identification dataset. I used the Market1501-trained checkpoint, downloaded separately and passed in explicitly. Same model, same code, completely different behavior. Suddenly the scores separated: the right person came back around 0.7 to 0.9, strangers sat well below.

The lesson isn't about torchreid. It's that "pretrained" is not a property, it's a question. Pretrained *on what*, for *what task*. A model that loads without error and returns plausible-looking numbers can still be answering a question you never asked, and nothing in the code will tell you. The only symptom was that the output was subtly useless.

## Why it can't run every frame

The obvious way to build this is to check every person in every frame. That is also the way to destroy your frame rate.

Object detection on the Jetson already costs most of the budget. I get about 8 frames per second of YOLO on this hardware, and that's the real ceiling of the whole system. Adding a second network that also has to run on every detected person, every frame, would have taken something that was merely slow and made it unusable.

So the re-identification runs as a second, slower loop, and it only runs when there's a reason:

- Nothing is locked yet, so it's looking for somebody to lock onto.
- The lock was just lost, so it's trying to find them again.
- Something is locked and has been for a while, so about once a second it double-checks that the person it's following is still the right person.

That last one matters more than it sounds. Without it, a lock is a leap of faith that never gets re-examined. With it, drifting onto the wrong person gets caught within a second or so.

The split is the useful idea here, and it generalizes way past this project. Fast loop, cheap, every frame, keeps the camera pointed. Slow loop, expensive, occasional, decides *what* to point at. Anything that has to run on tight hardware ends up looking something like this: figure out which decisions genuinely need to happen 30 times a second, and stop pretending the rest do.

## The bug that looked like a deadlock

Here's the one that got me.

Uploading a reference photo should be fast. Decode the image, find the person in it, crop them, run one embedding, store the numbers. Call it a tenth of a second of actual work.

The first time I tested it against the live camera, it took over two minutes.

Not an error. Not a crash. It eventually finished and worked correctly, having taken roughly a thousand times longer than it should have. That's a deeply confusing failure, because a broken thing that returns the right answer very slowly is much harder to reason about than a broken thing that just breaks.

I was fairly sure I'd deadlocked something. Went digging into what the process was actually doing while it sat there, expecting to find two things waiting on each other forever.

It wasn't a deadlock. It was contention, which is the more embarrassing cousin.

Finding the person in the uploaded photo means running the detection model. But the video pipeline is *also* running that same detection model, on every frame, continuously, from a different thread. Two threads, one model instance, both trying to use the same GPU through it at the same time. Nothing crashed and nothing hung, because nothing was actually broken in the "waiting forever" sense. They just got in each other's way so badly that both crawled, and the short job got buried under the long-running one.

The fix is one lock around every call into the model, so only one thing gets to use it at a time. Upload went from two minutes back to feeling instant, and the live pipeline stopped hitching when I uploaded a photo.

The part worth keeping: "this is impossibly slow" and "this is deadlocked" feel identical from the outside, and they have completely different causes. Slow can be a resource being shared badly. Stuck is a circular wait. Guessing which one you have sends you looking in entirely the wrong place, and I guessed wrong first.

## Showing its work

One decision I'm glad I made early: every comparison gets logged with its actual score.

When the camera decides to lock onto somebody, the console shows what it compared, what it scored, and what the threshold was. Same when it decides *not* to lock. So when it does something surprising, I'm reading numbers instead of guessing at vibes.

That mattered immediately. "It won't lock onto me" is a useless bug report. "It scored me 0.58 against a 0.62 threshold" tells you exactly what happened and exactly which knob to turn. And on the flip side, when it locks onto the wrong person, seeing that the wrong person scored 0.71 tells you the threshold isn't the problem, the reference photo is.

Anything that makes a judgment call should be able to show you the arithmetic behind it. Especially when it's going to be pointing a camera, and eventually steering an aircraft.

## Where it breaks

Two honest limitations.

**It's a person model, and only a person model.** OSNet was trained on people, so it recognizes people. It has no useful opinion about my cat. Recognizing a specific pet would mean a different, more general model running alongside this one, and I decided that wasn't worth the complexity for what this project needs.  

**Reflections.** This one's genuinely interesting, and it's the failure mode I'd worry about in the air.

I was testing indoors and the tracker cheerfully found two people: me, and my reflection in the TV. Funny on a bench. Less funny over water, or next to a glass building.

And here's the ugly part: re-identification makes this *worse*, not better. A mirror image of me looks almost exactly like me, so it produces almost exactly the same embedding. The recognition layer, the thing whose entire job is to tell people apart, will confidently confirm that my reflection is me. Every safeguard I have is looking at appearance, and appearance is precisely what a reflection copies perfectly. So follow mode would happily swing the camera toward the water, and none of my checks would object.

The fix I went with uses physics instead of more model, which is why I like it. A reflection is upside down. So when two candidates both clear the threshold, I flip the lower one vertically and score it again. A real person scores worse upside down. A reflection scores *better*, because flipping undoes the mirroring and lines it back up with the upright references. That turns "these two look identical, no idea" into an actual measurement, and it only costs anything in the rare moment the ambiguity actually exists. The rest of the time it never runs.

It works. My reflection in the TV doesn't get a lock anymore. And the part I like best is that it's honest about *why* it works: it isn't a smarter model or a tuned threshold, it's a fact about the physical world (reflections are mirrored) turned into one extra comparison. The same move as the earlier point about knowing why something can't work. Here it's knowing exactly why something can, and building the smallest possible thing that exploits it.

## The failure that no knob fixes

Here's the one I want to be honest about, because it's the most important thing I learned.

A roommate walked through the room while I had a lock on myself. Bad lighting, both of us basically silhouettes, similar build, both in dark clothing. I walked out one door, the camera did its thing and reset to search, he walked in another door, and it locked onto him. Then the reverse: he walked out, I walked back in, and it grabbed me. It could not tell us apart.

My first instinct was that the match threshold was too low. So I pulled the actual scores it logged, and that's where it got interesting. Every comparison logs a number between 0 and 1, how close the person in frame is to the stored reference. When it was me, I scored things like 0.79, 0.89, 0.93. When it was him, he scored 0.77, 0.82, 0.87, 0.91. Look at those two lists. They completely overlap. There is no number you can draw a line at that keeps me and drops him, because on any given frame he might score higher than I do.

That's the whole lesson. The threshold wasn't wrong. The *input* had no answer in it. Two silhouettes with the same build and the same dark clothing produce nearly the same fingerprint, because the fingerprint is built almost entirely from clothing color and texture and body shape, and bad light erases all three. The model wasn't failing. I'd handed it an image with the identifying information already removed and asked it to identify us anyway.

No amount of tuning fixes that. Raising the threshold would just make it refuse to lock onto *anyone*, me included. The fix isn't in the code at all. It's a light switch. Or an uploaded reference photo taken in decent light, so there's real texture to match against instead of silhouette-versus-silhouette.

There was also a nastier second effect hiding in there. The system keeps a small rolling gallery of the locked person, refreshed every couple of seconds while they're in view, so it can recognize them across different poses. But the moment it locked onto the wrong person, it started refreshing that gallery with *him*. So it was actively learning the wrong identity, cementing its own mistake, getting more confident about exactly the thing it had gotten wrong. I now freeze that gallery once it's built, so a bad lock can't poison it further. Doesn't fix the underlying "these two are identical to the sensor" problem, but at least the system stops digging.

The reason I'm dwelling on this: the useful skill wasn't building the re-identification. It was reading the score data and recognizing that the problem was upstream of anything I could code. Knowing *why* something can't work, with the numbers to prove it, is worth more than a clever hack that papers over it and fails in the field when you're trusting it. A drone that confidently follows the wrong person is worse than one that admits it lost you.

## What this changed

Before this, the camera followed a box and forgot you the moment you left. Now it follows *you*, and it can find you again.

That upgrade is what the whole re-identification detour bought, and standing in the room watching the gimbal swing back onto me specifically, after I'd walked out and come back, was the best moment of this build so far. Better than the first time it tracked anything at all, because this time it wasn't just seeing motion. It was making a decision about identity, on a little board bolted to a bench, in real time.

Next up, the harder half: getting this off the bench and onto something that flies.
