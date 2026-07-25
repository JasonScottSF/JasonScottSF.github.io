---
title: 'Protecting the Control Link: RF Coexistence on a Multi-Radio Drone'
description: 'A drone is a flying RF coexistence problem. Here is how I planned the spectrum on an AI-piloted build so the one radio that matters most never has to fight for air.'
pubDate: 'Jul 25 2026'
---

Most of my work lives in enterprise Wi-Fi, where "coexistence" means keeping a few dozen access points and a swarm of client devices from stepping on each other in the 2.4 and 5 GHz bands. The stakes are real but recoverable: a retransmit, a slower session, an annoyed user.

Then I started building an AI-piloted drone, and coexistence got a lot more literal. On an aircraft, several radios share a few square inches of airframe, and one of them — the manual control link — cannot be allowed to fail. If it drops at the wrong moment, the thing falls out of the sky. That reframes spectrum planning from an optimization problem into a safety problem, and it turns out the discipline transfers almost directly from the day job. This post walks through how I allocated the bands, and — more usefully — the *reasoning* that got me there, because the framework is portable to any multi-radio system.

## The cast of radios

The build carries three independent RF links, each with a different job:

- **2.4 GHz — manual control (ELRS).** ExpressLRS between a hand controller and the flight controller. This is the safety-critical link: it's how a human takes over. It is the must-have.
- **915 MHz — telemetry (SiK radio).** A MAVLink link down to a ground station running QGroundControl. Long range, low bandwidth — attitude, position, battery, mode.
- **5 GHz — companion link (Wi-Fi).** The onboard Linux companion computer's Wi-Fi, bridging a live view and telemetry to a tablet in the field.

There's a fourth high-bandwidth stream — the gimbal camera's video — but I deliberately kept that on **wired Ethernet** between the camera and the companion computer. More on why that's a coexistence decision in itself below.

## The one principle that drove every choice

Here it is, and everything else falls out of it:

> **The control link is the must-have. Every other link is expendable. Never let an expendable link share a band with the safety-critical one.**

In enterprise wireless we protect critical traffic with QoS, band steering, and careful channel planning. On the drone the equivalent move is blunter and more important: physically reserve the control link's band and keep everything else off it. A dropped video feed is an inconvenience. A desensed control receiver is a crash.

## The decisions

**2.4 GHz belongs to control, uncontested.** ELRS is the control link, so 2.4 GHz is reserved for it and nothing else on the aircraft transmits there. There's a wrinkle that matters: ELRS is a **frequency-hopping (FHSS)** system — it spreads across the entire 2.4 GHz band by design. That has a direct consequence most people miss: **you cannot channel-plan around a frequency hopper.** There's no "quiet corner" of 2.4 to tuck another radio into, because the hopper is, over time, everywhere. Your only real tool is to keep other 2.4 GHz emitters off the aircraft entirely.

**The tempting mistake: "just put the Wi-Fi on 2.4 for range."** My first instinct for the companion link was 2.4 GHz — better range and wall penetration than 5. It's the wrong call here, for three reasons that stack:

1. **It shares a band with the control link.** Immediate violation of the one principle.
2. **Physical proximity makes it worse.** The Wi-Fi radio and the ELRS receiver sit inches apart on the airframe. A Wi-Fi transmitter that close is a strong in-band interferer right on top of a sensitive receiver — classic receiver desense / blocking. Frequency separation on paper doesn't save you when the aggressor is bolted next to the victim's antenna.
3. **The range argument is moot anyway.** The companion Wi-Fi is a close-in convenience link; the long-range job belongs to the 915 MHz telemetry radio. So I'd be taking on real interference risk to the *critical* link to buy range on a link that doesn't need it.

So the companion Wi-Fi went to **5 GHz** — a different band, well clear of the control link.

**Pin the 5 GHz Wi-Fi to a non-DFS channel.** 5 GHz has a trap for real-time links: DFS (Dynamic Frequency Selection) channels must vacate if they detect radar, and that eviction can happen *mid-flight*. A control-adjacent telemetry link that can silently drop for 60 seconds because a weather radar swept past is a liability. So I pinned the AP to a **fixed non-DFS channel** (36/40/44/48 in the US) — no radar-avoidance surprises. This is the kind of thing that's easy to ignore indoors and costly to ignore on an aircraft.

**915 MHz carries the long-range telemetry.** The SiK radio on 915 MHz is the link that actually needs distance, and it's comfortably clear of both 2.4 and 5 GHz. Low bandwidth, long reach — exactly right for MAVLink.

**Move the bandwidth hog to wire.** The camera's video is the fattest stream on the system, and putting it on the air would mean either eating spectrum or compressing hard. Running it over wired Ethernet to the companion computer takes the single biggest RF consumer *off the air entirely*. The cleanest interference is the transmission you never make.

## What ends up on the airframe

| Band | Link | Role | Why it's here |
|------|------|------|---------------|
| 2.4 GHz | ELRS | Manual control | Safety-critical; kept uncontested |
| 915 MHz | SiK | Telemetry | Long range, low bandwidth |
| 5 GHz | Wi-Fi | Companion / field view | Close-in, off the control band, non-DFS |
| *(wired)* | Ethernet | Camera video | Biggest stream, kept off the air |

Three clean bands, no overlap, and the one link that can't fail has its spectrum to itself.

## The parts that generalize

Strip away the drone and this is just good RF engineering, which is why it maps so cleanly back to the enterprise world:

- **Separate by priority, not just by frequency.** Decide what traffic is critical and give it uncontested spectrum. Everything else plans around it.
- **Physical separation is a first-class variable.** Co-located radios desense each other regardless of what the channel plan says. Antenna placement and isolation matter as much as frequency selection — a lesson every WLAN engineer who's fought a co-located BLE or Zigbee radio already knows.
- **Beware channels that can disappear.** DFS is fine for best-effort; it's a hazard for links you're depending on in real time. Know which of your channels can be evicted.
- **You can't out-plan a frequency hopper.** FHSS systems occupy their whole band over time. Coexistence with them is about isolation and separation, not channel selection.
- **The best interference mitigation is not transmitting.** Moving the heaviest stream to wire removed an entire coexistence problem instead of managing it.

None of this is exotic. It's the same physical-layer discipline that makes an enterprise deployment reliable — deciding what matters, protecting it deliberately, and respecting the fact that RF is a shared physical medium that doesn't care about your logical design. The drone just raises the stakes from "slow Wi-Fi" to "controlled flight," which is a very effective way to make you take the physical layer seriously.

*This is the first in a series documenting an AI-piloted drone build — the RF, the edge AI, and the flight-control integration behind it. More to come.*
