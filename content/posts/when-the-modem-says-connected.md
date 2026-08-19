---
title: "When the Modem Says 'Connected'"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "modemmanager", "qmi", "lte", "quark-x1000", "debugging"]
summary: "The modem reported connected, and wwan0 had an IP address and a default route. Not a single packet went out. Two silent failure modes of an LTE modem on a 400 MHz single-core board, both with perfectly green status indicators, both only detectable by pushing real traffic."
---

<!-- Part 9 of the LTE572W Yocto-to-OpenWrt porting series -->

## Background: There Is No Single Modem

The LTE572W ships with several modem models. The Quectel EG25-G and EC25-AU
are `option` + `qmi_wwan` modules, so they expose `ttyUSB*`, `cdc-wdm0`, and
`wwan0`. The Cinterion PLS83-EP / PLS83-W are `cdc_acm` + `cdc_ether`
modules: `ttyACM*` and `wwan0`, with no cdc-wdm at all.

That means code on this board must never depend on tty names or USB VID/PID.
The one fact we allow ourselves to depend on is that **the modem is always
the USB device on EHCI port `1-1`** (Wi-Fi is `1-2`). Whatever module is
fitted, that doesn't change. It's a fact of the PCB.

This post is a record of two failures in that cellular stack. They share one
property: in both cases, every status indicator looked perfectly healthy.

## Part 1: The Silent Demotion from QMI to AT+PPP

### The Symptom Was That There Wasn't One

At some point the modem started attaching fine. Data flowed. But the CPU was
strangely busy, and there wasn't a single error in the logs.

The cause was a ModemManager fallback. When MM can't see a QMI control port,
it **silently falls back to AT + PPP**. PPP is userspace HDLC, which is a
disaster on a 400 MHz single core. And because the modem still connects, the
failure hides itself.

### Why the QMI Port Went Missing

`cdc-wdm` has no hotplug event of its own. It's the QMI/MBIM *control*
node: it gets discovered through the network interface's sysfs and reported
separately, under the `usbmisc` subsystem. In an earlier post in this series
I described a no-bus guard we added to the hotplug scripts (if MM isn't up
yet, cache the event and skip the `mmcli` fork). The first version of that
guard did its `exit 0` *before* the code path that reports usbmisc devices.

The result: `cdc-wdm0` was never reported to MM, so MM marked the interface
`wwan0 (ignored)`, which meant demotion to AT+PPP. Not one line of error
output.

### How to Tell Which Path You're On

This is the most practically useful part of the post. Two lines of
`mmcli -m N` output tell you which path the modem attached over.

| State | primary port | manufacturer | Where the value comes from |
| --- | --- | --- | --- |
| Healthy (QMI) | `cdc-wdm0` | `QUALCOMM INCORPORATED` | QMI DMS response |
| Demoted (AT+PPP) | `ttyUSB2` | `Quectel` | AT `+CGMI` response |

Same modem, different manufacturer string. Where the string comes from tells
you which path you're on: ask over QMI and the chipset vendor answers; ask
over AT and the module vendor answers.

```
# check that you're on the healthy QMI path
mmcli -m 0 | grep -E 'primary port|manufacturer'
   primary port: cdc-wdm0
   manufacturer: QUALCOMM INCORPORATED
```

One more thing: this bug is invisible on the PLS83, because that module has
no cdc-wdm to lose. It only shows up with a Quectel module fitted — a bug
that requires a hardware configuration change to become observable. A
classic trap for a product that ships multiple module variants.

## Part 2: The Half-Failure of APN Auto-Profiling

### Every Surface Signal Was Green

We booted with a SIM from a domestic carrier. `mmcli` said `connected`.
`wwan0` got an address (a `102.x` one that even looked like a public IP),
and a default route appeared. On paper, perfect.

But no data flowed at all. ICMP was dropped, and every TCP connect came back
with `EPERM`.

```
# the signature of "connected" with no data path
ping -I wwan0 8.8.8.8      → 100% loss
curl https://...           → connect: Operation not permitted (EPERM)
```

### Cause and Fix

For this SIM, APN auto-profiling silently half-fails. The modem attaches,
but to a bearer that passes no data. Setting the carrier's correct APN
explicitly made it immediately reattach to a real bearer (`10.x/23`), and
ping, HTTP, and HTTPS all worked.

The rule that came out of this:

> "The interface is up and has an IP" **proves nothing.**
> A connection is one successful real TCP request.

The same rule applies directly to health-check design. A health check that
judges uplink health by link state or the presence of an IP will classify
this exact state as healthy.

### The Product Decision

We did not put that APN into the firmware defaults. The product is not tied
to any one carrier; the user sets the APN in the UI. Baking one carrier's
convenience into the defaults would manufacture exactly this kind of silent
failure for every other carrier.

## Part 3: The Time the Documentation Was Wrong

Our internal records said the EC25-AU "fails on a certain domestic carrier
with `USIM application state illegal` / `sim-missing`." We tried it on an
actual shipping board with that carrier's SIM, and **it did not reproduce**.
The EC25-AU registered and attached normally, got a `10.x` address over the
`cdc-wdm0`/QMI path, and passed data (`ping 8.8.8.8 -I wwan0`, 0% loss).

We corrected the record but didn't delete it. A wrong record shouldn't be
erased; it should be kept alongside the reason it was overturned. That
practice gets its own post later in this series.

For contrast: with no SIM inserted at all, `mmcli` reports
`failed / sim-missing` and `wwan` never comes up. That's not a lie, that's
correct behavior. The problem was always the states that *looked* healthy
and weren't.

## Lessons

- In a cellular stack, the only trustworthy signal is end-to-end traffic.
  `connected`, an IP address, a default route: we confirmed twice that all
  of them can be false positives.
- A product that supports multiple modules will inevitably grow bugs that
  only one module can reveal, so code should depend on invariants (here,
  the USB port position `1-1`) rather than on models.
- Fallbacks look like kindness, but a silent fallback hides bugs. If the
  QMI-to-PPP demotion had logged even one line, we would have found it far
  sooner. Related habit: read diagnostic output for where a value comes
  from, not just what it says. Which protocol answered the `manufacturer`
  query is what told us which path we were on.
