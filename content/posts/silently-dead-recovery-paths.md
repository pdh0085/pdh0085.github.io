---
title: "Two Recovery Paths That Were Silently Dead"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [openwrt, sysupgrade, factory-reset, procd, shell-scripting, embedded]
summary: "`sysupgrade -n` had never once wiped the configuration, and factory reset was broken twice, for two different reasons. Both reported success without doing anything. One died at a process boundary; the other because a volume abstraction confidently declared 256 KB of alignment padding to be rootfs_data."
---

<!-- Part 6 of the LTE572W Yocto-to-OpenWrt porting series -->

Recovery features have one nasty property: nobody uses them day to day, so
when they break, nobody notices. This post is the record of finding two such
paths on the LTE572W, both silently dead. Each of them reported success,
error-free, while doing nothing at all. They broke for different reasons,
too: one at a process boundary, the other inside an abstraction that
misfired.

## 1. `sysupgrade -n` Doesn't Wipe the Configuration

The symptom: flash new firmware with the "reset settings" option, and the
old settings are still there. Worse, because the overlay survived, the
`uci-defaults` scripts never run again either. The "factory-fresh new
firmware" was not fresh in any way.

The code looks perfectly fine. Inside `platform_do_upgrade` there was this
check:

```sh
[ "${SAVE_CONFIG:-1}" != "1" ] && rm -f "$overlay_img"
```

The syntax is correct and so is the logic. The problem is that the variable
never makes it to that line.

`platform_do_upgrade` runs in stage 2. After `ubus call system sysupgrade`,
procd execs into a fresh environment, and the only values that cross that
boundary are prefix / path / force / backup / command / options, which
procd hands over as `$IMAGE`, `$UPGRADE_BACKUP`, `$WDTFD`, and
`$UPGRADE_OPT_*`.

```
stage1 (/sbin/sysupgrade)
   SAVE_CONFIG=1 export ─────────┐
   ubus call system sysupgrade   │  what crosses this boundary:
                                 │  prefix / path / force / backup / command / options
                                 ▼
stage2 (procd execs a fresh environment)
   $IMAGE  $UPGRADE_BACKUP  $WDTFD  $UPGRADE_OPT_*
   ← $SAVE_CONFIG does not exist here (empty → defaults to "keep config")
```

`$SAVE_CONFIG` is exported in stage 1 and dies there. So the test always
reads an empty variable, falls into the `:-1` default, and behaves as "keep
the configuration." Every single time, `-n` or not.

The correct signal was already crossing the boundary. Stage 1's
`sbin/sysupgrade` only puts `backup` into the ubus call when
`SAVE_CONFIG=1`, so the *absence* of `$UPGRADE_BACKUP` is the wipe signal.
I switched the condition to that and verified both directions on the serial
console: with `-n`, the overlay-removal log line appears and the planted
markers (an APN, the hostname) are gone; with a normal upgrade, they
survive.

The lesson fits in one line. Shell's `${VAR:-default}` is convenient, but
it cannot tell "the variable is unset" apart from "the variable holds the
default." In code that crosses a process boundary, that's dangerous.

## 2. Factory Reset Was Broken Twice, for Different Reasons

Background first. On this board, a single `factoryreset` binary serves
firstboot, the hardware reset button, and TR-069's `rpc-sys factory`. Fix
it once and all three are fixed; break it once and all three break
together.

First failure: the volume can't be found. The stock code starts with
`volume_find("rootfs_data")`. On a board with real flash partitions that's
a perfectly sound assumption, but our overlay is a loop-mounted ext4 image
file sitting on FAT, so none of the drivers can match it. Not mtd (there
is no `/proc/mtd` at all), not ubi, not partname.

```
factoryreset: MTD partition 'rootfs_data' not found
```

Then it aborts. Every reset path quietly did nothing. Up to this point the
cause is our unusual layout, not the code.

First fix, which then rotted. I added a fallback: "if the volume can't be
found, wipe the mounted `/overlay`." It worked. Right up until the root
filesystem moved to a squashfs loop.

Second failure: the abstraction *succeeds*. Once the root became
squashfs, the `rootdisk` driver started binding. That driver looks for a
squashfs superblock on the root block device and declares everything after
it to be `rootfs_data`. For a board with squashfs on a real partition,
that's a reasonable rule. But what sits after *our* squashfs is 256 KB of
alignment padding on a read-only loop device. So:

- `volume_find()` succeeds, so the fallback never fires
- it returns an unwritable device a few KB in size
- `jffs2_reset()` tries to run `mkfs.ext4` on it

```
Filesystem too small for a journal
mkfs.ext4: I/O error while writing out and closing file system
factoryreset: /dev/loop2 will be erased on next mount
```

And then it returns 0. Nothing erased, no reboot. Every reset path was
silently dead again. In the first failure the volume layer answered "I
don't know," which left room for a fallback to live. This time it answered
confidently while being wrong, and that confidence is exactly what
disarmed the fallback.

The final fix was to remove the condition altogether. Stop asking the
volume layer entirely: if `/overlay` is mounted, delete its files. This is
precisely what `jffs2_reset()` does for a mounted volume anyway, so
volume-based boards are unaffected. As a bonus I fixed LuCI's reset page
too; it never checked the exit code, so it had been reporting the failed
resets above as successes.

One small postscript. `firstboot -y` erases but does not reboot. Wiping
the overlay also wipes the ssh host keys, so a board left running after it
will thoroughly confuse whoever connects next. Always add `-r`.

## Lessons

Recovery features are never exercised by normal use, so the only option is
to actually press them, every release. Neither of these bugs would have
surfaced on the happy path. And "failed, but returned 0" is the worst kind
of failure — the harder a feature's outcome is to observe (resets,
upgrades), the more its exit code and its post-state both need checking.

The abstraction story deserves its own sentence. An abstraction can coexist
with a safety net only when it admits what it doesn't know. `rootdisk`
answered confidently without understanding our layout, and that confidence
killed the fallback. Of course, half the cause is our own odd
loop-ext4-on-FAT arrangement. If you choose to live outside the world an
abstraction assumes, doubting its answers becomes your job too.
