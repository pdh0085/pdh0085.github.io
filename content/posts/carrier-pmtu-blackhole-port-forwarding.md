---
title: "Debugging a Port Forwarding Failure That Turned Out to Be a Carrier PMTU Blackhole"
date: 2026-08-16T12:00:00+09:00
draft: true
tags: ["openwrt", "lte", "networking", "mtu", "debugging"]
summary: "Port forwarding on an LTE router worked for small packets but silently died for large ones. The culprit: the carrier network dropping packets over ~1400 bytes without sending ICMP Fragmentation Needed."
---

<!-- SKELETON - we will fill this in together.
     Structure: Symptom -> Diagnosis -> Root cause -> Fix -> Lessons -->

## The Symptom

<!-- What the customer/you observed. Small requests OK, large transfers hang. -->

## First Suspects (All Wrong)

<!-- Firewall rules? NAT? The usual checklist that turned up nothing. -->

## Narrowing It Down

<!-- tcpdump on WAN, ping with DF bit and varying sizes, the moment it clicked. -->

```bash
# Example: probing the actual path MTU with DF set
ping -M do -s 1372 <target>   # 1372 + 28 = 1400 bytes: passes
ping -M do -s 1400 <target>   # 1428 bytes: silently dropped, no ICMP back
```

## Root Cause: A PMTU Blackhole in the Carrier Network

<!-- Carrier drops >~1400B without ICMP Frag Needed -> PMTUD breaks. -->

## The Fix

<!-- MTU 1400 on the WAN interface + why MSS clamping alone was not enough. -->

## Takeaways

<!-- 1. PMTUD assumes ICMP works. On cellular, it often does not.
     2. "Works for small packets" is the classic MTU smell.
     3. Bake the fix into provisioning, not a one-off hotfix. -->
