---
title: "From Linux 3.2 to OpenWrt 25.12: Porting the MA-2080X Cellular Router"
date: 2026-10-29T09:44:43+09:00
draft: false
tags: [openwrt, embedded, am335x, u-boot, firmware-update, reliability, debugging, ucode]
summary: "About seven weeks moving the MA-2080X industrial cellular router off its vendor Linux 3.2 firmware and onto OpenWrt 25.12.5: one irreversible change at a time, feature parity with the vendor firmware, the bugs that failed silently, and why the result will not ship."
aliases: ["/posts/openwrt-25-12-on-a-legacy-am335x-router/"]
---

## Why I did this

Over about seven weeks in autumn 2026 I moved the Maxon / RF Industries MA-2080X cellular router off its vendor Linux 3.2 firmware and onto OpenWrt 25.12.5. The result, OS-DHP 2.2.6, runs on two bench units with A/B updates, a web UI, TR-069 / TR-369 management and almost every feature the vendor firmware had.

The MA-2080X is a modular industrial router: a TI AM3352 board, an eMMC, a Sierra Wireless cellular module, two SIM slots, serial ports, digital I/O and GNSS. Its stock firmware still ran a 3.2 kernel from more than a decade ago. That means no current security fixes, an old TLS stack and no realistic way to add modern management protocols.

The product will not ship with it, so this post is the record: what the port involved, what bit me, and what I would do the same way again.

## The starting point

I started with a 2015 schematic, the vendor's Linux 3.2 source tree, and two bench units. Where the schematic and the vendor board file disagreed, the board file won: it is what actually shipped.

What the hardware turned out to be:

- **SoC and memory:** TI AM3352 (Cortex-A8), 512 MB RAM, an eMMC as the only storage. Every GPMC pin is reused for eMMC data, the second RGMII port or GPIO.
- **Ethernet:** two separate EMACs, not a switch. OpenWrt's omap target already moves am335x boards back to the legacy `cpsw` driver, so the device tree uses two plain MAC ports.
- **Cellular:** a Sierra Wireless module on the expansion socket, two SIM slots behind a mux, SIM presence read through an ADC voltage band. GNSS comes from the module on production boards.
- **Field I/O:** serial ports used as a TCP/serial gateway, digital inputs and outputs, analogue inputs, status LEDs and a reset button.

The eMMC carries the vendor's partitions: MLO and U-Boot in p1/p2, a vendor "nvram" settings store in p3, the U-Boot environment in p4, two kernel slots in p5/p6, and a media partition in p7.

The vendor firmware runs entirely from RAM (`root=/dev/ram0`), packed as an Android boot image inside a small vendor container with a 16-bit checksum. I reverse-engineered that container and reproduced it byte for byte. Its packer even wrote two header fields from uninitialised stack, so they differed on every run.

## Bring-up: OpenWrt first, the bootloader later

The plan was to change one irreversible thing at a time, so a unit in the field would never need a site visit.

1. **Phase 1, OpenWrt on the vendor bootloader.** The vendor U-Boot (2011.09) boots an Android boot image with a fixed load address and command line. I built OpenWrt for `target/linux/omap` as a kernel plus initramfs in exactly that format, so the first migration is a single upload through the vendor's own web page. The first OpenWrt boot on the board was on 11 September.
2. **Phase 2, a new U-Boot on the bench.** The vendor bootloader had no fallback at all: a bad image dropped it to a prompt for ever, and its SPL turned the watchdog off. I ported U-Boot 2025.10 to the board, first by chainloading it from the vendor prompt, then from a microSD card with two bare pads bridged to change the boot order. That SD path became the proven brick-recovery route before anything touched MLO in the eMMC.
3. **Phase 3, the bootloader from the web UI.** Every OS image carries its matching MLO / u-boot.img pair. An install job checks the board, the partition table and the current checksums, writes p2 then p1, reads each back and restores the old one on a mismatch.

With the new U-Boot the two kernel slots become a real A/B pair. Sysupgrade writes the inactive slot, verifies it and only then flips `ab_primary`. U-Boot counts boot attempts; after three failures it boots the other slot, and the hardware watchdog turns a hung kernel into such a failure. A boot that reaches a healthy userspace clears the counter.

