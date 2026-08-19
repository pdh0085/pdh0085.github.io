---
title: "From Sealed Box to In Service in 101 Seconds"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["tr-069", "zero-touch-provisioning", "dhcp-option-43", "openwrt", "acs", "fwa"]
summary: "A factory-fresh router found the carrier's management server on its own, pulled its configuration, and was in service in 101.5 seconds, 100.9 of which were boot time. On the way to that number, we discovered a feature that our docs said we supported but that had never once executed."
---

## The Installer Should Type Nothing

If you sell subscriber lines rather than routers, a device's value isn't a
performance figure. It's how little the installer has to do on install day,
and how visible the device is from the office (the NOC) when something goes
wrong. If a technician is standing in a living room typing an SSID into a
web UI, that labor cost eats the device margin. If the NOC can't see radio
conditions, there's no diagnosis, so the answer defaults to "roll a truck."

The standard for all of this is TR-069 (CWMP). On our 5G FWA routers, the
LS500X and NM500G (Qualcomm IPQ8072, based on OpenWrt 25.12), we implemented
the whole path and timed it second by second, so that the answer to a
carrier's questions is session logs and numbers rather than "yes, supported."

One thing to nail down before any numbers: every TR-069 measurement below
was taken over Ethernet WAN. We have not yet observed a single management
session over the cellular path. That path is unmeasured.

## How a Blank Device Finds Its Management Server

The mechanism fits in three sentences. When the device requests a DHCP lease
on the WAN, it sends the vendor class `dslforum.org` and asks specifically
for option 43. The operator's DHCP server answers with the ACS address in a
sub-option. Since no address is baked into the firmware, the same image
ships unchanged to every operator.

### Except the Feature Had Never Been On

This is the heart of the post. The configuration switch had existed for a
long time. The code behind it was complete. The build script was even set up
to install it. **But the package never copied the executable into the
image.** The binary simply wasn't on the device.

Worse, it failed silently. The code looks like this:

```c
new_url = dhcp_discovery ? (STRLEN(dhcp_url) ? dhcp_url : url) : url;
```

The discovered URL was always empty, so turning discovery on just fell back
to the static URL. Not one line of logging. From the outside, it "worked"
perfectly. Which meant every historical note saying "we tested option 43"
had to be re-read as never actually executed. The switch existed, nothing
was wired behind it, and even the failure looked like success.

### On by Default, Latched After First Contact

Putting the binary into the image wasn't the end of it. If discovery is on
by default, whoever controls DHCP on the WAN side can designate this
device's management server. So we made a rule: the moment the device
successfully connects to an ACS for the first time, it locks. Discovery is
disabled, the learned value is erased, and only the confirmed URL remains.
From then on the device talks only to the ACS that provisioned it. The
exposure window is the initial installation, not the device's lifetime.

Even that wasn't a single flag. After latching, two variables still held the
same URL, and through the retry path that meant a long ACS outage could
reopen discovery, even years later. The missing step was clearing the
learned value.

The verification is worth writing down too. We kept the ACS dead for 394
seconds and watched that discovery never reopened. We also checked that the
test wasn't spinning in place, i.e. that the daemon was actually alive and
cycling through retries, because a dead daemon produces the exact same
"nothing reopened" result while proving nothing.

We chose the default's condition deliberately as well: not a "run once"
flag, but **"only when the ACS address is empty."** If discovery ever ran
again on a device already talking to an ACS, the next DHCP lease could swap
out that device's management server.

And one more boundary, stated precisely: option 43 was measured in the
Ethernet WAN installation scenario. A cellular-only installation would
need the URL provisioned at the factory.

## The Measurement

From power-on, through receiving the configuration the operator had queued
in advance (SSID, Wi-Fi key, disabling the setup wizard), to being in
service. We rehearsed it twice; both runs were identical.

| Segment | Measured |
|---|---|
| Cold start → first ACS contact (Inform) | **100.9 s** |
| Cold start → provisioning complete | **101.5 s** |
| Config received and applied after contact | **0.5 s** |

Power to service in under two minutes, and almost all of it is boot time.
The provisioning itself takes half a second.

The first lines on the ACS console summarize the whole picture:

```
inform events=["0 BOOTSTRAP", ...]
send_rpc SetParameterValues -> status=0     # SSID + Wi-Fi key
send_rpc SetParameterValues -> status=0     # disable setup wizard
send_rpc GetParameterValues -> Device.WiFi.SSID.1.SSID = CarrierWiFi
```

`0 BOOTSTRAP` is the device saying "this is the first contact with a
management server in my life."

