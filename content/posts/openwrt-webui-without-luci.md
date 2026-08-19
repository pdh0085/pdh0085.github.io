---
title: "What Dropping LuCI Bought Us"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["openwrt", "luci", "web-ui", "performance", "embedded"]
summary: "On an MT7628 router (580 MHz MIPS, 8 MB flash), the LuCI dispatcher was charging 430 ms per page navigation. We removed LuCI from the image entirely and rewrote the runtime on nothing but static files and /ubus JSON-RPC. Login went from 459 ms to 48 ms, and all 57 existing views run unmodified."
---

We build a commercial router on an MT7628 (MIPS 24Kc, 8 MB flash, 64 MB RAM)
running OpenWrt 24.10. Its web UI had lived on LuCI for a long time, and
every page navigation visibly stuttered. This time we removed LuCI from the
image entirely and replaced it with a browser runtime we wrote ourselves
(`ls-webui`). On the server side there is no CGI, no dispatcher, no
templates. uhttpd serves static files and relays `/ubus` JSON-RPC, and that
is all it does.

The short version: the login screen went from 459 ms to 48 ms, the fixed
cost of a page navigation from 368 ms to 6 ms (then 0 from cache), and **not
a single line of the 57 existing views was modified.**

This post is a record of what we measured, why we chose this structure, and
where the traps were.

## 1. Catching the Culprit by Measurement

"LuCI is slow" is a common sentiment, but until you measure you don't know
exactly what is slow. On real hardware, after subtracting the TLS handshake
(128 ms), the pure server time came out as:

| Request | Server time |
|---|---|
| Static file | ~2–10 ms |
| `/ubus` JSON-RPC | ~5 ms |
| `/cgi-bin/luci/` | **430 ms (warm) / 3,400 ms (cold)** |

uhttpd itself is innocent. Static files are fast, and uhttpd's built-in
`/ubus` relay is fast. The only slow thing is the LuCI dispatcher path. The
dispatcher forks on every page navigation, re-parses all of `menu.d/*.json`
and `acl.d/*.json`, and only then produces its first byte. On a 580 MHz
single-core MIPS, that cost repeats on every request.

So the fix was not "optimize LuCI." It was to remove the forking path
altogether.

> The measurement itself had one trap. curl's `--next` does not reuse
> connections with this combination (uhttpd + TLS); every request shows
> `num_connects=1`, so a 125 ms handshake swallows every number. We had to
> hold the connection open ourselves with Python's `http.client` before the
> server time became visible.

## 2. The Structure: Remove Code from the Server

```
Browser ──HTTPS──► uhttpd (no forks)
                     ├── /                → index.html (static shell)
                     ├── /ls-static/*     → runtime JS·CSS·menu.json·i18n (static)
                     ├── /ubus            → in-process JSON-RPC ─► rpcd ─► uci/file/session/…
                     └── /cgi-bin/cgi-*   → cgi-io only (firmware upload · backup download)
```

The key observation: LuCI's modern views are already client-side rendered.
JavaScript runs in the browser and fetches data over ubus. What server-side
LuCI actually did was assemble the menu tree, check ACLs, and generate the
initial HTML shell, all of which can be precomputed or moved to the client.

- The HTML shell becomes one static file.
- The menu tree is baked once at boot: `menu.d/*.json` is merged into
  `/www/ls-static/menu.json`, written only when the content changed to
  avoid overlay wear.
- ACLs and sessions were always rpcd's and uhttpd's job anyway. uhttpd's
  `/ubus` relay checks permissions against the session token, and that
  layer survives with LuCI gone.
- For i18n, instead of converting `.po` to `.lmo` (mmap'd by the
  dispatcher), we convert `.po` to JSON and the browser fetches one file
  per language.

The only thing left on CGI is `cgi-io`. Firmware image uploads and backup
downloads are too big to ride `/ubus` as JSON+base64, so exactly two
file-transfer CGIs remain.

## 3. A Compatibility Layer That Leaves 57 Views Untouched

This migration was realistic for one reason. When we exhaustively surveyed
which LuCI client APIs the views actually use, it came to **43 symbols**
(`form.*`, `ui.*`, `uci.*`, `rpc.*`, `fs.*`, `L.*`). Write a new runtime
that preserves the install paths and the signatures of those 43 symbols, and
the 57 existing views plus the theme run unmodified.

We didn't even need the heavyweight modules like the 125 KB `network.js`.
The `network.*` the views call turned out to be a ubus object name, not a
client module.

There was a bonus, too. We made ubus calls issued in the same synchronous
block coalesce in a microtask and go out as one HTTP request, so the status
page's 6 calls became a single round trip (280 ms individually, 72 ms
batched). That optimization was structurally impossible in the dispatcher
era.

