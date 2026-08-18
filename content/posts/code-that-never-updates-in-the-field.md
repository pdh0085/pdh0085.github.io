---
title: "Code That Never Updates in the Field"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "uci-defaults", "overlayfs", "modemmanager", "hotplug", "embedded"]
summary: "Every boot on a fresh overlay, about 25 bytes of binary garbage hit the serial console around the 185-second mark. The culprit was ModemManager probing the console UART as if it were a modem. I wrote a filter and put it in uci-defaults — and it did nothing, for two separate reasons."
---

<!-- Part 7 of the LTE572W Yocto-to-OpenWrt porting series -->

## Four Bytes on the Console

Every time the board booted on a fresh overlay, at about 185 seconds — right
around `init complete` — roughly 25 bytes of binary noise appeared on the
serial console. Then normal output resumed. Perfectly reproducible, cause
nowhere in sight.

The thing to do in that situation is to actually read the bytes. The same
four bytes appeared every time:

```
... 00 78 f0 7e ...   ← HDLC: command 0x00 + CRC 78 f0 + flag 7e
```

That's an HDLC frame — specifically a QCDM/DIAG frame. It also explains why
the front of the blob looked mangled: something had opened the port, run
`tcsetattr` to its own probe baud rate (not 115200), written `AT`, then
written a QCDM frame and closed the port. The AT part comes out garbled; the
QCDM part prints as-is. **ModemManager was probing the console UART as a
modem.**

Why would it touch the console at all? On this image MM is built without
udev, so the only way it learns about ports is `mmcli --report-kernel-event`
called from `hotplug.d`. And the stock tty hotplug script **reports every
tty** — the console UART (ttyS1), and even the virtual consoles `tty3`
through `tty63`.

## I Wrote a Filter. It Did Nothing

I wrote a report filter and dropped it into `uci-defaults`. The console
garbage kept coming. There were two separate reasons, and those two reasons
are what this post is about.

## Reason 1: procd Coldplugs Before rcS

```
kernel uevent replay (STATE_EARLY) → [preinit: where the filter must live]
    → rcS → S10boot (uci-defaults)   ← already too late here
    → S60 dbus → S70 ModemManager → cache replay
    → (without a filter) console UART probe
```

Kernel uevents are replayed in procd's `STATE_EARLY`. By the time
`uci-defaults` (S10boot) or any init script runs, it's already over: the
stock script has processed `ttyS0`/`ttyS1` and pushed them into the event
cache, and MM replays that cache when it starts — probing the console UART.
The only code that runs before procd is exec'd is preinit. So preinit is the
answer.

## Reason 2: uci-defaults Never Updates on This Board

This is the more important discovery. A `uci-defaults` script is `rm`'d
after it runs. But on a persistent overlay, `rm` isn't deletion — it leaves
a **whiteout** in the upper layer:

```
/overlay/upper/etc/uci-defaults/
c---------  1 root  root  0, 0  ...  99-something
```

A config-preserving sysupgrade replaces only the squashfs and leaves the
overlay alone, so the `uci-defaults` shipped by every new image is **masked
forever**. Check a real board and the merged `/etc/uci-defaults` is empty.
Stock OpenWrt doesn't hit this because its sysupgrade formats `rootfs_data`.
Ours doesn't.

And no, this can't be fixed by adding a version marker — **the script that
would read the marker never runs in the first place.** Which yields a
general rule: logic that must be updatable in the field does not belong in
`uci-defaults`. `/lib/preinit` and `/etc/init.d` are squashfs files — they
can't be whited out, and they always come from the new image.

## What the Correct Filter Looks Like

- **Whitelist ttys**: `ttyACM*`, `ttyUSB*`, `cdc-wdm*` — that covers every
  modem we ship.
- **Blacklist net interfaces**: a modem's network interface name is
  unpredictable (`wwan0`/`usb0`/...). Whitelist those and one day your WAN
  dies silently.
- **A no-bus guard**: the modem gets power at S11 and enumerates around 50
  seconds, but MM is S70 — so most port events fire while MM doesn't exist
  yet. Calling `mmcli` at that point burns 6–13 seconds and fails with
  `couldn't get bus`. So the guard caches the event and skips the fork.
  Nothing is lost, because the MM wrapper replays the cache at startup — a
  modem that showed up about 90 seconds before MM was still listed correctly
  by `mmcli -L`.

Bottom line: `mmcli` forks during boot went from 23 to 0. What that means
for boot time gets its own numbers in the next part. The guard hid one trap
of its own, too — `cdc-wdm` control nodes have no hotplug event of their own,
and an early version of the guard swallowed their report, silently demoting
the modem to AT+PPP. That story belongs to the cellular part of this series.

## A Bonus Trap: Partial Marker Matches

A copy of this hotplug script lives in `/etc` — the persistent overlay —
which means old copies outlive firmware updates. So the "already patched?"
marker is a **versioned string** (`QUARK-MM-HOTPLUG-vN`) checked with an
**anchored exact match**. A substring check (`grep -q QUARK-MM-HOTPLUG`)
also matches `-v1` or `-OLD` — and one stale copy really did survive an
"update" that way. Finding it took a while.

## Lessons

- When choosing where a script lives, ask not only "when does this run?"
  but **"does it run again after an update?"** On overlay-based systems,
  deletion isn't deletion — it's masking.
- Auto-probing is as dangerous as it is convenient: the target might be
  your console. We later made our own serial daemon refuse to open the
  console device for exactly this reason.
- A blob of mystery bytes can simply be read. Protocol frames usually tell
  you who they are.
