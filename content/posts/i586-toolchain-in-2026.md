---
title: "Building an i586 Toolchain in 2026"
date: 2026-08-21T08:10:06+09:00
draft: false
tags: ["openwrt", "toolchain", "gcc", "musl", "quark-x1000", "i586"]
summary: "The Quark X1000 is an i586 with no MMX, no SSE, and an x87 FPU. The first OpenWrt toolchain build for it broke not in the compiler but in libc's fabs(). The problem wasn't a missing instruction; it was a wrong default. Verification meant scanning 285 build artifacts."
---

<!-- Part 2 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Target

The core inside the Quark X1000 is Lakemont: i586, no MMX, no SSE, no SMP,
and it does have an x87 FPU. Those four facts dictate every toolchain
decision in this post.

I created a new `x86/quark` subtarget in OpenWrt with the CPU type set to
`lakemont`. The flags started out like this:

```
-march=lakemont -mtune=lakemont -Wa,-momit-lock-prefix=yes
```

`-momit-lock-prefix` is there because on a uniprocessor core the `lock`
prefix does nothing useful. A reasonable-looking starting point.

## The First Build Failure

The toolchain build stopped while compiling musl:

```
src/math/i386/fabs.c: error: impossible constraint in 'asm'
```

`fabs()`. Absolute value, about the simplest math function libc has, and
the compiler is saying it cannot satisfy an inline-asm constraint there.
Why? A missing instruction? No.

## The Cause

GCC's `-march=lakemont` defaults to `-msoft-float`. The compiler assumes
this CPU has no FPU at all and refuses to touch the x87 registers. But
musl's i386 port implements its math functions as x87 inline asm, and it
has no soft-float i386 variant. Feed asm with x87 register constraints to a
compiler that has x87 disabled and you get `impossible constraint`.

Meanwhile, the hardware really does have an x87. The old Yocto image had
been running an i586 x87 userspace all along, and its kernel had
`MATH_EMULATION` turned off. The compiler's assumption simply disagreed with
the hardware.

When you target an old ISA, the trouble doesn't come from "the instruction
isn't there." It comes from **the toolchain's defaults assuming a different
machine than yours.**

## The Fix

Add `-mhard-float`. The final flags:

```
# include/target.mk — i386 block
CPU_CFLAGS_lakemont = -march=lakemont -mtune=lakemont -mhard-float -Wa,-momit-lock-prefix=yes
```

To make sure flipping one flag didn't drag anything else along with it, I
checked `gcc -Q --help=target`: x87 on, MMX and SSE still off. Which is
what it showed.

One more thing. Going from soft-float to hard-float is a floating-point ABI
change. Objects built under the old assumption must not mix with the new
ones, so I deleted `build_dir/{toolchain,target}-i386_lakemont*` and
staging, and rebuilt from scratch.

## Verification

The flag was fixed and the build went through. That is not the place to
stop, because "I passed the flag, so it must be fine" is not verification.

I scanned the 285 ELF binaries and kernel modules in the build output:

| Check | Result |
| --- | --- |
| MMX/SSE instructions | 0 |
| `lock` prefixes | 0 |
| x87 instructions in libc | present (hard-float confirmed) |

In a cross toolchain, any package is free to override `-march` or ignore
your CFLAGS, and the build will still succeed silently. The failure shows up
later, on the board, only when that exact code path runs. So you count the
artifacts.

## Lessons

- On an old target, the fight is against wrong defaults, not missing
  instructions. You don't find out that `-march=lakemont` implies soft-float
  until you turn it on.
- A compiler's assumption can be disproven with hardware evidence. The
  previously shipping firmware, an x87 userspace with `MATH_EMULATION=n`,
  was that evidence.
- Floating-point flags are ABI. If you change them, you rebuild everything.
- Cross-build verification is done against the artifacts, not the flags.
  The time it takes to scan 285 files is far cheaper than chasing a SIGILL
  on the board.

What this stage bought was not a boot. It was a correct floor: proof that
every binary uses only what this CPU actually has, and nothing it doesn't.
The next post in this series is about the kernel refusing to come up on top
of that floor.
