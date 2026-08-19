---
title: "The Boot Escalation Ladder"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [openwrt, boot-fallback, sysupgrade, quark-x1000, embedded, reliability]
summary: "Once the root filesystem became a file on eMMC, sysupgrade became the act of replacing that file. So what happens when the new root fails to boot? We built a five-rung fallback ladder driven by a single boot_state file, and validated it by literally pulling the 12V supply mid-sysupgrade. Result: 144 modules loaded, no brick. The rung that actually matters was found in review, not testing."
---

<!-- Part 5 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Problem

In an earlier part of this series, the root filesystem became a squashfs file
sitting on the eMMC's FAT partition. That makes sysupgrade "the act of
replacing that file with a new one," which leaves exactly one question:
**what if the new root fails to boot?**

Industrial equipment usually lives where hands can't reach it: factory
ceilings, remote monitoring sites. A failed boot means a site visit. So
before trusting the new boot scheme at all, we designed the failure path
first. The conclusion up front: a fallback is not a nice-to-have, it is a
state machine. Where does the state live, what consumes it, and what does
not consume it? The design boiled down to those questions.

## Three Design Principles

1. Write the marker immediately before entering the dangerous path, just
   before the point of no return.
2. The marker is a file, not an environment variable (reasons below).
3. **A failed validation does not consume the marker.** A missing or broken
   fallback image must never cost the board a boot attempt.

## The Ladder

A single one-shot marker file on the FAT, `boot_state`, drives five rungs:

```
no boot_state    ─→ primary + persistent overlay
      │ (hang)
boot_state=primary ─→ golden + persistent overlay
      │ (hang)
boot_state=golden ─→ golden + volatile overlay
      │ (hang)
boot_state=golden-noovl ─→ primary + volatile overlay   ← the rung that actually saves boards
      │ (hang)
boot_state=primary-noovl ─→ rescue shell (auto-reboot after 300 s)

failed validation (file missing / mount failure / vermagic mismatch)
  = skipped WITHOUT writing the marker
  → a skipped rung falls through to the next one within the same boot
```

The marker is written right before `switch_root`, and an S99 script deletes
it once the boot comes up healthy. So if the loader finds the marker still
present, that means exactly one thing: **that candidate hung on the previous
boot.** On the serial console it reads like this:

```
quark-squashfs-init: boot_state=primary  → primary hung on the previous boot; escalating to golden
```

## Why There Are "No-Overlay" Rungs

The most plausible cause of a boot hang is neither the kernel nor the root.
It's a badly saved configuration. In that case the right move is to boot
the same root again while ignoring the config. But the root must not be
mounted read-only for this: dropbear can't write its host keys, so ssh never
comes up. Losing remote access in precisely the mode where remote access
matters most would be the worst outcome. So instead of a read-only root, the
no-overlay rungs mount a volatile tmpfs upper: the boot comes up with
clean config, and you can ssh in and repair the real overlay. As a related
belt-and-suspenders choice, the LAN bridge, Ethernet, and overlayfs drivers
are all built into the kernel, so ssh comes up even if module loading fails
entirely.

## The Rung That Does the Real Work Is primary-noovl

Here is this post's twist. At first glance the two golden rungs look like the
heroes. But golden can do nothing at all in the two situations that matter
most:

- A board fresh from the factory has no golden at all; golden is created
  by the first sysupgrade.
- After an update that changes the kernel version, the old golden has no
  `/lib/modules/$(uname -r)` and is rejected by validation.

In both cases, both golden rungs get skipped wholesale. Without a
`primary-noovl` rung, the next stop would be a rescue shell with no
networking, for a board whose root was probably fine and whose only
problem was configuration. Structurally, primary is the one root that is
always paired with the current kernel, so "primary once more, minus the
config" was a rung the ladder could not do without.

How it was found is worth recording too: this rung came out of a design
review, not a test. I had the ladder drawn out as a table and traced "where
does a factory-fresh board go if its first boot hangs?", and there was the
hole. It was a bug caught on paper, not reported by hardware, which made it
scarier: the first place it would have been stepped on was the field.

