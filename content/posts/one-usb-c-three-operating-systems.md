---
title: "One USB-C Port, Three Operating Systems"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["usb", "usb-gadget", "embedded", "networking", "performance", "5g"]
summary: "Our 5G CPE had to enumerate as a network device on Windows, macOS, and Linux — driver-free. ECM covered Mac and Linux, RNDIS covered Windows, and putting both in made the hardware refuse. This is the story of finding the one answer that satisfies all three, and of distrusting our own first benchmark: 900 Mbit/s that turned out to be a lie. The real numbers were 315/365 Mbit/s — the practical ceiling of USB 2.0."
---

## The Requirement, in One Line

The requirement on our 5G CPE fit in a single sentence.

> Plug in the USB-C cable and the PC sees a network device. No driver
> installation. Windows, macOS, and Linux.

One sentence — but all of its weight sits in the last five words.

A Linux USB gadget can present itself to a host as a network device in
several different ways, and **the problem is that none of the obvious ones
satisfies all three operating systems.**

| Function | Windows | macOS | Linux |
|---|---|---|---|
| **ECM** (what we had been shipping) | ✗ | ✓ | ✓ |
| **RNDIS** | ✓ | ✗ | ✗ |
| **NCM** | ✓ | ✓ | ✓ |

Looking at the table, the answer seems obvious. And in the end it really
was NCM. What this post is about is the time it took to get there.

## "Just Put In Both"

The first idea anyone has is to expose both ECM and RNDIS and let each host
pick the one it understands.

We tried. The USB controller refused.

```
Out of memory
```

It wasn't memory that ran out — it was **endpoints**. A USB device has a
hardware-fixed budget of communication channels, and this module was already
spending most of its budget on several serial ports and a diagnostic
channel. There was no room for a second network function.

To make room, we would have had to **give up a serial port**. In fact,
looking at the other configuration combinations the module vendor offers,
every one that carries multiple network functions sacrifices serial to do
it. We were using those serial ports for diagnostics and control, so they
were not negotiable.

So we had to **pick exactly one** — and the only one that works on all
three operating systems was NCM.

## The Thing You Must Not Change: the PID

There is a trap here.

A USB device identifies itself by its VID (vendor ID) and PID (product ID),
and host-side drivers are **bound to that combination** — the Linux serial
drivers, the Windows driver installer the vendor distributes, all of them.

Which means: **change the PID and every serial port on the host dies.** You
would be trading your diagnostic path for the network fix.

So what we actually did was **keep the PID and swap only the network
class**. Fortunately NCM needs the same number of endpoints as the ECM it
replaced, so we could slot it into that exact position without disturbing
the rest of the configuration.

## Then Every Module Came In Different

At this point real hardware threw us a problem we hadn't planned for.

These were modem modules of the same model, yet **each incoming unit
arrived with a different USB configuration value.** Any of four values could
be inside, and the four differed in exactly one thing: **the network
function.**

| Value | Network function | Result |
|---|---|---|
| A | no Ethernet class at all | never enumerates as a network device |
| B | MBIM | macOS ✗ |
| C | RNDIS | macOS ✗ |
| **D** | **NCM** | **all three ✓** |

Our fix targeted D and only D. So a unit that arrived as A, B, or C
**never boots into the configuration we fixed, no matter how correct the
code is.**

Two roads from here:

1. Fix all four configurations to NCM.
2. **Converge every unit onto one configuration** at first boot.

We chose the second, because **four PIDs means four of everything on the
host side** — four driver bindings, four sets of documentation, four
support procedures. Converge to one, and every unit in the world looks
**identical** to every host. That is the direction that shrinks support
cost.

So first boot got a normalization step for the configuration value. The
hardware verification was unambiguous: we deliberately put a unit into
state A, flashed the new firmware, and **on its first boot it corrected
itself to D and restored the MAC pinning as well.**

## Names Will Fool You

