---
title: "Honest Router Numbers: Measuring a Gateway Without Lying"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [openwrt, vpn, ipsec, wireguard, failover, benchmarking]
summary: "On our gateway, IPsec and WireGuard push almost the same throughput — but one uses 1.54 cores and the other 2.51. And our first IPsec 'success' returned pings in 0.74 ms while the SA counters read zero bytes: the tunnel wasn't encrypting anything. After that day, no number got recorded until a counter proved the traffic actually took the path."
---

## Why Router Spec Numbers Lose People's Trust

We measured every performance figure destined for the datasheet of our 5G
FWA routers, the LS500X and NM500G (Qualcomm IPQ8072, based on OpenWrt
25.12). Before starting, we wrote down the two ways router specs usually
get inflated.

1. **Measuring on the device.** Run the traffic generator on the router
   itself and the kernel's local-endpoint optimizations kick in, producing
   numbers far above reality. But a router's job in the field is to forward
   *other people's* traffic.
2. **Calling "connected" a success.** The ping comes back, the status page
   is green, done.

So we set two rules:

> ① Every number is measured on the **forwarded path**, through the device.
> ② A number is only recorded after a **counter proves** the traffic
> actually took that path.

Rule ① is common sense. Rule ② was born in the middle of this campaign —
the story of why is below. Every measurement shares the same conditions:
**single stream, 20 seconds, median of 3 runs.**

## Wired NAT: Line Rate in Software, and What It Costs

| Path | Up | Down | Gateway CPU (upstream) |
|---|---|---|---|
| Plain NAT | **896** | **938** Mbit/s | 2.28 of 4 cores |

We measured a control alongside: take the router out and connect the hosts
directly, and you get 929 Mbit/s. The cost of going through the box is not
measurable. Adding streams — four instead of one — doesn't raise the
number, which is the classic signature of the gigabit port being the
limit, not the box.

The footnote we owe you: this line rate comes from **software forwarding**,
and it pays for itself with roughly two of the four cores. Getting those
two cores back via hardware offloading is still an open task.

## VPN: The Interesting Result Wasn't Speed. It Was CPU.

| Tunnel | Up | Down | Gateway CPU (upstream) |
|---|---|---|---|
| WireGuard (ChaCha20-Poly1305) | **529** | 461 | **2.51** of 4 cores |
| IPsec (IKEv2, ESP AES-GCM) | **520** | 372 | **1.54** of 4 cores |
| OpenVPN / UDP (AES-256-GCM) | 223 | 169 | 1.34 |
| OpenVPN / TCP | 157 | 135 | 1.27 |

### IPsec Delivers the Same Speed on 40% Less CPU

The first two rows are the headline. Upstream throughput is effectively
identical — 529 versus 520 — but WireGuard takes 2.51 cores where IPsec
takes 1.54. The reason is whether the cipher is accelerated by this SoC's
ARM64 cryptography extensions. AES-GCM is (5.4 Gbps raw);
ChaCha20-Poly1305 is not.

This difference doesn't show up as *speed*. It shows up as **headroom**.
On a box that already spends two cores on forwarding, whether the tunnel
eats 1.5 cores or 2.5 decides whether anything else can still run. On a
spec sheet the two rows look interchangeable; in practice, one of them
consumes the box.

### OpenVPN's Ceiling Is One Core

When OpenVPN is the bottleneck, total device CPU sits near 1.4 cores.
That looks like headroom, but it isn't. Inspecting the single process
showed it pushing 217 Mbit/s while using 91% of the only core available
to it. Two footnotes fall out of this:

- Making the forwarding path faster won't raise the OpenVPN numbers. That
  work touches NAT's two cores, not this one core.
- Conversely, OpenVPN is the cheapest tunnel to run *alongside other
  work*: it hammers one core and leaves the rest alone.

So we position OpenVPN by **reachability**, not speed. Its TCP transport
is 30% slower than UDP, but on networks that block ESP and non-web UDP
outright, it is the only tunnel left standing.

### The Twist: Our First IPsec "Success" Was Plaintext

This is where rule ② comes from.

In the first IPsec measurement, pings returned in 0.74 ms — faster than
WireGuard. The security association read `ESTABLISHED`, the child SA
`INSTALLED`, every status light green. But the SA counters said:

```
in 0 bytes / out 0 bytes
```

The packets weren't entering the tunnel at all; they were going straight
to the neighboring device. The firewall's address translation was
rewriting the source address *before* the encryption policy lookup, so
the policy never matched. **"Huh, that's fast" was actually "huh, that's
not encrypted."** No status light distinguishes those two.

From then on, every number was gated on counters:

| Tunnel | Path proof |
|---|---|
| WireGuard | Interface TX counter increments across 5 pings |
| IPsec | ESP byte count grows 672 MB → 4.76 GB during the run |
| OpenVPN | Daemon's own status file counts 92 MB, matching data moved |

### The Numbers You Must Not Quote — We'll Say It First

Measured *on* the device, WireGuard does 840/889 Mbit/s, because the
kernel can use segmentation offload on locally generated traffic and
there's no forwarding. That is not what a site-to-site gateway does, so
**we do not use those numbers.** The forwarded-path figures in the table
above are the honest ones.

## WAN Failover: From 51.30 Seconds to 9.58

The core FWA feature: the wired WAN dies, the box moves to 5G. Here, too,
how you measure decides the number.

