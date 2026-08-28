---
title: "Getting the Root Filesystem Out of RAM"
date: 2026-08-28T15:18:38+09:00
draft: false
tags: ["openwrt", "quark-x1000", "squashfs", "initramfs", "oom", "embedded"]
summary: "Firmware uploads through the web UI were dying with HTTP 502. The web server was innocent. The root filesystem was sitting in RAM, with about 70 MB of /rom pinned and unreclaimable, and zram didn't help. The fix was moving the root onto a loop-mounted squashfs, and along the way admitting that an old claim of '~147 MB freed' had never been true."
---

<!-- Part 4 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Symptom: A 502 That Only Sometimes Reproduces

The report from the field went like this: the web UI's "Upload & Verify"
firmware upload dies with HTTP 502. But not always. On a freshly booted
board it succeeds. That "sometimes it works" turned out to be the biggest
hint. Of everything in this project, this is the problem where the distance
between symptom and cause was the longest.

## Reproducing It

LuCI's upload streams the 19 MB image into `/tmp`. `/tmp` is tmpfs, which is
RAM. And at this point in the project, the root filesystem itself was an
initramfs unpacked into tmpfs, which meant about 70 MB of `/rom` was pinned
and unreclaimable. tmpfs pages can't be dropped the way page cache can.

Upload on a board whose free RAM has shrunk, and the OOM killer takes out
cgi-io; uhttpd returns 502.

| Condition | MemAvailable | Result |
| --- | --- | --- |
| Freshly booted board | ~81 MB | 200 |
| Simulated low memory | ~21 MB | 502 |

An exact match for what customers were seeing.

## The Most Plausible Quick Fix: zram (It Failed)

The first thing I tried was zram-swap. Even with compressed swap enabled and
44 MB swapped out, all four hardware tests returned 502. On a single
400 MHz Lakemont core, the kernel's reclaim-and-compress path simply cannot
keep up with the rate of an upload burst.

The lesson was blunt: instead of squeezing memory harder, we should never
have been holding it in the first place.

## The Real Fix, in Outline

Put the root filesystem on the FAT partition as a squashfs file, loop-mount
it, and stack an overlayfs on top, the upperdir being a 64 MB ext4 image
file on the same FAT, also loop-mounted. Unlike tmpfs, squashfs pages are
file-backed page cache: under memory pressure the kernel can drop them and
read them back later. They become reclaimable.

## Step 1: Prove the squashfs Is Actually a Bootable Root

Before touching the boot path, I copied the built squashfs onto a running
board, loop-mounted it, and `chroot`ed in. Two traps surfaced immediately.

First, the file must be padded to a 256 KB boundary. Truncate it to its
exact size and the loop device rounds the capacity *down*, and the mount
dies like this:

```
SQUASHFS error: Failed to read block ...: -5
```

Second, the compression has to be gzip. With XZ, reading the full tree
takes over 90 seconds on this CPU. After switching to gzip, individual file
access is under a second, and a full-tree read settles at about 81 seconds.
The remaining time is I/O bound (squashfs over loop over vfat), not
decompression, which also means switching further to lz4 would buy nothing.

## Step 2, Attempt 1: Failure, and a Hung Board

The first attempt was to loop-mount in a preinit hook and `exec
switch_root`. busybox refused:

```
BusyBox switch_root ... Usage: ...  PID must be 1. NEW_ROOT must be a mountpoint.
```

A preinit hook is a subshell forked by procd, so it is not PID 1. (The
reason OpenWrt's stock overlay transition works fine is that it uses
`pivot_root`, which has no PID 1 requirement.) Worse: when an `exec` fails
like this, your process has already been replaced, so the fallback code
written further down the script is never reached.

The board hung. What saved it was a one-shot flag planted in advance:
power-cycle, and it booted the old path. This was the moment that safety
pattern actually paid for itself; the full story belongs to the next post in
this series.

## Step 2, Attempt 2: Rewrite PID 1 Itself

The process that *is* PID 1 is the kernel initramfs's own `/init`. That
script is already the thing that copies the rootfs into tmpfs and calls
`switch_root`, so the answer is to rewrite it to try the squashfs path
first. Things I learned doing that:

- At `/init` time, `/dev` is empty (no devtmpfs). The nodes you need
  (`mmcblk0p1`, `loop0`, `loop-control`) have to be created with `mknod` by
  hand.
- OpenWrt's `modprobe` is `kmodloader`. In a loader, the reliable move is
  full-path `insmod` in dependency order. (Eventually I built FAT and NLS
  into the kernel and eliminated the modules altogether.)
- The squashfs loop pins its backing file on the FAT, so even after
  `switch_root` drops the FAT mount point, reads keep flowing through the
  loop device's fd.

```
loop0: detected capacity change from 0 to 40960
```

This time it went through.

## How Much Better It Got, and Where Honesty Is Required

Here are the numbers.

| Metric | Before | After |
| --- | --- | --- |
| Loader kernel size | 19,804,672 bytes (~19.8 MB) | 5,034,496 bytes (~5.0 MB) |
| `Freeing unused kernel image (initmem)` | 67456K | 1564K |
| Peak boot RAM | ~133 MB | ~2 MB |
| Steady-state MemAvailable | ~116–134 MB | unchanged |

Two things need to be said plainly.

First, the 5 MB kernel is not this post's work. It comes from the
minimal-loader initramfs conversion that followed a day later, which is the
next post's story. The two changes are a day apart and easy to conflate;
this post's share ends at "the root is no longer tmpfs, and its pages are
reclaimable."

Second, an old internal document credited this work with "about 147 MB
freed." **That number was never true, and it was later corrected.**
Steady-state `MemAvailable` is essentially unchanged, because those 67 MB
were being returned by `free_initmem()` at the end of boot anyway. The
metrics that actually moved are peak boot RAM and the reclaimability of
`/rom` during an upload burst, and it's the latter that killed the OOM. The
most common lie in optimization write-ups is picking the metric that looks
like it improved.

## Verification, and the Next Wall

The upload OOM was gone. During a 40 MB upload, `MemAvailable` held at about
117 MB with zero OOM kills.

A different limit appeared instead. cgi-io computes both md5 and sha256
over the whole upload, and on this CPU that is CPU bound. A 19 MB image
takes about 51 seconds, which fits under uhttpd's default
`script_timeout=60`, so it returns 200. A 40 MB upload blows past 60
seconds and returns 502, but this time it is not OOM. Raise
`script_timeout` to 300 and it completes in about 108 seconds with a 200.

Same status code, completely different cause. The second demonstration in
this project that you must not infer the cause from the symptom code.

## Lessons

- What you put in tmpfs cannot be dropped under memory pressure. Before
  parking anything large in RAM, ask whether it is reclaimable. And
  compressed swap only works when the CPU can keep up; sometimes the answer
  is not squeezing harder but restructuring so the memory is never held.
- No fallback code can live after a point of no return like `exec`. Plant
  the fallback outside that boundary, in advance.
- In an optimization report, name the metric that actually moved. Credit a
  metric that didn't move and the document eventually disproves itself.
  Related: the same error code does not mean the same cause. Both 502s here
  had different reasons.