<figure style="margin:1.5em 0">
<svg viewBox="0 0 760 484" role="img" aria-label="A bad image falls back to the other slot by itself" font-size="13" style="width:100%;height:auto;font-family:inherit">
<defs><marker id="ab-arrow-en" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0 0L10 5L0 10z" fill="var(--secondary)"/></marker></defs>
<text x="24" y="34" font-size="15" font-weight="600" fill="var(--primary)">A bad image falls back to the other slot by itself</text>
<g fill="none" stroke="var(--secondary)" stroke-width="1.25"><path d="M208 100H288" marker-end="url(#ab-arrow-en)"/><path d="M472 100H544" marker-end="url(#ab-arrow-en)"/><path d="M636 126V174" marker-end="url(#ab-arrow-en)"/><path d="M552 210H472" marker-end="url(#ab-arrow-en)"/><path d="M636 246V294" marker-end="url(#ab-arrow-en)"/><path d="M380 236V284" marker-end="url(#ab-arrow-en)"/><path d="M544 320H464" marker-end="url(#ab-arrow-en)"/><path d="M296 320H208" marker-end="url(#ab-arrow-en)"/><path d="M380 356V404" marker-end="url(#ab-arrow-en)"/><path d="M472 430H744V100H728" marker-end="url(#ab-arrow-en)"/></g>
<rect x="24" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="116" y="96" text-anchor="middle" fill="var(--primary)">Write the spare slot</text><text x="116" y="112" text-anchor="middle" fill="var(--primary)">and verify it</text>
<rect x="288" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="96" text-anchor="middle" fill="var(--primary)">Flip ab_primary,</text><text x="380" y="112" text-anchor="middle" fill="var(--primary)">arm the boot counter</text>
<rect x="544" y="74" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="96" text-anchor="middle" fill="var(--primary)">U-Boot counts</text><text x="636" y="112" text-anchor="middle" fill="var(--primary)">one more attempt</text>
<polygon points="552,210 636,174 720,210 636,246" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="214" text-anchor="middle" fill="var(--primary)">Over 3 attempts?</text>
<rect x="288" y="184" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="214" text-anchor="middle" fill="var(--primary)">Boot ab_primary</text>
<rect x="544" y="294" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="636" y="324" text-anchor="middle" fill="var(--primary)">Boot the other slot</text>
<polygon points="296,320 380,284 464,320 380,356" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="324" text-anchor="middle" fill="var(--primary)">Userspace healthy?</text>
<rect x="24" y="294" width="184" height="52" rx="8" fill="#3b82f6" fill-opacity="0.12" stroke="#3b82f6" stroke-width="2"/><text x="116" y="316" text-anchor="middle" font-weight="600" fill="var(--primary)">Boot confirmed,</text><text x="116" y="332" text-anchor="middle" fill="var(--primary)">counter cleared</text>
<rect x="288" y="404" width="184" height="52" rx="8" fill="none" stroke="var(--secondary)" stroke-width="1.25"/><text x="380" y="426" text-anchor="middle" fill="var(--primary)">Watchdog resets</text><text x="380" y="442" text-anchor="middle" fill="var(--primary)">the board</text>
<text x="512" y="202" text-anchor="middle" font-size="11.5" fill="var(--secondary)">no</text>
<text x="646" y="274" font-size="11.5" fill="var(--secondary)">yes</text>
<text x="252" y="312" text-anchor="middle" font-size="11.5" fill="var(--secondary)">yes</text>
<text x="390" y="384" font-size="11.5" fill="var(--secondary)">no, it hangs</text>
<text x="608" y="422" text-anchor="middle" font-size="11.5" fill="var(--secondary)">reset, then the next attempt</text>
</svg>
<figcaption style="text-align:center;font-size:0.85em;font-weight:400;color:var(--secondary)">A/B update and fallback · U-Boot boot counter with bootlimit 3</figcaption>
</figure>

No human is needed between a bad upgrade and a working unit: a hang becomes a watchdog reset, and after three failed attempts U-Boot boots the slot that worked before.

One more problem came from the RAM root. OpenWrt's normal overlay needs `pivot_root`, which the kernel refuses on an initramfs. So a preinit hook mounts a new ext4 `config` partition and overlays `/etc` alone. On a unit fresh off the vendor firmware, an earlier hook recognises the factory partition table by its checksum, shrinks the media partition, and writes a new GPT from bytes computed ahead of time. Then an importer reads the vendor's p3 settings store (more on that below).

The first real bootloader install on a unit still found a bad bug. The write and read-back verify passed, but the unit never autobooted again because a hand-edited patch had a wrong hunk header, so one boot variable was never defined. The serial console and a careful write order turned it into a fix, not a dead unit.

## Feature parity with the vendor firmware

A new base system is worthless to an operator if the router stops doing its job. So after bring-up most of the time went into the features the vendor firmware had, built on dhplab, a shared OpenWrt application framework I also use for other products. In October a survey of the vendor source found what OS-DHP still lacked; 14 feature branches closed that gap, and each went through a review before it was merged.

