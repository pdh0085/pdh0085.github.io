---
title: "The Port Forwarding Bug That Wasn't: A Silent MTU Failure on an LTE Link"
date: 2026-08-16T15:30:00+09:00
draft: true
tags: ["lte", "networking", "mtu", "debugging", "port-forwarding"]
summary: "External access to an IP camera behind our LTE router kept dying while the router's own web UI worked fine from outside. The port forwarding rules were innocent: large TCP segments were coming off the LTE link damaged, and forcing the WAN MTU down to 1400 made the whole thing work."
---

This one is from a while back, so the exact logs are long gone — commands and
numbers below are reconstructed. The debugging sequence and the wrong turns,
however, I remember vividly. You tend to remember the ones that make you feel
stupid for a week.

## The Setup

A customer was deploying our LTE router (the LTE572W, one of our 4G CPE
products) as a relay for IP cameras: camera on the router's LAN
(192.168.10.0/24), LTE uplink to the carrier, and port forwards so their
operations team could reach each camera's web UI from the office.

Simple. The kind of setup you configure in five minutes and never think about
again.

## The Symptom

The complaint came in as: *"port forwarding is broken — we can't reach the
camera's web UI from outside."*

And it looked plausible, because of one detail that sent me down the wrong
road for a while: **the router's own web UI was perfectly reachable from
outside.** WAN was up, the LTE link was fine, remote access worked — so the
box itself was healthy, and the finger naturally pointed at the forwarding
rules.

I added another rule to test with something simpler than HTTP: external port
8022 forwarded to the camera's SSH on 22. Same story. The SSH client just sat
there and died. Meanwhile, from the LAN side, everything about the camera was
fine — web UI, SSH, all of it. Camera healthy, router healthy, rules
apparently correct, external access dead.

## First Suspects (All Wrong)

The usual checklist: stared at the firewall and DNAT rules until my eyes hurt,
double-checked the camera's default gateway, re-created the forwards,
rebooted things that didn't need rebooting. Everything was exactly as it
should be. That's the worst kind of bug — nothing is misconfigured, and it
still doesn't work.

## The First Real Clue: TX Climbs, RX Stays Flat

The turning point was low-tech. My "external client" was a company PC in our
Taiwan office, which I drove over remote desktop from my desk in Korea — so
I could hammer the router's public address and watch the interface packet
counters on the camera-facing port at the same time. One person, two vantage
points, two countries apart.

Every connection attempt from outside, the **TX counter toward the camera
kept climbing. RX stayed flat.**

That one observation flipped the whole picture. The port forwarding was
working — the router was translating and delivering packets to the camera the
entire time. The camera was receiving *something* and refusing to answer. My
working theory became: whatever the camera is receiving, it's arriving in a
state the camera's TCP stack won't accept, so it silently drops it.

## tcpdump: The Packets Were Arriving Damaged

I installed tcpdump on the router and captured the forwarded traffic. Small
packets — TCP handshake and friends — looked clean. But the large segments
carrying the actual data were wrong: the TCP payload was damaged, cut short
partway through, like something along the LTE path had taken a bite out of
them.

That also explained the silence perfectly. A router doing NAT doesn't
validate TCP checksums — it rewrites what it needs and forwards the rest as
is. So the damaged segments sailed right through to the camera, the camera's
TCP stack dropped them as corrupt, no ACK ever went back, and the sender just
kept retransmitting into the void. TX climbs, RX flat.

And in hindsight, the red herring explains itself too: the router's own web
UI survived because its exchanges happened to stay small enough to slip under
the damage threshold. A reachable box tells you the path passes *small*
packets. It tells you nothing about full-size ones.

## The Fix (The Immediate One)

If large packets die and small ones live, you cap the packet size. I forced
the WAN MTU on the modem interface from 1500 down to **1400** and re-tested.

Everything worked. Web UI, SSH, video — instantly.

I won't pretend I can reconstruct the exact mechanism from memory — MSS
following the interface MTU down for forwarded connections would be the
usual suspect — but the observable result was unambiguous: cap the WAN at
1400 and nothing on the link gets bitten.

These days, my first probe for this class of problem is a DF-bit ping sweep
to find the real path ceiling before touching anything:

```bash
# Reconstructed - what I'd run today, from a host behind the router
ping -M do -s 1372 <target>   # 1372 + 28 = 1400 bytes: passes
ping -M do -s 1472 <target>   # 1500 bytes: dies silently, no ICMP back
```

## Root Cause: Officially, I Still Don't Know

Here's the part that doesn't make it into tidy postmortems: I never got a
definitive answer on *what* was damaging those packets.

The evidence says the LTE path could not deliver ~1500-byte packets intact,
and nothing on the path signaled it — no ICMP, no error, just quiet damage.
Whether that lived in the modem firmware or somewhere in the carrier's
network, I can't prove.

What I can tell you is that I'd seen this movie before. Back in the early 3G
days we hit a very similar large-packet problem, and that one *was*
eventually confirmed as a modem firmware bug. That modem came from a small
vendor, so we worked the issue together — we characterized it, they fixed
the firmware, done. This time the modem came from a tier-1 vendor, and
realistically a single field case from one CPE maker was not going to open a
firmware investigation. Different vendor size, different physics.

## The Real Fix: Stop Trusting the Modem's MTU

The lasting fix wasn't the number 1400. It was a product change: our firmware
used to take the MTU the modem reported and run with it. After this case, we
added a **manual MTU override** — if the field says the path can't carry what
the modem claims, an operator can pin the WAN MTU themselves, per deployment,
without waiting for anyone's firmware.

One field incident, one permanent knob. That knob has paid for itself since.

## Takeaways

1. **"Small packets work, large packets die" is the MTU smell.** Partial
   connectivity — reachable UI, hanging transfers, SSH that connects and
   stalls — should make you think packet size before configuration.
2. **Interface counters are a free diagnostic.** TX climbing while RX stays
   flat told me more in five minutes than hours of rule-staring: the
   forwarding path was fine and the answers were never coming back.
3. **A reachable device is not a healthy path.** The router's own UI working
   from outside was the most misleading fact in the whole case.
4. **On cellular, don't assume the path delivers 1500 — or that it will tell
   you when it doesn't.** Design as if silent large-packet loss is a thing,
   because it is.
5. **You won't always get root cause. Ship the mitigation anyway.** Between a
   tier-1 vendor's queue and a customer's deadline, a well-designed override
   in your own firmware wins. Make the fix a product feature, not a war
   story.
