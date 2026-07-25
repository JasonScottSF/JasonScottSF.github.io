---
title: 'Protecting the Control Link: RF Coexistence on a Multi-Radio Drone'
description: "A drone is a flying RF coexistence problem, and one of the radios is not allowed to fail. Here's how I split up the spectrum so the link that matters never has to fight for air."
pubDate: 'Jul 25 2026'
---

Day job, I do enterprise Wi-Fi. "Coexistence" there means keeping a few dozen access points and a pile of client devices from stepping on each other across 2.4 and 5 GHz. When it goes sideways you get a retransmit, a slow session, a cranky user. Annoying, sure, but nobody gets hurt.

Then I started building a drone that flies itself, and coexistence got a lot more literal.

On an aircraft you've got several radios crammed into a few square inches, and one of them - the manual control link - is not allowed to fail. If it drops at the wrong second, the thing falls out of the sky. That turns spectrum planning from "optimize the network" into "don't put a drone through somebody's window," which is a different kind of problem. The good news is the actual discipline is the same one I use at work. So here's how I split up the bands, and more to the point, how I reasoned about it, because the thinking works on anything with more than one radio bolted to it.

## The radios

Three RF links on the build, each doing its own job:

- **2.4 GHz - manual control (ELRS).** ExpressLRS from a hand controller to the flight controller. This is the one that matters. It's how a human grabs the wheel. Non-negotiable.
- **915 MHz - telemetry (SiK radio).** MAVLink back to a ground station running QGroundControl. Long range, barely any bandwidth. Attitude, position, battery, mode.
- **5 GHz - companion link (Wi-Fi).** The onboard Linux computer's Wi-Fi, throwing a live view and telemetry out to a tablet in the field.

There's a fourth heavy stream, the gimbal camera video, but I kept that on **wired Ethernet** on purpose. That's a coexistence decision too, and I'll get to it.

## The one rule everything hangs on

Here's the whole thing in a sentence:

> **The control link is the must-have. Everything else is expendable. Never let an expendable link share a band with the safety-critical one.**

At work I protect the important traffic with QoS, band steering, careful channel plans. On the drone the move is blunter: physically wall off the control link's band and keep everything else out of it. A dropped video feed is a shrug. A desensed control receiver is a crash. Those aren't the same thing, so I don't treat them the same.

## The calls I made

**2.4 GHz is control's, and control's alone.** ELRS owns 2.4, and nothing else on the aircraft transmits there. One wrinkle worth knowing: ELRS is frequency-hopping, so it smears across the entire 2.4 band by design. That trips people up, because it means **you can't channel-plan around it.** There's no quiet corner to tuck a second radio into. Over time, the hopper is everywhere. Your only real move is to keep other 2.4 GHz transmitters off the aircraft, full stop.

**The trap I almost walked into: "just put the Wi-Fi on 2.4 for range."** That was my first instinct for the companion link, since 2.4 reaches further and punches through walls better than 5. Wrong call, for three reasons that pile up:

1. **It's in the control link's band.** Breaks the one rule right out of the gate.
2. **They're inches apart.** The Wi-Fi radio and the ELRS receiver are bolted to the same little airframe. A Wi-Fi transmitter that close is a loud in-band interferer sitting right on top of a sensitive receiver. Textbook desense. Frequency separation on a spreadsheet doesn't save you when the offender is zip-tied next to the victim's antenna.
3. **The range doesn't even matter.** The companion Wi-Fi is a close-in convenience link. The long-haul job belongs to the 915 MHz radio. So I'd be handing real interference risk to the *critical* link to buy range on a link that doesn't need any.

So the companion Wi-Fi went to **5 GHz**, out of the control link's hair entirely.

**Pin the 5 GHz Wi-Fi to a non-DFS channel.** 5 GHz has a nasty little trap for anything real-time: DFS channels have to bail the second they think they hear radar, and that can happen *mid-flight*. A link riding shotgun to my control setup that just vanishes for 60 seconds because a weather radar swept past? Hard no. So I locked the AP to a **fixed non-DFS channel** (36/40/44/48 in the US). No radar-avoidance surprises. Easy to ignore indoors. Expensive to ignore at altitude.

**915 MHz does the long haul.** The SiK radio on 915 is the one that actually needs distance, and it's nowhere near 2.4 or 5. Low bandwidth, long reach. Exactly what MAVLink wants.

**Send the bandwidth hog down a wire.** The camera video is the fattest stream in the whole system. Put it on the air and I'm either burning spectrum or compressing it to mush. Running it over wired Ethernet to the companion computer takes my single biggest RF consumer *off the air completely*. The cleanest interference is the transmission you never make.

## What actually ends up on the airframe

| Band | Link | Role | Why it's here |
|------|------|------|---------------|
| 2.4 GHz | ELRS | Manual control | Safety-critical, kept uncontested |
| 915 MHz | SiK | Telemetry | Long range, low bandwidth |
| 5 GHz | Wi-Fi | Companion / field view | Close-in, off the control band, non-DFS |
| *(wired)* | Ethernet | Camera video | Biggest stream, kept off the air |

Three clean bands, zero overlap, and the one link that can't fail has its spectrum to itself.

## The part that isn't really about drones

Take the drone out of it and this is just RF engineering, which is exactly why it maps straight back to the day job:

- **Separate by priority, not just frequency.** Figure out what traffic is critical, give it uncontested spectrum, make everything else plan around it.
- **Physical separation counts as much as the channel plan.** Co-located radios desense each other no matter what your spreadsheet says. Antenna placement and isolation matter every bit as much as which channel you picked. Anybody who's fought a co-located BLE or Zigbee radio already knows this one in their bones.
- **Watch out for channels that can disappear.** DFS is fine for best-effort. It's a hazard for anything you're leaning on in real time. Know which of your channels can get evicted.
- **You can't out-plan a frequency hopper.** FHSS gear owns its whole band over time. Living with it is about isolation and separation, not channel selection.
- **The best interference fix is not transmitting.** Moving the heaviest stream to wire didn't manage a coexistence problem, it deleted one.

None of this is exotic. It's the same physical-layer discipline that makes an enterprise deployment boring and reliable: decide what matters, protect it on purpose, and respect the fact that RF is a shared physical medium that does not care one bit about your logical design. The drone just swaps the consequence from "slow Wi-Fi" to "controlled flight," which turns out to be a really effective way to start taking the physical layer seriously.

*This is the first in a series on an AI-piloted drone build - the RF, the edge AI, and the flight-control guts behind it. More coming.*