This work added four automated checks: the configuration value, the
**actually-bound function**, bridge membership, and MAC pinning.

Why the second one exists is the point of this section.

**MBIM also creates a network device named `usb0`.** So a check that only
asks "does `usb0` exist?" will **pass a unit that is unusable on macOS.**

We didn't figure this out by reasoning. We deliberately injected an MBIM
configuration and **watched the check pass.** So the check was changed from
"the name" to "the function actually bound," and verified in both
directions — a healthy unit passes, an injected fault fails.

> **An interface name is not an implementation.** A check that looks only
> at the name will pass a wrong implementation that happens to share it.

MAC pinning is a similar story. The generic NCM driver does not read the
permanent MAC the module vendor programmed in, so **the address changed on
every boot.** From the host's point of view, a brand-new device appeared at
every plug-in, and network profiles piled up. We fixed it by explicitly
pinning the vendor MAC during configuration, and confirmed it survives a
reboot.

## We Measured 900 Mbit/s. It Was a Lie.

With the function working, we measured throughput. The first result was
**in the 900 Mbit/s range.**

We almost celebrated. Then arithmetic intervened: USB 2.0's theoretical
maximum is 480 Mbit/s, and this number was above it.

**When a measurement is physically impossible, the measurement is wrong.**

Here is what had happened. The test PC was attached to **the same subnet
twice** — once over Ethernet, once over USB. In that state, telling the
benchmark tool "use this IP as the source" makes it **change the source
address, not the interface.** A large share of the traffic flowed over
Ethernet, and that is what produced the 900.

We re-measured with the interface bound explicitly, and **cross-checked
against the USB interface's own byte counters.**

| Direction | Measured |
|---|---|
| PC → router | **315 Mbit/s** |
| Router → PC | **365 Mbit/s** |

That also matched what users had actually been seeing on Windows
(~300 Mbit/s).

### Where the Bottleneck Was Mattered

Once those numbers came in, the natural question surfaced inside the team:
NCM doesn't ride the hardware-accelerated path — is that why it's slow?
And if so, should the configuration choice be reconsidered?

We checked, and the link was **negotiating at High Speed (480 Mbit/s).**
Accounting for protocol overhead, 315/365 is **the practical ceiling of
USB 2.0.**

So the bottleneck was neither NCM nor the CPU — it was **the physical
link.** Which means there is **no performance-based case for revisiting the
configuration choice.** Any of the other functions would have hit the same
wall.

Pinning this down mattered, because an unanswered question like this comes
back every six months. A question left open keeps demanding re-review.

## And It Almost Didn't Ship

Finally, the scariest part of this whole job.

We flashed the first image, and **the USB configuration change wasn't in
it.**

When the build system decided which packages to recompile, the path holding
our modified file **was not in its change-mapping.** So the change went
undetected, and a stale, previously-built artifact went into the image.
**With no error whatsoever.**

There was exactly one way it got caught — **opening the built image's
filesystem and looking.**

> **A successful build does not mean your change is in the image.**
> Skip the pre-release image inspection, and this class of omission gets
> discovered in the field.

Since this incident, the build script's change-detection scope has been
widened, and inspecting image contents right before release is now a fixed
step.

## Lessons

- **"Support all three" is not a feature request — it's a constraint.** It
  narrows the options down to one.
- **Hardware resources (endpoints) decide software design.** "Just put in
  both" usually doesn't survive contact with the endpoint budget.
- **Change an identifier and you change the host's whole world.** Keeping
  the PID and swapping the contents is almost always the better trade.
- **Never assume incoming parts are uniform.** Absorbing the variance with
  **first-boot normalization** is cheaper than absorbing it with inspection.
- **Check the implementation, not the name.** And the only way to trust
  that check is to inject a fault against it.
- **When an impossible number appears, suspect the measurement.** And the
  question isn't closed until you know where the bottleneck actually is.
- **Build success ≠ change included.** You know only after opening the
  image.
