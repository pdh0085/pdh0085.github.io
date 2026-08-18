---
title: "Epilogue: Hardware Is the Truth"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "quark-x1000", "embedded", "debugging", "retrospective"]
summary: "The port ended with the 3.0.0 release. Looking back, the most expensive thing was never a hard bug — it was wrong records like \"freed about 147 MB\" and \"verified\". The four beliefs this series disproved, and the habits that replaced them."
---

<!-- Part 13 of the LTE572W Yocto-to-OpenWrt porting series -->

## It Ended with 3.0.0

The results fit in one paragraph. Our Yocto-based firmware became OpenWrt
25.12; cellular and GNSS moved to ModemManager; boot became a squashfs on FAT
plus a fallback ladder; the web UI became a LuCI-based app of our own. The
3.0.0 production release shipped on July 15, 2026.

This final post is a retrospective not on the results but on the way we
worked. The thing that consumed the most time in this project was not any
difficult bug. It was **wrong certainty** — more precisely, incorrect facts
that had been written down as "already confirmed." A record like that narrows
the search space of the next person, who is usually yourself a few weeks
later. Here are the four beliefs this series disproved.

## Wrong Belief 1: "Verified by GPIO Readback"

This is from the serial post. A readback only confirms that the intended
value landed in the register. It says nothing about what that value **means**
to the hardware. In reality the code had SER_MODE0 and SER_MODE2 swapped, so
requesting RS-232 actually selected RS-485 half-duplex. But the document said
"verified," and that single word kept everyone from re-reading the software.
In its place, a hardware conclusion took root: "SP336 TX charge pump dead."

## Wrong Belief 2: Plausible Causality

There was a comment: "powering the modem early makes it contend with Wi-Fi
enumeration on the shared EHCI bus." Physically plausible, and it even
matched a symptom observed long ago. Except Wi-Fi enumerates about 3 seconds
into kernel boot — before init scripts even exist. **A sentence that welds an
observation to a cause loses the observation over time; only the cause
survives.** That one comment pinned the modem power script at START=99 and
pushed the modem's USB appearance out to about 162 seconds. After moving it,
about 50.

## Wrong Belief 3: The Metric That Looks Good

A record claimed "this work freed about 147 MB of memory." What actually
moved was **peak boot RAM** (initmem freed: 67456K down to 1564K); steady-state
`MemAvailable` stayed essentially unchanged at roughly 116–134 MB. Those
67 MB were being returned by the kernel during boot anyway. The most common
lie in optimization write-ups is not inventing numbers — it is **choosing the
metric that looks improved.**

## Wrong Belief 4: The Failure Record Nobody Retried

There was a record that the EC25-AU fails on one domestic carrier's network
with `sim-missing`. Retried on a shipping board with a real SIM, it did not
reproduce — the modem registered, got an address over QMI, and passed data
with 0% ping loss. **Failure records outlive success records**, because
nobody ever tries again.

## Five Methods That Actually Worked

The four cases share one thing: progress happened only when the evidence came
from what the hardware was saying right now — not from documents, comments,
or memory.

- **Read the answer off a working system.** The ACPI RSDP address and the
  correct RS-232 GPIO values were both lifted directly from the old Yocto
  firmware booting happily on the bench. Faster than re-reading schematics.
- **Deliberately make the suspected condition worse.** The modem-power
  hypothesis was disproved by repeating the test under three conditions:
  idle, heavy CPU load, and during a live Wi-Fi firmware upload. All three
  enumerated cleanly in 9–10 seconds with zero errors.
- **Verify at the far end.** An interface holding an IP address is not proof
  of connectivity (the modem post), and a successful file write is not proof
  the configuration reached the daemon (the routing post).
- **Break it on purpose.** Until we physically pulled 12V in the middle of a
  sysupgrade, the fallback design was just a plausible story.
- **Log close to the hardware.** The early-boot `/init` and the init-script
  profiler both write to `/dev/kmsg`. With a serial console, you see
  everything even when networking is dead.

## How We Write Documents Now

- Every fact carries **how we know it**. Not "verified" but "confirmed on a
  real board with this command." The rule goes down to a single connector
  name (a document said CON8; it was actually CON6).
- Items proven wrong are not deleted — they stay, together with the reason
  they were overturned. Delete them and the next person rebuilds the same
  hypothesis from scratch.
- The rule "never put logic where the field can't update it" (from the
  uci-defaults post) applies to documents too: a document nobody reads might
  as well not exist.

## What Remains

- Boot still takes about 190 seconds. We closed one large hole; the rest of
  the profile (S95frr at about 30 seconds, among others) is still
  uninvestigated.
- TR-069 (icwmp) has never been tested against a real ACS.
- Two backends the UI references but that do not exist (`battery-mgrd`,
  `xoleds`) are still outstanding.

Leaving the unfinished list in plain sight is part of the record too.

The thing I did most often on this project was not fixing code. It was
**taking something that someone — usually me — had written down as already
confirmed, and confirming it again in front of the hardware.**
