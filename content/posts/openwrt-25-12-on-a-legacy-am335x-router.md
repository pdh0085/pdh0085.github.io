---
title: "A Month Putting OpenWrt 25.12 on a 2010s Industrial Cellular Router"
date: 2026-10-13T10:29:20+09:00
draft: false
tags: [openwrt, embedded, am335x, u-boot, firmware-update, reliability, debugging, ipsec]
summary: "A record of moving aging AM335x industrial routers, still on vendor Linux 3.2 and a 2011 U-Boot, to OpenWrt 25.12 without pulling units from the field. It is written around what turned out to be wrong rather than what went well."
---

This is a record of moving aging AM335x devices, running vendor Linux 3.2 and a 2011-vintage U-Boot, to current OpenWrt without pulling units from the field. I've written it around what turned out to be wrong rather than what went well.

| Item | Details |
| --- | --- |
| SoC | TI AM3352, 512 MB DDR3 |
| Storage | eMMC 1.78 GiB (no NAND) |
| Existing software | Linux 3.2, U-Boot 2011.09 |
| Target | OpenWrt 25.12.5, kernel 6.12 |
| Peripherals | LTE module, two gigabit ports, isolated I/O, RS-232/485 |

## Why move it, and what stood in the way

This is an industrial cellular router that has been running in the field for close to ten years. The last vendor firmware build dates from around 2019, and the kernel is stuck at 3.2. Adding features was out of the question; even security updates were hard.

There were three requirements.

- Units in the field have to be migrated remotely, without anyone going on site.
- After the move, the existing configuration (APN, LAN address, firewall rules and so on) has to survive intact.
- If the new firmware fails to boot, the unit has to recover on its own.

## The kernel came up, but the ramdisk was gone

The vendor bootloader reads the Android boot image format (header + kernel + ramdisk). It is a 2011 build that knows nothing about device trees, so I passed the DTB appended to the zImage.

The first snag was the ramdisk address. The vendor placed the ramdisk at `0x81000000`, but the 6.12 kernel, once decompressed, reaches all the way to `0x81105ad0` including its bss. The kernel was overwriting its own ramdisk the moment it decompressed. Moving the ramdisk to `0x84000000` fixed it.

The second was the console. With `earlycon` enabled, boot hung. This was because the omap-serial driver reinitialized the UART while the early console was still polling its status register. For this kernel combination, I settled on not using earlycon.

## Borrowing the vendor's own update path

The units in the field only have the vendor firmware's update function. So I built a tool that reproduces, byte for byte, the container format the vendor firmware accepts (header, length, checksum). It includes a self-test that feeds in an image the vendor built and checks that the result matches. The vendor unit takes an OpenWrt image sent this way for one of its own updates and writes it to both slots as is.

The migration was split into two stages.

1. Leave the vendor bootloader in place and install only OpenWrt. It is already a bootable format, so the risk is low.
2. Then, with the administrator's consent in the OpenWrt web UI, replace it with the new U-Boot 2025.10. The sequence is write, read back and verify, and restore the previous bootloader on failure.

**Replacing the bootloader is the only step in this project that could require a site visit.** There is only one copy of the MLO (the first-stage loader) on the eMMC, so if power drops mid-write, the unit won't wake up. That is why, before the first attempt on a real unit, I proved the SD card recovery path on the bench.

## No partition to keep the configuration in

The root filesystem is a ramdisk, so if nothing is done, the configuration disappears on every reboot. The vendor kept its settings as key=value pairs in a separate nvram partition. But there was no room for OpenWrt's `/etc`.

I shrank the media partition and created a new 256 MB ext4 partition after it. `/etc` is an overlay on top of it. The interesting part is that the vendor kernel can't see this partition at all. The vendor kernel only creates 8 device numbers per eMMC (the disk plus 7 partitions), so an 8th partition is treated as if it doesn't exist. That means going back to the vendor firmware causes no trouble, so it ended up acting as a safety net.

### Partitioning itself on first boot

Nobody can partition a unit in the field by hand. So on first boot, the unit checks whether the GPT is the vendor original. If it is, it shrinks the filesystem, writes the new GPT, backup table first and primary last, reads it back to confirm, and reboots. The new partition is formatted on the next boot.

A bug turned up here. The check for "has the new GPT already been written?" compares against an md5 constant, and that constant was wrong. Once the table was judged to be neither the original nor the new GPT, the code did nothing and didn't even log. I only found the cause after measuring the md5 again on a real eMMC. One more thing: after writing the new GPT, telling the kernel with `partx -u` does not bring the partition names back. A reboot was required.

## An import that mistook "still at defaults" for configuration

On first boot, the unit reads the vendor nvram and carries the APN, LAN address, DHCP range, time zone, VPN and firewall rules over into OpenWrt configuration, exactly once. It leaves a marker so this doesn't run again.

The bug showed up the first time I ran it against real nvram from the field. In the vendor UI, "switch the port to WAN mode" and "enter a WAN address" are separate actions. On a unit where only the first had been done, the WAN address field was still at its factory default, and the import carried that over as a static WAN with no gateway. As it happened, that value was the same as the new firmware's default LAN address. Now, if all three values are factory defaults, the import leaves WAN on DHCP and logs why. The rule came from comparing the vendor's defaults file against real nvram.

## A/B rollback was only guarding the first boot

The new U-Boot has A/B slots and a boot counter. Each boot attempt increments the counter, and past the limit it switches to the other slot. When a boot completes successfully, a script on the OpenWrt side clears the counter.

Then one day, an image that had already booted fine on the same unit hung and restarted 77 times in a row. Not once did it roll back.