| Area | What OS-DHP does |
| --- | --- |
| Cellular | Dual SIM with failover (no signal, failed connections, roaming, data limit), per-SIM APNs, uplink switching between Ethernet and cellular, data usage per SIM, network time from the cell network |
| Serial | TCP / UDP serial gateway with the vendor's AT-command "IPStack" set, dial emulation, AT over IP, back-pressure instead of dropped connections |
| Industrial protocols | Modbus (including the vendor's FTP upload formats), DNP3 outstation with dial-out and unsolicited reports |
| I/O and alarms | Digital and analogue I/O, alarm events to SMS and e-mail groups, SMS commands (status, SIM switch, APN, reboot and more), user scripts on events |
| GNSS | Position from the cellular module, NMEA and tracking reports to servers, geofences |
| Networking | IPsec (including IKEv1 aggressive mode), OpenVPN with a certificate store, PPTP, VRRP, SNMP with the vendor's MIB, content filtering, scheduled reboots, power saving |
| Management | Web UI, TR-069, TR-369 / USP over WebSocket and STOMP, RADIUS login, log export, signed plugins |

The settings importer is what makes this a migration rather than a reinstall. It reads the vendor settings on the first boot and writes the matching OpenWrt configuration, so a unit comes up after the upload with its LAN, APNs, serial ports, I/O and alarm contacts as they were. The work included a 236-page user manual for the new firmware.

## Things that bit

Most of the lost days came from things that failed silently. These are the ones worth passing on.

**Hardware**

- **Signal names lie.** Several GPIOs (SIM mux, modem power, RF enable) had the opposite polarity to their names. The vendor code and a measurement were right; the schematic net names were not.
- **One part can wedge a whole bus.** On development boards the discrete GNSS receiver held I2C0, so the power-management IC behind it became unreachable. The fix was to keep the receiver in reset until the I2C users had probed, and to let CPU frequency scaling switch itself off when the PMIC never appears (720 MHz instead of 1 GHz).
- **Analogue thresholds depend on the board revision.** SIM presence is an ADC voltage band, and the bands differ between PCB revisions. A wrong revision setting gives no error, only wrong answers.

**Software**

- **ucode compiles regexes with REG_NEWLINE by default.** In ucode 2026.04.24, unless a regex carries the `s` flag, `^` and `$` match at every line, so an anchored validator accepts `"ok\nevil"`. A newline then passed into a shell command line. An audit fixed every validator and sanitiser in the framework and the product layer.
- **Stock sysupgrade refuses backups on a RAM root and still exits 0.** On this board the root is always a ramdisk, so "Download settings" produced an empty file on every build until 2.2.4. Nothing reported an error; a restore test found it.
- **A non-blocking socket is not a reliable one.** The serial gateway dropped a TCP connection on `EAGAIN`. When a peer paused, about 45 KB later the connection closed and the rest of the stream was parsed as AT commands. Bounded queues with UART back-pressure fixed it.
- **The vendor had bugs worth not copying.** Its geofence mixed up degree formats and never worked, its power-save setting put the serial transceiver into loopback, and its DNP3 class masks did not map one-to-one. The importer translates these instead of copying them.
- **The build system can drift without telling you.** Run through a pipe into a log, `./scripts/feeds update` printed "failed." while the build script saw exit status 0, and a `make defconfig` after a feed change once reset a vendor prefix option. A build guard now refuses any build whose `.config` or feed pins differ from the recorded ones.
- **Overlays never forget.** A file copied by hand into `/etc` is copied up into the persistent store and masks the image's version of it for ever.

## How the work was organised

Discipline mattered more than speed, because every mistake on a unit in the field costs a site visit.

- **Two different bench units.** One had a dead PMIC bus and ran at 720 MHz, the other was a healthy older board revision at 1 GHz. Each change ran on both, identified by MAC address before any write. Serial consoles stayed attached, and a small on-unit checker ran a soak test every six hours.
- **Every build is a record.** A wrapper numbers each clean build, backs up the images with their checksums and pins, and refuses to build when the configuration or a feed pin has drifted. More than 80 builds have a release-notes entry saying what changed and what was checked on the units.
- **One flash procedure.** Checksum, settings backup, upload, verify on the unit, upgrade, then check the slot, the boot counter, the configuration store, the boot log and the cloud links. Always from the backed-up image, never from the build directory.
- **A real management server.** A test server ran TR-069, TR-369 (USP), XMPP and STUN, so remote configuration, backup and restore were tested end to end instead of assumed.
- **AI-assisted, human-decided.** I did the work with Claude Code: a lead session planned and reviewed, worker agents wrote code and ran bench checks, and a separate session that owns the shared framework reviewed every branch. Builds, flashes, bootloader writes and every product decision waited for my word.

## Where it ended

OS-DHP 2.2.6, built on 27 October, is the final release. It runs on both bench units with every check passing, and it comes with a 236-page user manual, a four-page overview and release notes.

I offered it to the company that sells the MA-2080X, and they decided not to ship it. That decision was about the product, not the port, and I understand it.

Not much of the work is lost. Most of the features above were built in the shared dhplab framework and are already used by other products: SMS commands, the serial gateway's back-pressure, signed plugins, the security fixes and the ucode newline audit. The board port, the U-Boot patches and the build records stay in the archive, ready if they are ever needed.

## What I would do the same way again

1. **Make the recovery path work before the risky write.** The SD-card boot was proven before MLO was ever touched, and that is what made the bootloader work calm.
2. **One irreversible change per step.** OpenWrt first on the old bootloader, the new bootloader later and separately, never both in one unattended boot.
3. **Treat "exit 0" as a claim, not a result.** The empty backups, the silent feed failures and the regex anchors all passed every check that only looked at return codes. Test the outcome: restore the backup, diff the configuration, send the bytes.
4. **Trust what shipped and what you measure.** The vendor board file over the schematic, a measurement over a net name.
5. **Write it down as you go.** Release notes per build, engineering notes per finding. That record is also what this post was written from.

The MA-2080X goes into the cabinet. The things it taught me do not.
