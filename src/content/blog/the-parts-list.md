---
title: "The Parts List (and the Ones That Didn't Make It)"
description: "Everything currently bolted to the drone, plus a graveyard of the parts I tried, considered, or almost bought before backing away slowly."
pubDate: 'Jul 9 2026'
---

People always want the parts list. Fair enough, half the fun of a build like this is arguing about gear. So here's everything that's actually on the drone right now, followed by the more interesting list: the stuff I tried, considered, or almost bought before backing away slowly.

Fair warning, this is a living document. A build like this is never really "done," it's just "flying today." I'll update it as things change, and things will change.

## What's actually on the drone

### Airframe and propulsion

The frame is a **Holybro S500 V2**. It's a 500mm-class quad, carbon and aluminum, with a big flat lower deck that's practically begging you to hang payload off it. (There's a story about how I ended up on the S500 instead of the X500. It's in the graveyard below.)

Propulsion is the S500 V2 stock kit: four brushless motors, four ESCs, and 10-inch props. Stable over fast, which is exactly the point. I'm not racing anything. I want a platform that goes where I point it and hovers like it's bored.

| Part | What it is |
|------|-----------|
| Holybro S500 V2 | Frame, landing gear, power distribution board |
| Stock motors (x4) | 2216-class brushless |
| Stock ESCs (x4) | 20A BLHeli_S |
| 1045 props (x4) | 10 inch, stable not fast |
| 4S LiPo | Main flight battery |

### Flight control and navigation

The brain that actually flies is a **Pixhawk 6C** running ArduPilot. Everything safety-critical lives here (I've got a whole rant about why the flaky Linux box does not get to make flight decisions).

| Part | Role |
|------|------|
| Holybro Pixhawk 6C | Flight controller (ArduPilot) |
| Holybro M10 GPS/compass | Position and heading |
| PMW3901 optical flow | Downward-facing, position hold without GPS |
| TF-Luna lidar (x4) | Obstacle avoidance, feeding the flight controller directly |

### Companion computer and vision

This is where the "AI" in "AI-piloted" actually lives.

| Part | Role |
|------|------|
| NVIDIA Jetson Orin Nano (dev kit) | Edge AI: detection, tracking, the dashboard |
| SIYI A8 mini | 3-axis gimbal camera, video over Ethernet |
| Onboard RTL8822CE Wi-Fi | Companion link out to a tablet |

The Jetson does the seeing and thinking. The A8 mini does the looking and pointing. They talk over wired Ethernet, which, as I've written about elsewhere, is itself a deliberate RF decision.

### Radios

Three links, three bands, and a [whole article about why](/blog/rf-coexistence-on-a-drone/).

| Link | Gear | Band |
|------|------|------|
| Manual control | RadioMaster Boxer + ELRS | 2.4 GHz |
| Telemetry | Holybro SiK radio | 915 MHz |
| Companion / field view | Jetson Wi-Fi | 5 GHz |

Plus a Remote ID module, because the FAA says so and I'd like to keep flying.

### On order

- **Ubiquiti USW-Flex-Mini.** A tiny Ethernet switch for the day I want the Jetson AI and a video controller live at the same time. It hurt my soul to buy Ubiquiti, but nobody makes a 46-gram enterprise switch, so here we are.

## The parts graveyard

This is the good stuff. Every one of these was a real consideration, and every one taught me something.

**Holybro X500 V2, the frame I actually wanted.** The X500 has taller landing gear and folding arms, which is genuinely nicer for hanging a gimbal underneath. It was also sold out absolutely everywhere. Pivoted to the S500, which is a fine frame, just shorter in the legs. Which, it turns out, matters a lot when you're bolting an 85mm camera to the belly. Still measuring.

**Arducam AR0234 global shutter camera (B0579).** This was going to be the vision camera. Nice global-shutter CSI module, great for fast motion. It turned into a saga. No driver existed for the JetPack version I was on, so I reflashed the whole Jetson back a major version to get one. Then the driver installed but the camera gave me a solid green frame, which turned out to be the wrong CSI port. Got it detected, got a real image out of it, and then decided the A8 mini gimbal was the better camera path anyway, since it hands me clean RTSP video over Ethernet with none of the CSI driver fragility. The Arducam's on the bench. Maybe it comes back as a fixed second camera someday.

**Intel RealSense D435i / D455.** The depth-camera plan for obstacle avoidance, back before I committed to lidar. Superseded by four TF-Luna lidars feeding the flight controller directly: simpler, lighter, and I already owned the Lunas. The RealSense stays on the shelf as a fallback if the lidar coverage turns out too thin.

**The RealSense clone I almost bought.** There's a depth camera out there that looks like a RealSense and costs about $100 less. I talked myself out of it. Cross-vendor "it's basically the same thing" integration is exactly where builds go to die, and I'd already lived that once with the Arducam. Not worth the savings.

**SIYI's onboard AI tracking.** The A8 mini has its own built-in AI tracker. I'm not using it. It's a closed, fixed-function thing that recognizes three classes, needs a separate module to unlock, and SIYI's own docs call parts of it "under development." My Jetson runs its own tracking that I actually control and can extend. Our AI is better.

**A giant battery for more flight time.** The obvious move for endurance is "bigger battery." I ran the numbers on a 10,000mAh pack and it buys almost nothing, maybe 14 minutes, because the extra weight fights both the hover thrust and the motor efficiency. Real endurance isn't a bigger LiPo on this frame, it's a different design point entirely: lithium-ion cells and a bigger, slower, hover-optimized airframe. That's a future build, not an upgrade.

**Pulling video off DJI Goggles.** I've got DJI FPV gear and briefly wanted to yank its video feed onto an external display. The Goggles 3 have no HDMI out, and DJI's only official path needs a controller I don't own. Found a third-party adapter, didn't trust it, dropped the whole idea as out of scope.

## The one decision still coming

The Gen-2 plan is to replace the Boxer-plus-laptop-plus-tablet juggling act with a single integrated controller, something like a SIYI MK15 or MK32 that does control, telemetry, and video in one handheld unit. I've priced out the whole field (Herelink, Skydroid, the SIYI lineup) and I'm leaning SIYI, mostly because it plugs straight into the A8 mini I already own with no cross-brand guesswork. But that's a "get it flying first" problem. The Boxer works today.