The cause was in U-Boot's environment-based counter. That mechanism only stores and reads the counter while `upgrade_available` is 1. And our confirmation script was setting that value to 0 as its way of saying "boot succeeded". In other words, rollback was armed only for the first boot after an upgrade and stayed off forever after.

```
# Before: success disarms monitoring
fw_setenv bootcount 0
fw_setenv upgrade_available 0

# Now: clear only the counter, keep monitoring armed
fw_setenv -s - <<EOF
bootcount 0
upgrade_available 1
EOF
```

The assumption that "an image that booted once keeps booting" was wrong. Even the same image could hang partway through boot depending on conditions at power-on, and rollback existed for exactly that case. After the fix, I confirmed on a real unit that the counter climbs 1, 2, 3, 4 and the fourth attempt switches to the other slot. Combined with the 180-second hardware watchdog, automatic recovery takes 12 to 13 minutes.

### The patch hiding next to it

While hand-editing the U-Boot patch that generates the A/B environment file, I once extended the hunk body to 94 lines but left the header's line count at 85. `patch` applied only the first 85 lines without an error, and for several builds the result was a bootloader missing two boot commands entirely. I only found out when the first real install repeated "`ab_bootcmd` not defined" endlessly.

## Four GPIOs that behaved opposite to their names

This board turned up four lines whose actual polarity was the opposite of their signal name or existing comment. Each time, the deciding evidence was the schematic, the value the vendor code actually uses, and measurement.

| Line | What we believed | Reality | Symptom |
| --- | --- | --- | --- |
| SIM select mux | high = SIM1 | low = SIM1 | "No SIM" with a good SIM inserted |
| Modem power | active high | powered when low | Modem kept the "no SIM" verdict from the previous boot |
| RF disable (`W_DISABLE_N`) | active low, as named | raw high = RF off | Radio had been off since the factory; the modem only reported "Invalid transition" |
| Reset button | external pull-down, active high | external 10 kΩ pull-up, pressed = GND | The button event module was missing, so nobody noticed |

The third line stayed hidden for quite a while. The `_N` in the signal name reads as "active when low", but there was an inversion between the pad and the module pin. Nowhere in the modem's error messages was there any mention of a GPIO.

**The rule I settled on:** when handling a new GPIO on this board, trust neither the signal name nor our own comments. Go by the value the vendor firmware actually uses and by measurement.

## SMTP was in .config but not in the binary

The day I first tested the alarm email feature on a real unit, curl answered "Protocol smtp is disabled". `.config` clearly had `CONFIG_LIBCURL_SMTP=y`.

OpenWrt builds the curl package with `--disable-dependency-tracking`. After the option was turned on, configure ran again, but object files that had already been built were not recompiled. As a result, for a month every image shipped with a libcurl that had no SMTP. There was an earlier record of checking with `strings | grep smtp` and concluding "it's in there", and that was wrong too. That was because recent curl keeps the names of disabled protocols around.

After changing a package option, I now `clean` that package, and I verify with `curl -V` on the device instead of a string search.

## L2TP that went out in plaintext when the IPsec child SA closed

On the L2TP/IPsec client, when the IPsec child SA closed, xl2tpd's keepalives were going out in plaintext on UDP port 1701. Even with no user traffic, an idle tunnel was leaking, and once it went on for 4 minutes 30 seconds. The receive side was already strict, accepting only L2TP that arrived over IPsec; it was the send side that was loose.

The fix was setting `close_action = trap` on the child SA. Verification had its own pitfall. Tearing the SA down on the unit itself with `swanctl --terminate` is an administrative action and does not fire `close_action`. So the first two tests tested nothing. Only when I had the server on the other end close the child SA did the trap policy engage, and I saw the plaintext stop.

## Holding out without a site visit

Sending a person to one field unit costs more than the unit itself. So the goal was that faults the unit can notice on its own should not turn into a site visit.

- **Dead-man reboot:** if every uplink is unresponsive for 30 minutes, reboot. On consecutive reboots the wait doubles, capped at one reboot per day. The existing package default, "every 6 hours, forever", kept rebooting units at private-network sites, accomplishing nothing but wiping the evidence.
- **Boot record:** keeps how each of the last 16 boots ended (web reboot, upgrade, dead-man, power loss and so on) and the tail of the log from the last clean shutdown. It survives a factory reset.
- **Firmware image reporting:** exposes the A/B slots separately to TR-069/USP. When an update fails and the unit falls back to the previous slot, the remote management server sees "reverted to a different image", not "success".
- **Storage monitoring:** watches usage of the configuration partition and any switch to read-only.

While filling in this list, I found that the reset button did nothing at all. That is the fourth row of the GPIO table above.

## Lessons, looking back

1. **A surprising measurement is a reason to suspect the measuring tool before it is a discovery.** The SMTP string search, a truncated list output and a wrong md5 constant all produced "plausible results".
2. **"Found no counterexample" is not "there is no counterexample".** The tool may have been unable to see that kind of case. The cleaner the result, the sooner I check whether the tool was blind.
3. **An explanation that fits the measurement perfectly is not evidence that the measurement is right.** Attach a plausible reason to a wrong number, and the reason makes the number look more trustworthy.
4. **A log line becomes a cause only when read in sequence with what came before and after it.** Read without context, a single log line easily looks like the cause.
5. **"The build passed" and "it runs on the device" are different sentences.** Right to the end, most defects were the kind that only surface in real use.

Based on OpenWrt 25.12.5, Linux 6.12 and U-Boot 2025.10. The board-specific patches and DTS can be released under the GPL; if there's interest, I'll tidy them up and post them.
