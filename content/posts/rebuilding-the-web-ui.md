---
title: "Rebuilding the Web UI on LuCI"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "luci", "web-ui", "rpcd", "vnstat", "playwright"]
summary: "We rebuilt the LTE572W's web UI on top of LuCI. The screens themselves are the short part of the story. The real content is a secret value with all three delivery paths blocked, custom UI that vanished on every save, and a counter bug that inflated data usage by exactly 2^32 bytes (4 GiB) per reboot."
---

<!-- Part 12 of the LTE572W Yocto-to-OpenWrt porting series -->

## What We Built (Briefly)

We removed the legacy vendor UI and built our own app plus our own theme on
top of LuCI. The theme is UniFi-style: an icon rail with a pinned panel, a
unified top band, an account menu, an off-canvas drawer on mobile, and a
slide-over shell. To meet WCAG contrast requirements we re-derived the color
token ramps from scratch. The pages cover uplink/failover, SIM and APN, modem
status, serial ports, routing, data usage, and so on.

That's it for the screens. The real content of this post is the traps we hit
along the way.

## Trap 1: There Is No Way to Pass a Secret

One page needed to accept a server authentication token and hand it to a
backend helper for storage. Walk through the paths from browser to backend
one by one:

- argv? No. It shows up verbatim in `ps`.
- stdin? LuCI's `fs.exec` has no stdin. And the third argument looks like it
  might be stdin, but it is actually an environment table. Pass it a string
  and `if (!L.isObject(env)) env = null` silently discards it. That one kept
  me staring at "token is empty" errors for quite a while.
- Environment variables? The backend refuses them. rpcd rejects
  caller-supplied env on session (ACL-checked) calls: when both `sid` and
  `env` are present, the call fails with `PERMISSION_DENIED`. Which is the
  right security decision, because otherwise a caller could change the
  behavior of a command the ACL had approved.

All three paths blocked. The solution was a staging file: the page writes
the value with `fs.write` into a 0700 directory (the ACL permits exactly that
path and nothing else), and the helper script moves it into a 0600 token
file. The value never appears in the process list or in a URL.

"How do I pass a secret" turned out not to be a UI question at all. It's a
privilege-boundary question, and the blocked paths were mostly blocked for
good reasons.

## Trap 2: Saving Deletes My Own UI

I had appended a token input and a payload preview into the form's DOM, and
they disappeared on every Save & Apply. The cause is simple: LuCI's
`form.Map` re-renders itself on save, and anything appended directly inside
it is gone on the next render.

The fix was structural: return the custom elements as siblings of the map
instead of appending into it. In a framework with declarative rendering,
anything you attach to the DOM imperatively lives exactly until the next
render.

## Trap 3: The Save Worked, but the Screen Says It Failed

Pressing Save & Apply produced an error, yet checking the config showed the
values had been saved correctly.

The cause: `m.save()` had already committed everything, so the delta to
apply was empty, and calling `uci.apply()` on an empty delta makes ubus
throw `NO_DATA` (code 5). A perfectly normal situation, rendered on screen as
"apply failed." In this case `NO_DATA` has to be recognized and swallowed.

As a bonus, `uci.apply()` doesn't touch LuCI's own change indicator, so the
navigation bar sat frozen at "Unsaved Changes: 1." After the commit you have
to call `ui.changes.setIndicator(0)` yourself.

Just as "no error" is not the same as "success," an error is not always a
failure.

## Bonus: Data Usage Grows by 4 GB per Reboot

Once the data usage page was in place, we noticed the statistics jumped by
about 4 GB every time the board rebooted.

The symptom: vnstat totals leapt by roughly 4 GiB on every reboot or
reconnect. The decisive clue was in the database. One five-minute row
recorded `rx ≈ 4294965968`, almost exactly 2^32, while the kernel counters
for the same window showed a few hundred KB. The root cause was vnstat's
`64bitInterfaceCounters` sitting at its default of `-2` (auto). The byte
counters on `wwan0` reset to zero on every reboot/reconnect, and auto mode
interprets that decrease as a 32-bit wraparound, adding **2^32 = 4.00 GiB**.
The fix is forcing `64bitInterfaceCounters 1`. A 64-bit counter cannot wrap,
so a decrease means a genuine reset, and nothing gets added.

And the fix had to land in two places: uci-defaults (new and factory-reset
boards) and a hotplug script (boards already deployed in the field), because
uci-defaults never runs again on a config-preserving upgrade. That is the
exact lesson from an earlier post in this series. One last thing: history
that was already polluted only goes away when the user resets the usage
statistics once. The config change only stops new inflation; the past stays
as it is.

## Small Things

The band display showed values like `enum 8`. The daemon was already parsing
`AT+QENG="servingcell"` and reporting real band numbers (3 = B3, 8 = B8),
but the LuCI-side decoder was re-interpreting those numbers through an old
Qualcomm/QMI NAS enum table. When the data source changes, the display layer
has to change with it.

We also found two backends referenced by the UI that don't exist at all:
`battery-mgrd` behind the battery widget and `xoleds` behind the LED toggle.
With no package present, they silently do nothing. A widget that fails
leaves no trace, which is exactly why it survives so long.

## What Automated Verification Taught Us

We automated the screen verification with Playwright and lost time to three
things we didn't know:

- Setting inputs via `.value` or `page.fill()` does not trigger LuCI's
  change tracking. Only real simulated keystrokes (`pressSequentially`)
  are recognized, and `<select>` elements need an explicit
  `dispatchEvent('change')`.
- A plain "Save" only accumulates session deltas. What's actually on disk
  can only be verified after Save & Apply.
- On a 400 MHz board, unless you poll until "Loading view…" disappears, the
  status pages screenshot as blank.

## Lessons

- A framework's security constraints are design inputs, not obstacles to
  route around. The blocked paths are usually blocked for a reason.
- DOM attached imperatively to a declarative UI lives for exactly one
  render.
- Counter bugs accumulate. Fixing the bug doesn't fix the past, and the
  later the discovery, the higher the price.
- The path that reaches new boards and the path that reaches fielded boards
  are different. A fix isn't done until it covers both.