We verified over the air, not in the UI. Confirming that a configuration
value changed is not enough, so we confirmed the radio actually restarted
and was beaconing that SSID, by scanning from a second device sitting next
to it. This project has produced several cases of "the status light is
green but nothing is actually passing."

## Remote Upgrade

When the ACS issues a download command, the device writes the image to the
inactive slot, reboots, and in the first session on the new firmware sends
the success report (TransferComplete, FaultCode 0). About 92 seconds from
command to report. A corrupted image is bounced with an error in 3.9
seconds, without touching the flash.

One thing we don't hide: images before a certain version never sent the
completion report at all. The report record was written to volatile
storage, and the reboot that the upgrade itself triggers wiped it. The
upgrade succeeded; the ACS just never found out. It's fixed now, and when
we demo on an older unit, we say so.

## Can the Office Call the Device?

To reach a device at the moment a customer calls in, the ACS has to be able
to call the device. We measured both paths.

Over HTTP, a firewall pinhole is generated automatically, open only to the
ACS address. Call to established session: 0.58 s. The interesting
verification here isn't the success, it's the refusals. A call from any
other address is rejected, a wrong password gets 401, and if no ACS address
is configured, the rule doesn't exist at all.

Over XMPP (TR-069 Annex K), the device holds one outbound session, so zero
inbound ports are open. Call to session: 0.65 s. That number, though, is
from a test server. Against a commercial ACS (GenieACS) we have confirmed
the session stays up across the internet with certificate verification
passing; the final step, an inbound call pushed from that side, is still
pending. On CGNAT (most 5G lines) this is the path that matters, so we keep
that distinction exact rather than blurring it.

This path surfaced a defect of its own: the XMPP client crashed on TLS
handshake failure. A null-dereference bug in the library, fixed in our fork
of the package.

One line we drew on purpose: **the admin password cannot be set through the
ACS.** By design it is not writable remotely. Whoever compromises the
management channel does not get the admin account thrown in. The initial
credentials on the shipping label are generated per unit, and that is all
we will say about them.

## The Blacklist That Silenced Telemetry Forever

To prevent "can't diagnose, so roll a truck," the NOC needs RSRP/RSRQ/SINR,
operator, band, and attach state. We exposed all of it through the TR-069
data model, with six items carried in the periodic report.

At some point, though, the modem fields started coming back empty. The cause
was a permanent blacklist in a library underneath the data model. Ten
consecutive cycles of slow modem responses, and it kills that data source
forever; after that it quietly answers "no modem" defaults, permanently.
A service restart doesn't clear it. And it poisoned not just query responses
but the periodic reports: 5 of 43 test sessions went out with a blank IMEI.

In other words, one slow 50-second window could silence a device's cellular
telemetry for the rest of its life. Behind CGNAT, where the periodic report
is effectively the only monitoring path, that is quiet blindness.

The fix replaces the permanent ban with a 30-second grace period and retry.
Worst case, a 5-second delay every 30 seconds, in exchange for no permanent
loss. We drilled the whole cycle on real hardware (reproduce it, fix it,
watch it recover) and verified it after a factory-clean flash: the stronger
check, proving the fix is in the image, not just in the source tree.

## The Numbers

All measured over Ethernet WAN. The cellular path remains unmeasured.

| Item | Value |
|---|---|
| Zero-touch: cold start → in service | **101.5 s** (first contact 100.9 s, provisioning 0.5 s) |
| ACS remote upgrade | **~92 s**, completion report received |
| Corrupt image rejection | **3.9 s**, flash untouched |
| ACS call → session (HTTP, open only to ACS address) | **0.58 s** |
| ACS call → session (XMPP, zero inbound ports) | **0.65 s** (test server) |
| Periodic report pollution (during the bug) | 5 of 43 sessions with blank IMEI → 0 after the fix |

## What We Keep

Most of what this project taught us is about verification. A silent
fallback disguises an unverified feature as a working one; if flipping the
switch does nothing behind the scenes, that fact has to show up in a log.
When you enable a convenience feature by default, design its exposure
window with it. The security decision isn't on versus off, it's how long
the window stays open, and you only get to claim it's latched after
watching it stay latched for real time (394 seconds, in our case).

Verify against the final effect, not the setting: the SSID actually in the
air rather than the string in the UI, the image after a factory-clean flash
rather than the source tree. Defensive logic that remembers failure forever
eventually becomes the failure; grace and retry is almost always cheaper.
And measurement conditions are part of the number. Writing "measured on
Ethernet" and "test-server figure" where true is what makes the rest of
the numbers believable.
