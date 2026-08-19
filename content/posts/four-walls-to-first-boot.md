---
title: "Four Walls on the Way to First Boot"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "quark-x1000", "kernel-config", "acpi", "serial-console", "debugging"]
summary: "The kernel came up fine. There was just no console, no disk, and no ethernet. Four walls stood between our Quark X1000 board and a working OpenWrt first boot, and three of the symptoms turned out to converge on a single disabled kernel option: CONFIG_ACPI."
---

<!-- Part 3 of the LTE572W Yocto-to-OpenWrt porting series -->

## We Made It to Userspace

In an earlier part of this series we built the `x86/quark` subtarget and its
toolchain. Time to hook the kernel into the GRUB menu and boot. The short
version: the kernel came up. It reached
`kmodloader: done loading kernel modules`, so userspace was alive.

But the early boot output was garbage characters, the console went completely
silent partway through, eMMC and USB failed to probe, and the ethernet
interfaces didn't exist at all, so the PC on the LAN never got an IP. Being
x86 does not mean a board just boots, and none of this is unusual for
embedded x86. It's the normal shape of it. There were four walls to climb,
and in hindsight they all had the same shape: the kernel wasn't receiving
something it needed from the firmware.

## Wall 1: Garbled earlycon Output

```
# wrong earlycon (garbled output)
earlycon=uart8250,mmio32,$ADDR,115200n8
# correct form — omit the baud
earlycon=uart8250,mmio32,$ADDR
```

If you specify a baud rate, the kernel reprograms the divisor assuming the
standard 1.8432 MHz UART clock. The Quark's UART clock is much higher than
that, so every character comes out mangled. Omit the baud and the kernel
inherits the setup GRUB already programmed. One tip: add `keep_bootcon` and
earlycon keeps printing the whole way through, so you can watch everything
while you fix the real console.

## Wall 2: There Is No Real Console

After `legacy console [ttyS1] enabled`, silence. The answer was already
sitting in the log:

```
serial 0000:00:14.1: ignoring port, enable SERIAL_8250_LPSS to handle
```

The Quark's UARTs are PCI devices (8086:0936), and nothing claims them
unless `CONFIG_SERIAL_8250_LPSS=y`. With it enabled, 00:14.1 became ttyS0
and 00:14.5 became ttyS1, reporting base_baud = 2764800, which is
44.2368 MHz / 16. That's the numeric confirmation of why specifying a baud
in wall 1 was wrong.

## Wall 3: eMMC, USB, and Ethernet All Dead

The symptoms ran in three directions: sdhci failed to probe with `-16`
(`genirq: Flags mismatch irq 0 ... (mmc0) vs. ... (timer)`), USB threw IRQ
warnings, and the ethernet driver never probed at all. I chased them
separately for a while, until one log line pulled everything together:

```
APIC: ACPI MADT or MP tables are not detected
```

There was one cause behind all three: **`CONFIG_ACPI` was off.** OpenWrt's
shared x86 kernel config disables ACPI and every x86 subtarget re-enables it
individually, a detail I missed when creating the new subtarget. The causal
chain:

```
CONFIG_ACPI=n → no MADT/_PRT → IOAPIC never configured
             → PCI IRQ routing wiped out → sdhci (eMMC), USB, ethernet all fail
```

When several symptoms appear at once, it pays to suspect the shared
foundation underneath them, something as low-level as IRQ routing, before
chasing each symptom on its own.

## Wall 3.5: Enabling ACPI Isn't Enough

With ACPI on, the next message was
`ACPI BIOS Error (bug): A valid RSDP was not found`. The Quark's UEFI hands
the RSDP over via the EFI path, but the custom handoff in this board's GRUB
fork doesn't pass it through to a mainline kernel. So I forced the address
with a kernel parameter: `acpi_rsdp=0x0f00e014`.

Where did that address come from? From `dmesg | grep RSDP` on the
known-good Yocto boot. This is the first appearance of a technique that
recurs throughout the series: when a working system is sitting next to you,
it is a more accurate reference implementation than any document. One
caveat: that address is specific to this firmware build. If the firmware
changes, it changes, so read it again. And for the record, `pci=biosirq`
does not work here; this is a UEFI-only board with no legacy PCI BIOS.

## Wall 4: Still No Ethernet

IRQs were alive, and eth0 still didn't exist. The Quark GbE is PCI
8086:0937, and `stmmac_pci.c` (the driver you'd expect) only carries the
generic Synopsys IDs and doesn't know this device. The driver that actually
claims it is `dwmac-intel.c`, i.e. `CONFIG_DWMAC_INTEL`. With it enabled,
`intel-eth-pci` bound 00:14.6 as eth0 and 00:14.7 as eth1, and link came up.

The final kernel command line, in summary:

```
console=ttyS1,115200n8 earlycon=uart8250,mmio32,$ADDR acpi_rsdp=0x... reboot=p net.ifnames=0 biosdevname=0
```

## Bonus: The Board Cannot Reboot

The `reboot=p` in that command line is the fifth story. Running `reboot`
hung right after `reboot: machine restart`. This kernel is built with
`CONFIG_EFI=n`, so `reboot=efi` is an empty stub, and control falls through
to `machine_real_restart(MRR_BIOS)` — a far jump to the legacy reset vector
at 0xFFFF0, where there is no BIOS. Worse, that function is `__noreturn`, so
the CF9 and triple-fault fallbacks that come after it in the code are never
reached at all. The fix is `reboot=p`: a CF9 hard reset.

There's an epilogue. Because of this bug, the `reboot -f` at the end of
`sysupgrade` took the same path. The image had been written correctly, but
the board hung, so it looked exactly like a failed update. Boards already in
the field can be rescued with `echo pci > /sys/kernel/reboot/type` before
rebooting.

## Lessons

- A kernel config trimmed by a vendor (or a distro) reflects what they
  needed, not what this hardware needs, so every disabled option in an
  inherited config is a suspect.
- When several seemingly unrelated symptoms show up together, check the
  shared foundation first. Here it was IRQ routing.
- A working firmware next to you beats any document as a reference
  implementation. If a system can be asked for the right answer, ask it.