## Golden Is Not a Factory Image

The most counter-intuitive part of the design. Golden is not a frozen factory
copy. It is the last root that booted healthily on the kernel currently
sitting in the FAT. Freeze a factory copy instead, and the first kernel
update makes its vermagic mismatch; the safety net dies silently, at
exactly the moment it becomes needed. So golden is refreshed after every
healthy primary boot, and never after booting from golden (overwriting
your fallback with a fallback boot would let the ladder eat itself).

## Why the Marker Is a File

The temptation is to pass this state as an environment variable. But procd
scrubs its environ before rcS, and, more fundamentally, when a human runs
the script by hand over ssh, that variable simply doesn't exist. Both
failures are silent. If a safety decision like "is it safe to refresh
golden?" hangs off a signal that can silently vanish, the safety net rots
without a sound. A file reads the same for everyone, every time.

## The vermagic Gate and Two Own Goals

To filter out combinations where `uname -r` matches but the module ABI
doesn't, the loader carries the module vermagic string of its build-time
kernel and compares it against a `.ko` read from the candidate squashfs.
Implementing this cost two round trips to the hardware:

- The substitution placeholder had to appear exactly once, but a global
  `sed` also rewrote the comparison statement itself, so the gate was
  comparing against itself and could never fire.
- After a clean build, the loader got stamped from a stale `.ko` while
  the squashfs got the new one, producing an image that rejected its own
  primary. The build now self-checks that loader and rootfs vermagic
  match.

## One Detail from the Rescue Shell

The bottom rung, the rescue shell, prints the FAT contents and rewrite
instructions, then counts down about 300 seconds, clears the marker, and
reboots itself, so that a transient failure on a board with no serial
attached heals on its own. Any keypress drops to an interactive shell. The
fun trap: this shell is PID 1. `reboot` becomes `kill(1, SIGTERM)`, and
the kernel discards that signal when it's addressed to PID 1. You have to
install your own `trap ... TERM` or the reboot never happens.

## The Power-Pull Test

Everything up to this point is a plausible story. So we manufactured the
worst case for real. At the moment sysupgrade had finished writing the
squashfs but had not yet written the kernel, **we physically pulled the 12V
supply.**

| Item | Result |
| --- | --- |
| FAT state | new squashfs + old kernel + leftover `.new` file |
| vermagic | same build, so it matched → primary passed validation |
| Boot | primary came up clean, 144 modules loaded |
| Brick | none |

Not a single line of code differs between before and after that experiment.
But what I trusted about this design changed completely. A failure path is
not validated until you have deliberately made it fail.

## Disciplines We Picked Up Along the Way

- The build produces two images, but only one is flashable. The other (a
  kernel-only debug image with no rootfs) would leave the board rootless if
  installed, so the image check rejects it even under forced flashing
  (`-F`).
- Never overwrite a live squashfs in place. Write to `.new` and `mv`, so
  the active loop mount keeps the old inode. We learned this by doing
  `cat new > file` exactly once; uncached reads started returning I/O
  errors.
- If the loader kernel exceeds 6.5 MB, the build fails. The regression
  where the full rootfs cpio sneaks back into the initramfs is blocked by
  the build, not by human vigilance.
- We once made the firmware update `grub.conf` and then backed it out: that
  file is owned by the migration tool, which stamps per-board values into
  it. Ownership boundaries outlive convenience.

## Lessons

- A fallback is a state machine, and the central design question turned out
  to be what consumes the state. Only a hang consumes the marker; a failed
  validation skips without consuming it. That one distinction holds up the
  whole ladder.
- A safety net that isn't regularly refreshed rots. Golden is the latest
  healthy boot, not a factory image.
- Never hang a safety decision off a signal that can silently disappear
  (environment variables).
- Design review catches a class of bugs that testing can't. The most
  important rung of this ladder came from staring at a table, not at a
  board.
- A failure path is not validated until you have made it fail on purpose.
  Pulling the plug took one second, and that second turned this design
  from a story into a fact.
