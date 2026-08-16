---
title: "We Stopped Trusting 'Verified by Design' — Bricking Our Router Twice, On Purpose"
date: 2026-09-07T06:00:00+09:00
draft: true
tags: ["u-boot", "openwrt", "firmware-update", "reliability", "testing"]
summary: "The A/B rollback on our 5G FWA router had never actually fired in three years, and the docs graded it 'verified by design.' So we broke the router two different ways on purpose. It came back on its own in 26.5 and 43.5 seconds — and along the way we found two places where our documentation was wrong."
---

## The Worst Case of a Remote Update

Our 5G FWA routers, the LS500X and NM500G (Qualcomm IPQ8072, based on
OpenWrt 25.12), sit in homes and small offices, and the carrier's management
server (ACS) pushes firmware to them remotely. In that picture there is
exactly one worst case: **the update finishes and the box doesn't come
back.** That means a technician in a truck, and in the FWA business one
truck roll can cost more than the device itself.

So the flash is split into two slots, A and B. An upgrade always writes to
the slot not currently in use, and if the new slot fails to boot, the
bootloader falls back to the previous one.

The implementation has one constraint worth knowing: this A/B is built
**without modifying a single line of bootloader source** — it lives entirely
in the vendor U-Boot's environment scripts. Not touching the production
bootloader is itself risk management. But this U-Boot has no `setexpr`,
which means no arithmetic. The boot counter is incremented by a ladder of
`if test` comparisons, and because of that structure, the retry limit maxes
out at 3.

The cycle: U-Boot bumps the counter on every boot and switches slots when it
hits the limit; once userspace comes up, a service resets the counter to
zero. In other words, **"the counter went back to zero" is the definition of
"the boot succeeded."**

## Nobody Had Ever Broken It

Everything around the rollback was measured. A corrupted image gets rejected
in 3.9 seconds without touching the flash. We had watched the slots swap
A→B→A. We had watched an ACS-initiated upgrade go from download command to
success report in about 92 seconds.

**The rollback itself had never fired. Not once.** Our internal docs graded
that line "verified by design" — which, read again, means *nobody has ever
tried it*.

So we induced it. Two different ways.

**Drill A — the pushed image can't be read at all.** We overwrote the
inactive slot's kernel volume with 1 MiB of random data; the FDT magic went
from `d00dfeed` to `4a46f528`. Then we staged the board to look exactly like
a unit that had just finished an upgrade — counter at zero, active slot
pointing at the broken one — and powered it on.

**Drill B — the kernel boots, but the system never comes alive.** The nastier
one. We didn't touch a single byte of flash; we just appended
`init=/bin/false` to that slot's kernel arguments. The kernel boots and
userspace dies instantly.

What the console showed:

```
Drill A                                    Drill B
[13.08] [A/B] slot=1 bootcount=1          [13.13] [A/B] slot=1 bootcount=1
[13.51] Wrong Image Format for bootm      [15.34] Kernel command line: ... init=/bin/false
[13.52] resetting ...                     [23.57] U-Boot 2016.01        <- died, reset
[22.90] [A/B] slot=1 bootcount=3          [40.16] [A/B] slot=1 bootcount=3
[23.39] [A/B] failover to slot 0          [40.66] [A/B] failover to slot 0
[36.49] Please press Enter to activate    [53.57] Please press Enter to activate
```

Results:

| | Drill A (corrupt image) | Drill B (boots, then dies) |
|---|---|---|
| Cost per failed attempt | 4.9 s | 13.5 s |
| First reset → usable device | **26.5 s** | **43.5 s** |

In both cases the board booted the previous slot and the counter went back
to zero — the recovered boot was recorded as a success. No truck, no human,
under a minute.

That would already be a happy ending. But the real payoff of the drills was
something else.

## Finding 1: "Rolls Back After 3 Failures" Was Wrong. It's 2.

The limit is 3, so the bad slot gets three attempts — that's what three
separate documents said. The logs say otherwise. U-Boot increments the
counter *first* and then checks it, and the check switches slots when the
counter *equals* the limit. So the third U-Boot execution bumps the counter
to 3 and switches slots **within that same run**, booting the good slot. The
bad slot only ever gets two actual boot attempts.