| Scenario | Meaning | Measured (median) |
|---|---|---|
| Blackhole | Upstream silently dies, link stays up | **9.58 s** |
| Link down | Someone pulls the cable | 0.85 s |
| Recovery | Wired WAN comes back | 0.00 s |

**The datasheet number is 9.58 seconds.** 0.85 s is what happens when
someone unplugs a cable — not when the upstream dies — and no datasheet
should call that the failover time. The 0.00 s recovery also needs a
narrow reading: it means no observable gap in an ICMP stream; the session
view was not separately measured.

The initial measurement was 51.30 seconds, and the cause was simply the
default timers: a 10-second probe interval times 5 failures is 50
seconds, and the measured value is that plus probe spacing and round
trips. Since the model predicted, tuning behaved as predicted: 3 seconds
times 3 failures gave 9.58 s — and that setting also had the smallest
variance. With a footnote attached: "the 3-second interval doesn't flap"
is based on nine minutes of idle observation and predicts nothing about a
customer's congested or lossy line. What guards against that case is the
voting rule — the line is only declared dead when 2 of 4 probe
destinations fail simultaneously.

### One Missing Setting Meant Failover Never Happened At All

Measured on the same board, minutes apart:

| Connection-tracking flush setting | Blackhole failover |
|---|---|
| Unset (as shipped) | **>120 s — never recovered** |
| Set | 50.15 s |
| Unset again (control) | >120 s |

We pinned the cause by reading the source, not by guessing. The failover
manager marks connections with their WAN and *restores* that mark onto
existing connections — and when the flush list is empty, it flushes
nothing. So established flows keep the dead line's mark forever. The
route changes; the traffic doesn't follow.

### And Then We Narrowed Our Own Finding

At this point you want to write "enable the flush and sessions survive
failover." We re-measured with TCP, and no:

| | Flush set | Unset (control) |
|---|---|---|
| New connections work again after | 8.5–10.9 s | 9.5–10.9 s |
| Connections held across failover | Stall, no response | Stall, no response |

For TCP applications that reconnect, this setting changes nothing — a new
connection is a new 5-tuple and gets marked with the live line anyway. So
the setting's real value is narrower:

- It saves flows with a **fixed 5-tuple** — UDP VPN tunnels (WireGuard,
  IPsec) and long-lived UDP sessions. Without it, those never fail over
  at all.
- **"TCP sessions survive failover" is false.** Do not sell it.

We shipped the setting anyway, for a clear reason: Ethernet+5G failover
sells into small businesses, and the site-to-site VPN running on top is
exactly that fixed-5-tuple UDP flow. We also wrote down the cost — the
flush is global, so at the moment of failover it also cuts a LAN client's
unrelated download.

One more footnote. However fast the failover, **the public IP changes.**
The box moves from the wired WAN to the mobile network's CGNAT, so any
upstream service doing IP-based authentication is affected regardless of
timing. Leave that sentence out and the 9.58 s becomes a lie.

## Wi-Fi Mesh Backhaul: Path Proven Here Too

| Item | Measured |
|---|---|
| Node joins the gateway as a peer | 5.1 s (the UI conservatively says 25 s) |
| Backhaul throughput | 723 / 664 Mbit/s |
| Latency | 3.21 / 6.31 ms |
| Teardown | 3.2 s |

Both units were *also* wired to the same host, so these numbers could
have traveled over Ethernet. We proved the path by watching the wireless
interface counters increment before recording anything. Rule ② again.

Two footnotes: the units sat about 1 meter apart, so this is a
**best-case** figure, and client roaming was not measured (there were no
Wi-Fi clients on the bench). 723 Mbit/s is not a deployment number.

## Sidebar: The Instrument Lied

To assess failover stability we counted how many times the status output
changed: 35 changes in 540 seconds. It was noise. The output contained an
uptime string, so it differed on every poll. The meaningful counter was
the online-time value that resets to zero on a flap — and it never reset
once. The line was fine; the thing flapping was our measurement script.

## The Number Card

All measured. Single stream, 20 seconds, median of 3.

| Item | Value |
|---|---|
| NAT | 896 / 938 Mbit/s (2.28 cores) · direct-connect control 929 |
| WireGuard | 529 / 461 (2.51 cores) |
| IPsec (AES-GCM) | 520 / 372 (1.54 cores — 40% less CPU per bit) |
| OpenVPN UDP / TCP | 223 / 169 · 157 / 135 (single-threaded, 91% of one core) |
| WAN failover (blackhole) | 9.58 s (51.30 s before tuning) |
| Mesh backhaul (1 m) | 723 / 664 Mbit/s, join in 5.1 s |

## What We Keep

- "It's connected" and "the traffic took that path" are different claims.
  If no counter moved, nothing happened. A green status light is not
  evidence.
- A number only becomes a value once its conditions are attached: 9.58 s
  carries "blackhole," 0.85 s carries "cable pulled," 723 Mbit/s carries
  "1 meter apart." A number without conditions *is* an inflated number.
- Saying first which numbers must not be quoted is what buys trust. If
  you can explain why you discard 840/889 and publish 529, the rest of
  your numbers become believable.
- Narrow your own findings yourself. The flush setting rescued failover
  but does not rescue TCP sessions — omit that one line and a finding
  turns into an exaggeration.
