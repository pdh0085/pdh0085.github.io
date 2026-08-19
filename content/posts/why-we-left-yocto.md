---
title: "We Threw Away Firmware That Worked"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "yocto", "quark-x1000", "embedded", "porting"]
summary: "The LTE572W runs on an Intel Quark X1000: a single 400 MHz core and 256 MB of RAM. Its shipping firmware was a Yocto image that worked fine. We replaced all of it with OpenWrt 25.12. This is why, and these are the three constraints that would go on to shape everything that followed."
---

<!-- Part 1 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Starting Point

An industrial LTE router, the LTE572W. The CPU is an Intel Quark X1000
(Lakemont): i586, single core at 400 MHz, no SMP, no MMX or SSE, though it
does have an x87 FPU. 256 MB of RAM (the kernel sees 253 MB), 3.6 GB of
eMMC.

The shipping firmware was a Yocto image based on Intel's X1000 reference
design, which we at Lightspeed had been modifying and shipping. Cellular,
GNSS, serial, the web UI, all of it lived on that stack. And it worked. This
was a released product, running in the field.

Even so, at the end of June 2026, we decided to replace all of it with
OpenWrt 25.12.

## Why

Three reasons.

First, we needed a maintainable base. The kernel, userspace, and daemons were
all specific to the reference stack, and the cellular side was tied to
Qualcomm's proprietary API. When the modem module changed, the code had to
change with it.

Second, mainline gives you a lot for free. ModemManager, mwan3, FRR, LuCI,
sysupgrade, overlayfs, the package feeds. Most of what we had been building
and maintaining ourselves already exists in OpenWrt.

Third, old hardware is not an excuse. The Quark X1000 is right there in the
mainline kernel (`CONFIG_X86_INTEL_QUARK`). The difficulty doesn't come from
missing support. It comes from the fact that **nobody had ever booted this
particular combination before.** That sentence sets the tone for the whole
series.

## Constraint 1: The eMMC Cannot Be Repartitioned

The boot chain looks like this: EDKII and a GRUB Legacy 0.97 fork live in
SPI flash, and GRUB reads `grub.conf` and the kernel from a FAT partition on
the eMMC.

```
SPI: EDKII → GRUB Legacy 0.97
                 │  (grub.conf and kernels are read from the eMMC FAT)
                 ▼
eMMC mmcblk0p1 (FAT, vendor files untouched)
   ├─ original kernel / initramfs / rootimage   ← left alone (rollback path)
   ├─ openwrt-bzImage                           ← file we added
   ├─ openwrt-root.squashfs                     ← file we added
   └─ openwrt-overlay.img                       ← file we added
```

Damage the partition table or the FAT and GRUB can't read its own menu, at
which point you can't even reach the recovery entry in SPI. The only way back
is physically reflashing the board. So the rule was set on day one:

> Every change must be additive and reversible. No repartitioning. Never run
> any grub tooling. The original files and the original boot entry stay.

OpenWrt exists on that FAT as three files (a kernel image, a squashfs root,
and an ext4 overlay image) and touches nothing else.

## Constraint 2: 400 MHz and 256 MB Is a Design Constraint, Not "Slowness"

On this hardware, slowness isn't a user-experience complaint. It's a force
that changes designs. Later posts cover these one by one, but here's a
preview of decisions this constraint actually reversed:

- squashfs compression had to be gzip, not XZ (XZ decompression is far too
  slow here)
- cellular must attach over QMI, because the AT+PPP fallback is userspace
  HDLC, which is a disaster on this CPU
- forking a single `mmcli` process during boot costs 6–9 seconds, and it was
  happening 23 times
- and if your root filesystem lives in RAM, large file uploads die from OOM.
  That one was reported from the field, and the entire middle of this series
  starts from it.

## Constraint 3: Coexistence

There is a separate Windows migration tool that moves existing units to the
new firmware, and that tool owns `grub.conf` (it stamps per-board values
into it). So the firmware never overwrites that file. We crossed that
ownership boundary exactly once and then backed out; that story belongs to
the boot-fallback post.

## The Working Setup

The board was always physically on the bench, and the decisive tool of this
project was the serial console. With it, you can watch the entire boot even
when networking is dead, and the boot scripts log each step to `/dev/kmsg`.
The difference between "reboot blind and guess" and "watch and iterate fast"
is enormous.

One small field detail: `scp` doesn't work on this board (dropbear ships
without an sftp-server). Every file copy went over an ssh pipe:

```sh
cat file | ssh root@192.168.10.1 'cat > /dest'
```

## How It Ended

From the subtarget skeleton at the end of June to the 3.0.0 production
release on July 15. In summary:

| Area | Before (Yocto) | After (OpenWrt 25.12) |
| --- | --- | --- |
| OS base | X1000-reference Yocto | new `x86/quark` subtarget |
| Cellular / GNSS | proprietary stack (Qualcomm API) | ModemManager (mainline) |
| Boot | A/B partition switch | squashfs on eMMC + boot fallback ladder |
| TR-069 | none | icwmp (new) |
| Web UI | legacy UI | LuCI-based app + custom theme |

There were known limitations at release, too: boot takes about three
minutes, a SIM is required, and some carriers need the APN set manually.
Writing those down instead of hiding them is also one of this series' rules.

## What This Series Is About

Not a feature tour. Most of the posts ahead are stories like these: a
factory reset that reported success while doing nothing, a config file that
had never once been read, a modem that said "connected" while passing zero
packets, and inherited assumptions I finally disproved with experiments.
The common thread is things that were failing silently, and what it took to
notice them.

Next up is the first gate: building a toolchain in 2026 for an i586 CPU with
no MMX and no SSE. The first build broke not in the compiler, but in libc's
`fabs()`.