The behavior is *better* than documented — recovery is one cycle faster. The
problem is that the documentation was wrong, and the more important question
is why nobody caught it.

An off-by-one in a recovery mechanism is invisible until someone induces a
failure. That line was read, re-read, and copied verbatim into three
documents. Reading is not verification.

## Finding 2: If the Kernel Hangs, Rollback Never Fires At All

The boot counter only increments when the board resets. And the shipping
kernel command line had no `panic=`. On a kernel panic, the board doesn't
reset — it just sits there. No reset, no counter, no A/B.

Which means the 43.5 seconds above was actually measured with `panic=5`
added by hand as test rig. We recorded it honestly as a floor value, not a
field number, and opened a new gap item.

The follow-up: we shipped `panic=5` in the board's kernel arguments via the
DTS and verified it on 3.1.33. Forced panic → reset → back in service in
about 75 seconds, counter reset to zero, and the panic dump preserved and
readable on the next boot.

And while doing that, we also had to *shrink* our own earlier claim. "A
panic always hangs the board" was too broad. OpenWrt applies kernel
parameters after boot completes, so a fully booted unit had always rebooted
on panic. The real hole was only the window before userspace applies that
setting — which is exactly the situation Drill B creates. When you find a
defect, you also write down its boundaries; otherwise the next reader of
your doc gets fooled in the other direction.

## Sidebar: The Instruments Lied Three Times

The biggest time sink in these drills wasn't the board. It was the
measurement setup.

1. **`grep` printed nothing.** Serial captures contain control characters,
   and this environment's grep silently returns empty results on such files.
   On screen, that is indistinguishable from "the rollback never happened."
   We only noticed after extracting the same lines a different way. The most
   dangerous instrument is the one that gives you a *plausible* answer
   instead of a wrong one.
2. **Two console readers shared one port.** The capture came out as
   interleaved garbage — while the board, it turned out, had performed the
   rollback flawlessly. A corrupted capture is not a failed test.
3. **Writing an empty value to a U-Boot environment variable deletes the
   variable — and returns success.** We had backed up a boot-critical
   variable to a temp directory that a reboot then wiped, and the restore
   command deleted the variable and returned 0. If we hadn't read it back to
   verify, we'd have genuinely disabled the board mid-test.

We also built a tool out of this: a serial capture mode that writes zero
bytes to the port. That's not a stylistic preference — one stray newline
during U-Boot's 2-second autoboot countdown parks the board at a prompt
forever, and during a rollback drill that is indistinguishable from the hang
you're trying to measure.

## What A/B Cannot Save You From

For honesty's sake. We once killed a build mid-package-install. The
resulting image passed validation, booted, and reached a shell — with no
networking. A/B rollback can't catch this, because the kernel is fine. Worse,
upgrading to a good image while keeping settings carried the corrupted
config layer forward and contaminated the new image too. Only a
factory-clean flash broke the loop.

The lesson is unglamorous: don't kill builds halfway, and if you do, delete
that image. And never say "our safety net catches everything." A/B protects
against boot failure. Nothing more, nothing less.

## The Numbers

All measured on real hardware.

| Item | Value |
|---|---|
| Corrupt image rejection | 3.9 s, flash untouched |
| Rollback — corrupt image | 4.9 s per failure · usable in 26.5 s from first reset |
| Rollback — boots then dies | 13.5 s per failure · 43.5 s |
| Kernel panic → automatic recovery (3.1.33) | ~75 s, panic dump preserved |
| Full ACS remote upgrade | ~92 s (download command → success report) |

## What We Keep

- "Verified by design" is a polite way of writing "nobody has ever tried
  it." A recovery path isn't verified until you've broken things on purpose.
- Documentation hardens with every read. One wrong line got copied into
  three documents. Only re-measuring against real hardware breaks that.
- Distrust your instruments first. All three lies in these drills came from
  the measurement side, not the board.
- When you sell a safety net, also write down what it cannot catch. That's
  what makes the rest of your numbers believable.
