---
title: "The Real Reason RS-232 Was Dead"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [openwrt, rs232, gpio, quark-x1000, debugging, serial]
summary: "The industrial serial port produced no RS-232 signal, and the internal docs blamed a dead charge pump in the transceiver. Then we read the three mode GPIOs from the known-good Yocto firmware: 0/0/1. Ours were driving 1/0/0. The chip was fine."
---

<!-- Part 10 of the LTE572W Yocto-to-OpenWrt porting series -->

## Background

The LTE572W has an industrial serial port. One UART (ttyS0) runs through an
SP336 transceiver out to a DB9 connector, and the line mode is selected by
three GPIOs (`SER_MODE0/1/2`). The SP336 datasheet's mode table looks like
this:

| Mode | MODE2/1/0 |
| --- | --- |
| RS-232 | `001` |
| RS-485/422 full duplex | `101` |
| RS-485 half duplex | `100` |
| Sleep | `110` |

One prerequisite worth noting: to see these GPIOs at all, the kernel needs
`CONFIG_LPC_SCH=y`. Without it there is no error — the `sch_gpio` chip simply
never appears. That alone was one trap along the way.

## The Symptom

Software says the port is in `rs232` mode. Reading the GPIOs back returns
exactly the values we wrote. And yet no RS-232 levels come out of the pins.

## The Existing Conclusions — All Plausible, All Wrong

This symptom already had a history. The internal documentation had closed it
out as a hardware failure: "the SP336's TX charge pump is dead." One piece of
evidence was the observation that a DB9 pin that should have been an output
carried no signal. But this board's DB9 layout is non-standard, and that pin
is actually an input (CTS), not an output. An input pin sitting at 0 V is
normal. What had been read as proof of a failure was, in fact, correct
behavior. On top of that, a loopback test had returned zero bytes of RX,
which was being blamed on flow control on the PC side.

I believed that document too, and spent a while staring only at the hardware.
When wrong conclusions stack up two layers deep, they narrow the search space
in exactly the wrong direction.

## Changing the Approach — Compare Against a Known-Good System

On this same board, the Yocto firmware produces RS-232 just fine. Which means
there is one question worth asking: what are the GPIOs set to *then*?

```
Known-good Yocto RS-232 boot:  three mode GPIOs = 0/0/1
Our OpenWrt:                   three mode GPIOs = 1/0/0
```

Completely different. Yet our code insisted it had written the requested
values, and readback agreed. So what was wrong wasn't the values — it was the
interpretation of the values.

## The Root Cause

The code had its bit-to-pin mapping inverted. The mapping the code assumed
and the actual board wiring were swapped on two of the signal lines —
`SER_MODE0` and `SER_MODE2` — which we confirmed against the schematic. So a
request for `rs232` (mode `001`) actually put the chip into mode `100`:
RS-485 half duplex. With the SP336 sitting in RS-485 half duplex, of course
it produced no RS-232 levels. That is what made it *look like* a dead charge
pump.

Fixing that one mapping line collapsed three unresolved items at once. The
"dead charge pump" — the chip was healthy; the port had simply never been in
RS-232 mode. The zero-byte loopback — not a PC flow-control problem, same
cause. The "failed" dry-contact test — same cause again. When a single root
cause explains three open issues simultaneously, that is the signal that you
have actually found it.

## What "Verified" Really Meant

So why did nobody know? An earlier comment in the code read: "verified by
GPIO readback."

Readback only confirms that the value you intended landed in the register. It
says nothing about what that value *means* in hardware. That comment used the
word "verified" to seal in an unverified assumption — and it sent the next
person (me) in the wrong direction more than once.

## A Side Story — The Kernel Patch That Did Nothing

From the same serial work, I also inherited an RS-485 kernel patch, and on
inspection it was doing nothing at all. The patch installed two callbacks in
a setup function — but the probe code called the common initialization
function on the very next line, and that function reassigned the same two
pointers, plus two more the patch had missed. Everything the patch wrote was
always overwritten. With the patch removed, a TIOCGRS485/TIOCSRS485 prober
confirmed RS-485 still worked, so the patch was deleted. Same lesson: "there
is a patch for it, so it must be working" is not verification either.

One last note. The serial daemon (`serialgw`) that owns this port reads
`/sys/class/tty/console/active` and refuses to open whatever the system
console device is. In an earlier part of this series, ModemManager probed the
console UART and wrecked the console. Once was enough.

## Lessons

- Readback is not verification. You have to observe the result in hardware —
  an actual signal, or a comparison against a known-good system.
- If another firmware works correctly on the same board, it is the most
  accurate reference document you have. Dumping and diffing GPIO state was
  faster than re-reading the documentation.
- A wrong conclusion does not stay where it was written. It enters the
  documentation and narrows the next person's search space. Because of one
  line that said "charge pump failure," nobody looked at the software side
  again.
