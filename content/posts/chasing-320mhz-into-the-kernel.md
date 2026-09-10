---
title: "Chasing 320MHz Into the Kernel"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [wifi7, mt7927, mac80211, cfg80211, ftrace, kprobe, linux, debugging]
summary: "The AP, the regulatory rules and the adapter all said 320MHz, and the negotiation still stopped at 160. The kernel message blamed regulatory; every tool I had said there was no restriction on the channel. Neither was true. The culprit was a country restriction table living in the firmware, and the flag it set was invisible from user space."
---

This started as a quick check: does the new Wi-Fi 7 adapter work on Linux,
and what does iperf3 say. The adapter came up in five minutes. The rest of
the day went to everything stacked on top of it. To give away the ending,
nothing was wrong with the adapter.

I've left my wrong conclusions in rather than editing them out. The
interesting part of an elimination hunt is usually where it slips, not
where it lands.

The device under test is an MT7927 (Filogic 380). Host is Ubuntu 26.04 on
kernel 7.0.0-31-generic; the other end is an OpenWrt AP running a 3-link
MLD (2.4G ch6 20MHz / 5G ch100 160MHz / 6G ch37 320MHz).

## If 6GHz Isn't There, Look at the Regulatory Domain First

`lspci` saw it, `mt7925e` bound to it, the interface came up. And not a
single 6GHz channel showed up in a scan.

```
$ iw reg get
global
country 00: DFS-UNSET

$ iw phy phy0 info | grep 6135
    * 6135.0 MHz [37] (disabled)
```

With the domain set to world (00), 6GHz is locked out entirely. One
`iw reg set KR` brought the channels back at 15 dBm, and only then did the
AP's 6GHz link appear in the scan. That's the place to look before you
start suspecting antennas or drivers.

Then came the plumbing. The session was remote over SSH, so polkit refused
to let me drive NetworkManager, and wired and wireless sat on the same
subnet, which made routing pick the wrong one. Specifying a source IP was
not enough; the interface has to be pinned as well.

```
$ iperf3 -c 10.1.1.1 -B 192.168.10.212%wlp3s0 -t 5
...
wlp3s0 TX delta: 489 MB   enp1s0 TX delta: 0 MB
```

Skip that byte-counter check and you will happily report wired throughput
as a wireless result. `-B <ip>` alone changes the source address while the
route still goes out the wire.

## The First Numbers Were Low Because the Driver Wasn't Inbox

The first run gave 497 Mbps up and 705 Mbps down against a PHY rate of
2882 Mbps, so roughly 20% efficiency. The wired path from AP to server ran
at 2.36 Gbps, so the bottleneck really was on the wireless side. I
eliminated TCP multistream, powersave, signal quality and host CPU in turn.
Four streams were not faster than one (488 against 497), the average was
-37 dBm with `tx failed 0`, and the retry rate was 2.1%. Congestion control
or window sizing would have shown up as a gain on multistream.

Then `modinfo` gave me pause.

```
$ modinfo mt7925e | head -2
filename: /lib/modules/7.0.0-31-generic/updates/dkms/mt7925e.ko.zst

$ dkms status
mediatek-mt7927/2.11, 7.0.0-31-generic: installed
```

Not the inbox driver at all, but a vendor DKMS package, and v2.11 dated
2026-04-15 when v2.14 was current. Moving to 2.14 took uplink to 705 Mbps.
What actually changed was a transmit power reporting bug: `txpower` had
been frozen at 15.00 dBm in 2.11 and came up to the regulatory limit of
23 dBm in 2.14. That explains why switching the domain from KR to TW, which
raised the limit by 8 dB, had moved uplink by nothing at all. The ceiling
went up; the actual output never followed.

Downlink wandered between 637 and 1190 Mbps across the whole session. With
the signal unchanged and only the receive MCS drifting between 8 and 5,
that reads as interference, not as a fix. Uplink was the only number that
moved for a reason.

## Every Condition Says 320, the Negotiation Says 160

This is where the day went. Everything I could check pointed at 320MHz.

| Condition | Measured |
| --- | --- |
| AP operating width | 320 MHz, center 6105 (actual chandef, not config) |
| AP EHT Operation | Channel Width: 320 MHz |
| AP EHT Capabilities | 320MHz in 6GHz Supported |
| Regulatory rule (TW) | 5945-6425 @320 MHz |
| Adapter capability | 320MHz in 6GHz Supported, MCS Map BW=320 |
| **Negotiated** | **160 MHz** |

