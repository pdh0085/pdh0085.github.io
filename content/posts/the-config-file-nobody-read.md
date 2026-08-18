---
title: "The Config File Nobody Read"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "frr", "uci", "ospf", "procd", "quark-x1000"]
summary: "Saving OSPF settings in the web UI succeeded without a single error — and the routing daemons never heard about any of it, because the file the pages wrote to was never read by anyone. This is the story of rebuilding that pipeline on UCI, and of a detached restart that inherited fd 1000 and froze every start/reload forever."
---

<!-- Part 11 of the LTE572W Yocto-to-OpenWrt porting series -->

## The Symptom: Saving Works, Nothing Happens

This product has routing pages — RIP, OSPF, BGP. Hit save and it saves,
no errors. But open `vtysh` and the routing daemons know nothing.

I split the check into three steps: save in the UI, then inspect the file
contents, then inspect the actual running-config. Steps one and two
succeeded. Only step three was empty. A successful save proves nothing
about arrival.

## The Cause: The Moment frr.conf Exists

The pages were writing to `/etc/frr/{ospfd,ripd,bgpd}.conf`. The problem is
that FRR's startup script, `frrcommon.sh`, switches to
**integrated-config mode the moment `/etc/frr/frr.conf` exists**:

```sh
daemon_prep(): [ -r "$C_PATH/frr.conf" ] && return 0  # per-daemon .conf never prepared or used
vtysh_b():     [ -r "$C_PATH/frr.conf" ] || return 0  # pushes frr.conf into every daemon
```

And this board had shipped with that file from the factory. Which means the
pages had spent years dutifully writing to a file that nobody read. The
reason OSPF had never once been proven end-to-end wasn't the daemons. The
writes were going into thin air.

## The New Structure: Decide Who Owns the File

The fix wasn't about patching a few fields — it was about deciding
ownership.

```
/etc/config/frr (UCI) → frr-gen → /etc/frr/frr.conf → FRR
                           │
                           └─ vtysh -C dry-run gate
```

The web pages become structured forms over UCI instead of raw file editors,
and a generator, `frr-gen`, renders `frr.conf` itself (its init script runs
at START=94, one slot ahead of FRR's S95). Five design decisions:

- **Dry-run parse with `vtysh -C` before installing.** A single syntax
  error takes down the entire routing stack on the next restart. If the
  config is rejected, the previous one keeps running and the page says so.
- **Exit codes as a contract.** `0` = new config installed (restart),
  `2` = no change (do NOT restart — a pointless restart tears down
  adjacencies), `1` = rejected. Without a distinct "no change" outcome,
  every press of the save button shakes the routing.
- **Keep an escape hatch.** Any FRR syntax the schema can't model is
  emitted verbatim from per-section `raw` lists. FRR's configuration
  surface is wider than any schema; without this, a UCI frontend is a
  feature regression.
- **Make ownership explicit.** UCI is the sole owner of `frr.conf`. A
  single setting, `frr.global.managed='0'`, hands ownership back to a
  human, and the pages then show a "manual mode" banner. Don't pretend to
  be the owner when you're not.
- **Never delete the existing file.** If a hand-written `frr.conf` already
  exists, it gets imported wholesale into `frr.global.raw`, with the
  original preserved as `frr.conf.pre-uci`.

## The Trap: flock 1000

`frr restart` takes tens of seconds on this board — far past rpcd's exec
timeout, so LuCI's Save & Apply dies with an XHR timeout. So I detached the
restart. After which every subsequent `start`/`reload` hung forever.

The cause was procd. Its `procd.sh` takes a per-service lock with
`exec 1000>...; flock 1000`, and **fd 1000 has no close-on-exec.** The
detached process inherits that fd, and then watchfrr, mgmtd, zebra, and
ospfd inherit it in turn. The routing daemons hold our service's lock for
as long as FRR runs. The only fix is closing the fd at spawn time:

```sh
( exec 1000>&- ; setsid /etc/init.d/frr restart </dev/null >/dev/null 2>&1 & ) &
```

I hit the same trap again in a Wi-Fi reapply script. On this system,
"detached execution" now implies "fd cleanup" as a rule.

## Verification: Check at the Far End

Enable OSPF in the UI and it flows through uci, the generator, and
`frr.conf`; the running-config shows `router ospf` with the router-id and
networks.

```
vtysh -c 'show running-config' | grep -A5 'router ospf'
vtysh -c 'show ip ospf interface br-lan'
```

`show ip ospf interface` shows `br-lan` in area 0 as `No Hellos (Passive
interface)`. **This was the first time on this board that routing
configured from the UI verifiably reached FRR.** One small detail: in
FRR 10, `passive-interface X` is deprecated (it isn't VRF-aware), so the
generator emits the per-interface `ip ospf passive` form instead — so that
the generated file and the running-config match character for character,
and the next person never has to wonder which one to trust.

## Lessons

- The first question when building a configuration UI isn't "which fields
  do we expose" — it's **"who owns this file."**
- A generator without validation (a dry run) and no-change detection is a
  dangerous tool.
- A schema-based UI needs an escape hatch. Without one, users route around
  the UI, and at that moment the ownership contract breaks.
- "Saved" is not "applied," and "applied" is not "working." Always verify
  at the far end.
