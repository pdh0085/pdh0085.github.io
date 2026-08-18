---
title: "Dissecting a 190-Second Boot (and Disproving My Own Hypothesis)"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "boot-time", "hotplug", "usb", "profiling", "quark-x1000"]
summary: "The LTE572W takes about 190 seconds to reach init complete. Profiling showed usb-modeswitch eating 25 of them, and the modem's power rail wired to the very end of boot. A code comment gave a 'verified' reason why the modem couldn't be powered earlier — and the experiment said otherwise."
---

<!-- Part 8 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Starting Point

This board takes about 190 seconds to reach `init complete`, and it idles at a
loadavg of about 4. On a single 400 MHz core some of that is unavoidable — but
a loadavg of 4 also means **something is doing work**. To separate the slowness
you're stuck with from the slowness you're paying for, you first have to
measure.

Let me say up front: this is not a "we cut boot from three minutes to one"
success story. It's a record of closing one large hole and knocking down one
inherited hypothesis.

## How I Measured It

Every OpenWrt init script sources `/etc/rc.common`. So one file is enough:
stamp entry and exit into `/dev/kmsg` there, and you get per-script timings on
the same axis as the kernel's own timestamps.

```sh
# near the top of /etc/rc.common
echo "rc: >> $1 $2" > /dev/kmsg
trap 'echo "rc: << $1 $2" > /dev/kmsg' EXIT
```

One trap within the trap: inside rc.common, `$0` is always `/etc/rc.common`.
The actual script path is `$1`. Log `$0` and every line comes out with the
same name, telling you nothing.

## Culprit 1: usb-modeswitch

This was the most satisfying find. The hotplug hook that the `usb-modeswitch`
package installs is effectively one unguarded line:

```sh
# usb-modeswitch's hotplug hook — effectively this one line
/etc/init.d/usbmode start        # no ACTION guard, no DEVTYPE guard
```

No `ACTION` check, no `DEVTYPE` check — so it runs on **every USB uevent**.
And one modem generates ten of those: one device plus nine interfaces. Each
run re-parses the 55 KB, 3,359-line `/etc/usb-mode.json` from scratch, which
costs about a second on this CPU.

Here is what the measurement actually showed:

```
51s        modem appears on USB
51s – 96s  usbmode runs back to back (starving the hotplug queue —
           even unrelated events are delayed 6–9 seconds each)
102s       eth0 comes up
115s       Wi-Fi AP comes up
```

And here's the punchline: every modem we ship enumerates as a modem from the
start. There is nothing to switch from storage mode. Forty-five seconds of
diligent parsing, and the answer was "nothing to do" every single time.

Removing the package saved **25 seconds**. In case someone reinstalls it
later, I left a separate safeguard that installs the missing
`ACTION`/`DEVTYPE` guards, so the regression can't quietly come back.

## Culprit 2: Modem Power at the Very End of Boot

The modem power-up script was START=99. From rail power to USB enumeration
takes about 7 seconds — that's a property of the module itself, not an
ordering problem — and since S99 runs at around 155 seconds, the modem didn't
appear until about 162 seconds. The device that needs the longest lead time
was being powered last.

So just move it earlier, right? Except the script carried this comment:

> Powering the modem early makes it compete with Wi-Fi enumeration on the
> shared EHCI bus, causing repeated `cdc_ether … -71` (EPROTO) errors and USB
> disconnects.

It's plausible. The modem and Wi-Fi really do share one EHCI controller, and
`-71` is a real error. I believed this comment for quite a while myself.

And it was wrong.

## Disproving It

First, a fact check. dmesg shows that **Wi-Fi enumerates at about 3 seconds
into the kernel boot** — before init scripts even exist. So whether the modem
powers up at S11 or S99, racing Wi-Fi *enumeration* is chronologically
impossible. The only late Wi-Fi USB activity is the firmware upload when the
AP is brought up.

The logic was already broken, but I ran the experiment anyway. I deliberately
made the suspected conditions as bad as possible: on a fully booted board, I
re-applied modem power under three conditions.

| Condition | Situation | Result |
| --- | --- | --- |
| (a) | idle | clean enumeration in 9–10 s |
| (b) | 4 CPU load hogs (loadavg 7.65) | clean enumeration in 9–10 s |
| (c) | during an actual `wifi up` firmware upload | clean enumeration in 9–10 s |

All three runs: zero `-71` errors, zero USB disconnects. The failure the
comment warned about would not reproduce even under the worst conditions I
could construct.

So I moved the script to START=11. The modem now appears on USB at **about 50
seconds instead of about 162**, and Wi-Fi is essentially unaffected
(`phy0-ap0` at 90.5 s versus 87.5 s before).

I want to be clear about one thing: whoever wrote that comment wasn't
incompetent. They probably really did see `-71` errors on that board at that
time. But "verified" in a comment usually means **observed once, under those
conditions** — not causation. I carried that belief around for weeks, and
there was only one way to test it: deliberately construct the worst case and
try to reproduce.

## Culprit 3: The Problem I Expected Early Power to Cause

The modem now shows up before ModemManager (S70). That's fine: the hotplug
script from an earlier post in this series caches the kernel events and the
MM wrapper replays them at startup. I verified that a modem appearing roughly
90 seconds before MM is picked up normally by `mmcli -L`.

What the profiling did expose along the way: 23 `mmcli` forks during boot.
`mmcli` is a glib+dbus binary, which costs 6–9 seconds per invocation on this
core — 6–13 seconds when it fails because the bus isn't up yet. A whitelist
filter plus a no-bus guard took that from **23 forks to 0**. The full story is
in that earlier post.

## Not Done Yet

After all of this, `init complete` still lands at about 190 seconds. Overlaid,
the before/after timelines look like this:

```
before:  3s Wi-Fi enum … 51s modem USB … 96s usbmode done
         … 102s eth0 … 115s Wi-Fi AP … ~190s init complete
after:   3s Wi-Fi enum … 50s modem USB (power at S11) … 90.5s Wi-Fi AP
         … ~190s init complete
         (usb-modeswitch removal: −25 s; mmcli forks during boot: 23 → 0)
```

| Metric | Before | After |
| --- | --- | --- |
| Modem appears on USB | ~162 s | ~50 s |
| Wi-Fi AP (`phy0-ap0`) | 87.5 s | 90.5 s |
| `mmcli` forks during boot | 23 | 0 |

The modem shows up 112 seconds earlier and the wasted work is gone — so why is
the total unchanged? The rc.common profile shows the remaining chunks plainly:
S95frr about 30 seconds, S10boot about 16, S80ucitrack about 12, S19firewall
about 11, plus a late-boot stretch where things like sysntpd, mwan3, dnsmasq,
and firewall get invoked over and over. **I have not investigated these yet.**
I have the numbers and not the causes. That's the list for next time.

## Lessons

- Price a hotplug hook per event, not per script. A "one-second script" fired
  by ten events is ten seconds.
- Decisions about start order (START=) belong to experiments, not comments. To
  disprove a claim, deliberately construct the worst case for it and try to
  reproduce.
- A design that avoids contention by starting late is usually a sign that the
  cause was never understood. Understand the cause and the ordering becomes
  free.