The kernel left this behind:

```
wlp3s0: regulatory prevented using AP config, downgraded
```

I took that as evidence and concluded it was a regulatory problem. Wrong.
Reading the source later, here is where the line comes from:

```c
if (chanreq->oper.width != ap_chandef->width || ap_mode != conn->mode)
        link_id_info(sdata, link_id,
                     "regulatory prevented using AP config, downgraded\n");
```

It prints whenever the final width differs from the AP's, whatever the
reason. A message saying "regulatory" does not make regulatory the culprit.
Kernel messages are strings somebody typed, same as any other.

## What the Tools Don't Show You

To see whether the channel carried something like `no 320MHz`, I went
through `iw phy info`. Nothing there. So I ruled regulatory out. Also
wrong.

As it turns out, `iw` cannot print that flag at all. The only channel flag
it renders as a string is `no IR`. So I wrote a tool that parses nl80211
directly and looked again, and got the same answer, this time for a
different reason: the kernel never exports the flag over nl80211.

```
raw attributes for 5955 MHz:
  FREQ, MAX_TX_POWER, NO_HT40_MINUS, INDOOR_ONLY, OFFSET
  (no NO_320MHZ)

chan->flags inside the kernel at the same moment:
  0x80220 = NO_320MHZ | INDOOR_ONLY | NO_HT40_MINUS
```

Invisible from user space, set inside the kernel. "I looked with the tool
and it wasn't there" is not evidence. Whether the tool can see the thing
has to be established first. I fell into that twice, once with `iw` and
once with a tool I had written myself.

## Going Inside

If it can't be seen from outside, look from inside. ftrace first, to catch
what mac80211 decided.

```
drv_add_chanctx: ap(6135MHz, width:13, center 6105)      <- AP parsed as 320MHz
                 chandef(6135MHz, width:5, center 6185)  <- actually used, 160MHz
```

Width 13 is `NL80211_CHAN_WIDTH_320`, 5 is 160MHz. The parse was correct
and something downstream cut it. Next, a kprobe on the return values of the
decision functions.

```
cfg80211_chandef_valid (320MHz)   ret=1   valid
cfg80211_chandef_usable(320MHz)   ret=0   rejected
```

And there I stalled. The 320MHz branch of `_cfg80211_chandef_usable()` has
only a handful of ways to fail, and every value I had measured satisfied
the passing conditions. Observation and source disagreed.

So I disassembled the module, took addresses for each branch inside the
function, and put kprobes on them.

```
br320 -> bandok -> bittest -> found -> retfalse

subchannel loop:
  freq=5955000  flags=0x80220  prohib=0x80041  -> no pass
  intersection 0x80000 = IEEE80211_CHAN_NO_320MHZ (BIT 19)
```

It dies on 5955 MHz, the first subchannel of the 320MHz block. cfg80211 puts
`NO_320MHZ` into the prohibited mask when checking a 320MHz width, the
channel carries that bit, and it returns false immediately. mac80211 then
drops one step to 160MHz and prints the misleading line from earlier.

Culprit found: every 6GHz channel had `NO_320MHZ` set. Except that nothing
in the regulatory rules or the driver's channel table sets it.

## The Culprit Was CLC, in the Firmware

One path left. The mt7925 family has a feature called CLC, Country Location
Control: the driver reads a per-country restriction table carried in the
firmware and overwrites channel flags with it. There's a module parameter
to turn it off.

Read the source and CLC looks like it only sets `NO_EHT` (BIT 20) and
`DISABLED` (BIT 0). The bits didn't match, so I decided it wasn't CLC. That
was my third wrong call. It sets `NO_320MHZ` (BIT 19) too.

```
$ modprobe mt7925_common disable_clc=1

channel 37 (6135 MHz), width: 320 MHz, center1: 6105 MHz
rx bitrate: 3459.3 MBit/s 320MHz EHT-MCS 8 EHT-NSS 2
```

One line, and it opened. Regulatory domain unchanged, AP config unchanged.

| Measurement | 160MHz | 320MHz |
| --- | --- | --- |
| TCP uplink | 705 Mbps | 1080 Mbps |
| TCP uplink, 4 streams | 488 Mbps | 1690 Mbps |
| TCP downlink | 637 Mbps | 1380 Mbps |
| UDP downlink loss | 21% | 0.06% |