## 4. Sessions: No Cookies, So No CSRF

Session handling actually got simpler. We use no cookies. The session id
travels inside the JSON-RPC payload and uhttpd authenticates with it. Since
no ambient credentials ride along with requests, CSRF as a problem simply
does not arise.

Instead, chasing a real-world "I keep getting logged out" symptom turned up
two causes:

1. Nobody was refreshing the session. rpcd resets the TTL every time a
   session is used, but leave a page open untouched and nothing touches the
   session, so an hour later you get kicked mid-edit. We now send a
   keepalive (`session.get`) every third of the session lifetime.
2. Mistaking `-32002` for expiry. uhttpd answers "session expired" and
   "this call is not permitted" with the same `-32002 Access denied`. Open
   a page that happens to include one call missing from the ACL, and the
   client was terminating a perfectly healthy session by itself. Now
   `-32002` only triggers a session liveness probe, and only the probe's
   result logs you out.

## 5. The Minefield: What Silently Disappears When LuCI Goes

The real cost of this migration wasn't writing code. It was discovering the
things LuCI had been providing implicitly, all of which share one property:
they vanish without any error.

- The `luci` ubus object belongs to luci-base, not `rpcd-mod-luci`. The
  name suggests the latter; it isn't. Remove luci-base and six methods
  disappear: `setPassword` (the password-change page!), `getConntrackList`,
  `getTimezones`, `getInitList`, `setInitAction`, `setLocaltime`. We
  reimplemented exactly those six as a ucode rpcd plugin.
- The switch page's backend has the same root.
  `getSwconfigFeatures`/`getSwconfigPortState` are luci-base's too, so with
  it gone there was no way to read hardware state from a swconfig switch.
  We added a new ubus `switch` object via an rpcd script.
- rpcd's `uci.commit` requires the `config` argument (missing it fails with
  status 2). The LuCI runtime had been supplying it silently, so we didn't
  know, and for a while **Save & Apply on every page was committing
  nothing.** You must fetch the changed-config list from `uci.changes` and
  commit each one. Same for `revert`.
- `file.stat` requires the `list` permission, not `read`. luci-base's ACL
  had been quietly slipping in `"/": ["list"]`; omit it and every
  file-existence check dies.
- Environment values like `L.env.cgi_base` were injected by the dispatcher.
  They became empty strings, and firmware upload, backup download, and
  restore all returned 404 — precisely the three features we had kept
  cgi-io for.

## 6. "Renders" ≠ "Works"

The most expensive lesson. We confirmed all 57 views render and thought we
were done; exhaustive on-hardware verification then produced ten bugs in
the save/apply/upload paths. The `uci.commit` and `cgi_base` items above
are the poster children. The screens were perfect while Save & Apply was a
no-op.

A render sweep only validates the read path. Write paths
(save/apply/rollback), file transfers, and the "apply cuts my connection"
rollback scenario are all separate paths that had to be exercised on real
hardware. In particular, an apply that changes the device's address makes
rollback confirmation structurally impossible: the confirm goes to a
session on the old address, which doesn't exist at the new one. We solved
that by detecting it up front and offering the user an unprotected apply.

## 7. Results

A/B measured on two identical units (TLS handshake excluded, connections
reused):

| | LuCI | ls-webui |
|---|---|---|
| Login screen | 459 ms | **48 ms** |
| Per-navigation fixed cost | 368 ms **every time** | **6 ms**, then 0 from cache |
| Status page, 6 data calls | — | batched 72 ms (280 ms individually) |
| First-visit transfer | 313 KB | 267 KB |
| 52 concurrent requests | 3 ucode forks + queueing | **0 forks** |
| /www size | 1,229 KB | 778 KB |
| Image size | — | **−315 KB** compressed |

On 8 MB of flash, 315 KB is a number you can feel. We deliberately make no
claim about steady-state RAM. The measurement conditions differed enough
that the comparison would be murky, and the win of this architecture is "no
fork per request," not resident memory.

## 8. When This Approach Applies

It's only fair to state the preconditions that made this work:

- The views were already client-side rendered. With a pile of legacy
  server-template views, a compatibility layer wouldn't have cut it.
- Every server capability we needed already existed in rpcd/ubus. What
  didn't (switch state, etc.) we added as rpcd script plugins, which is
  work you'd have to do with LuCI present anyway.
- We control the UI. This is a product image with a closed set of views. If
  your users must be able to install arbitrary luci-app-* packages, this is
  a different conversation.

Conversely, if you're building an OpenWrt product with its own UI on a
low-end SoC, the dispatcher's fork cost was the single largest performance
budget we could reclaim without touching the hardware. And most of that
work wasn't writing code. It was making a list of everything LuCI had
quietly been doing for us.