The loss figure is the one that stands out. It had simply been unable to
push the traffic through the width it had.

This needs stating plainly: `disable_clc=1` turns off a regulatory
compliance mechanism. It belongs in lab verification only, and in a real
deployment you have to confirm separately that the region permits 320MHz on
6GHz. Every 320MHz number above was taken under that condition.

## MLO Had the Same Root

The AP runs a 3-link MLD normally, and the STA associated on 6GHz alone. I
found where the driver declares its number of simultaneous links as one,
changed it, and nothing happened at all. So I read the gate above it with a
kprobe.

```
chip_cap = 0x0000000000000000
set bits: (none)
```

`chip_cap` is filled from the `MT_NIC_CAP_CHIP_CAP` TLV the firmware sends.
On MT7927 it arrives empty, and every capability gate downstream stays shut.

| Bit | Capability | Effect |
| --- | --- | --- |
| BIT(4) | 11D_EN | regulatory goes through CLC instead of `regulatory_hint()` |
| BIT(8) | MLO_EN | `WIPHY_FLAG_SUPPORTS_MLO` never gets set |
| BIT(9) | MLO_EML_EN | EML disabled |

The line I had edited sits below that gate, so it was dead code that never
ran. No wonder changing it did nothing.

Forcing the gate open brought all three links up, each with its own local
address, and the AP's debugfs agreed.

```
wlp3s0: [link 0] local 36:2e:4a:82:fb:df <- AP 06:0c:43:26:60:10  (2.4GHz)
wlp3s0: [link 1] local e6:e0:10:0c:19:3a <- AP 42:0c:43:26:60:10  (5GHz)
wlp3s0: [link 2] local 6a:e0:6c:f5:9a:4a <- AP 00:0c:43:26:60:10  (6GHz, assoc)
```

The datapath was another matter. Uplink rose to 1330 Mbps while downlink
collapsed from 1380 to 371 Mbps and retries climbed to 718. MT6639 shares
RF1 between 5GHz and 6GHz, so binding both links at once appears to tangle
the receive path. That lines up exactly with the state of the driver
without the rest of the fixes upstream listed as conditions for bringing
MLO back. Opening the capability bits was never going to be enough on its
own.

## What I Took Away

Nothing was wrong with the adapter. The MT7927 hardware is capable of both
320MHz and MLO, and the code for both is present in the kernel and the
driver.

What blocked them was what the firmware reported. 320MHz was blocked because
CLC stamped `NO_320MHZ` onto every channel; MLO was blocked because
`chip_cap` arrived as zero and shut the capability gates. Both are at the
firmware level, and they share a root.

And none of that was visible from user space. Not through `iw`, not through
the nl80211 tool I wrote myself. Until ftrace and kprobe let me look inside
the kernel, I was stuck at "the AP, the regulator and the adapter all agree,
so why doesn't it work".

Collecting the wrong calls:

| What I concluded | What was true |
| --- | --- |
| Driver doesn't implement 320MHz | It does, and it wasn't part of this decision |
| Regulatory domain problem | Fooled by the message text; the rules were innocent |
| No restriction flag on the channel | The tools simply cannot show that flag |
| Not CLC, the bits don't match | CLC was the culprit |
| AP service died from FD exhaustion | My own routing change caused it |

Four of the five were cases of believing I had checked something when I
hadn't. Elimination only works when each candidate can actually be
observed, and the fact that my instruments couldn't observe this one was
the last thing I learned, not the first. Next time I plan to establish what
a tool is able to print before I trust what it doesn't.

The whole reproduction is these four commands.

```sh
# open 6GHz
sudo iw reg set TW

# open 320MHz (lab only)
sudo modprobe mt7925_common disable_clc=1

# force the measurement onto wireless
iperf3 -c <server> -B <wifi-ip>%wlp3s0 -t 10 -P 4

# look inside the kernel
echo 'p:usa cfg80211_chandef_usable width=+8($arg2):u32 prohib=$arg3:x32' \
  > /sys/kernel/tracing/kprobe_events
```

Test setup was an MT7927 (Filogic 380) on Ubuntu 26.04 with kernel
7.0.0-31-generic and mediatek-mt7927 DKMS 2.14-6, against an OpenWrt AP on
kernel 6.18 running 3-link MLO. Every figure is a single 10-second run, and
downlink drifted between 637 and 1390 Mbps over the session with
interference.
