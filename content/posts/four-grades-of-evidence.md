---
title: "Four Grades of 'It Works': measured, armed, inference, gap"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["reliability", "testing", "openwrt", "fwa", "field-operations"]
summary: "The real value of an FWA device isn't its spec sheet; it's the fraction of incidents that end without a truck roll. So we built a table of truck-roll causes and their defenses, and graded every row measured, armed, inference, or gap. A zombie modem recovered unattended in 104 seconds. The table still has gap rows, and we wrote them down instead of erasing them."
---

## Why Truck Rolls

For an FWA or broadband operator, the thing scarier than the cost of the
device is the truck roll. One technician visit can burn the price of several
units. So the real value of a device like our LS500X/NM500G (Qualcomm
IPQ8072, based on OpenWrt 25.12) isn't the feature list. It's the fraction
of incidents that end without anyone driving anywhere.

We turned that into a table. Left column: the reason a truck rolls. Middle:
our product's defense against that cause. Right column: how we know the
defense works.

The third column is what this post is about.

## The Four Grades of Evidence

"How we know" is never written as yes/no. It's written as one of four words.

| Grade | Meaning |
|---|---|
| **measured** | We actually induced that failure, and the numbers are on record |
| **armed** | We verified the code is shipped and running, but we have never induced the failure it's supposed to catch |
| **inference** | It should work by design. Never demonstrated |
| **gap** | There is nothing |

The point of this grading is not bragging, it's self-regulation. The first
line of the document reads: "Quote measured rows freely; never promote the
other rows to measured in prose." RFP answers are given not as "yes" but
as one of these four words.

And I should say it up front: the table still has gap rows. We wrote them
down instead of erasing them, and the last section of this post is exactly
those rows.

A low-graded row wants to be upgraded, and there is only one way to do that:
actually break the thing. Here are four cases where we did.

## Case 1: The Zombie Modem

A 5G module doesn't die in just one way. Two ways we induced:

Registered but passing nothing. The link looks fine, so naive monitoring
is fooled. The defense is a keepalive aimed at a real upstream destination.
The way we induced it was fun: point the keepalive at a documentation-only
reserved address, and you get precisely that state, where the line is up
and nothing arrives. Result: three failures at 45-second intervals, then a
hardware reset, **93 seconds** from the first failure.

The zombie. The interface exists but nobody answers. We created a logical
USB disconnect with every power rail still healthy. Detection came at 41
seconds, the reset followed, the module re-enumerated 32 seconds later, and
**104 seconds** after the disconnect everything was back to normal. No human
involved.

That 104 seconds is the number for this specific failure shape. It is not
"all modem failures recover in 104 seconds"; writing that would break the
very rule this table exists to enforce.

But the real ending of this case isn't the number. The drill caught a
defect. The reset procedure started checking for USB presence immediately
after the reset, at which point the pre-reset enumeration was still
lingering, so it read "ready in 0 seconds." The actual module only dropped
off the bus 31 seconds later, and the watchdog logic misdiagnosed that as a
zombie. So every hardware reset was silently escalating straight to a power
cycle: a two-rung recovery ladder collapsed into one.

The fix was ordering. First wait for the module to disappear from the bus,
then confirm its return (measured: gone at 1 second after reset, back at
30). In the re-drill the ladder stopped at the first rung, as designed.

If we had never induced the failure, this defect would have stayed invisible
forever, because recovery "was working." It was just recovering harder than
necessary, every single time.

## Case 2: When One Bad Config Kills the Line

Change a setting remotely, break the WAN, and you've also just destroyed
your remote path to fix it. A classic truck-roll generator.

The defense: keep a last-known-good snapshot of the network, firewall, DHCP,
and failover configuration, and if the line isn't alive after a change,
restore the snapshot and re-apply the affected services.

We drilled it with the grace period shortened. We applied a bad WAN config,
the snapshot was restored at 124 seconds, and the line was back to normal at
about 130 seconds with no human intervention. That 130 seconds is a drill
value with the grace shortened; the shipping default grace is **10 minutes**.

We drilled the false-positive defenses too. A change made while the line is
already dead is accepted, not reverted, since otherwise the mechanism would
fight the very repair work being attempted. And a revert never happens
twice.

This row also carries a gap row right next to it. The device's only health
signal is the WAN, so a firewall mistake that leaves the line alive but
blocks only the path to the management server is not caught. Changes
outside the watched files aren't caught either, and that part is
deliberate: this mechanism must never revert a password or an SSID the
customer chose on purpose.

We write down what our product cannot do before anyone else does. That is
the whole reason the table exists.

## Case 3: The Last-Resort Reboot, and Why Only Three

If no upstream line comes back for 30 minutes, the device reboots itself
once. Attempts are spaced at least 4 hours apart, and after 3 attempts it
gives up.

The reason it gives up is the best part of this case: a reboot erases the
logs the device itself left behind. The log buffer is finite, and it's the
only evidence anyone will have for a post-mortem. An infinite reboot loop
destroys its own evidence. So we chose to automate *less*. Deciding the
ceiling of your automation is design work too.

## Case 4: The Row Where Our Document Was Wrong

The table had a row that read "no physical factory-reset path (gap)."
Serious, because it means phone support has no last resort.

It was wrong. The reset path had existed for a very long time: holding the
WPS button for more than 3 seconds wiped the configuration. It just wasn't
documented anywhere.

And acting on that wrong row ("there's none, so let's build one") produced
two reset paths, both firing on a single button press, because the system
runs both kinds of button handler.

Cleaning it up, we fixed three things at once:

- Raised the threshold to 10 seconds. Three seconds is far too easy to
  trip by accident.
- Added a model check. The file defining this behavior lives in a
  location shared across products built on the same chipset family, and
  because of that file structure, other companies' devices on the same
  chipset had been shipping the same "hold WPS 3 seconds and your config is
  gone" behavior. The model condition closes that path.
- Made the reset leave a kernel log entry.

We drilled it with three real presses. One second gets you WPS pairing. Six
seconds gets you nothing at all, though that press would have wiped the box
under the old threshold. Eleven seconds gets you a factory reset, with the
device back up in about 30 seconds. And we read the reset log back *after*
the reboot, so this reset is auditable after the fact.

A feature the design document said didn't exist actually did, and acting on
the false belief created a real defect. There is no better argument for why
the table has to be filled in.

## Finding the Cause Without Rolling a Truck

Beyond the drills, briefly, the other rows.

- Logs that survive a reboot. Kernel messages from just before a restart
  are kept in a DRAM region and read back on the next boot. Measured twice;
  a marker planted before the restart was read back byte-for-byte afterwards
  (63 KB). One thing we only learned by measuring: the ordinary console
  output path never reaches this store. There is a path that looks plausible
  and leaves nothing.
- Remote log collection. The management server pushes four parameters
  and the device uploads a diagnostic bundle. Isolation was proven by
  refusal: aiming at another device's area gets an authentication denial.
  The bundle also deliberately excludes one specific log file, because at
  debug level that file contains management credentials in plaintext.
- Delivering improved defaults to fielded units. How do you get a better
  default into devices already sold? We rejected "just push it from the
  management server," because this product ships with no management server
  address configured, so that answer would freeze every retail unit at
  factory defaults forever. Instead we built a numbered migration scheme and
  induced-tested that the failover-timer improvement from an earlier post in
  this series (51.30 s down to 9.58 s) actually reaches a unit that never
  had it. After a settings-preserving upgrade the value converged, and on
  the second reboot nothing happened at all. It's idempotent.

The most important thing about migrations is not the mechanism but the
boundary: a migration fixes only what is measurably wrong and never touches
a value the customer chose. There is no mechanism enforcing that boundary —
it is held by discipline. The document says exactly that, too.

## The Numbers

Measured rows only.

| Truck-roll cause | Defense | Measured |
|---|---|---|
| Bad firmware | A/B rollback | 26.5 s / 43.5 s; ~75 s on kernel panic |
| Wired line failure | 5G failover | 9.58 s |
| Modem zombie | Staged recovery ladder | 104 s unattended |
| Modem unresponsive (registered) | keepalive → hardware reset | 93 s from first failure |
| Bad configuration | Last-known-good restore | 130 s drilled (grace shortened); shipping default 10 min |
| Install-day misprovisioning | Zero-touch | 101.5 s |
| Phone-support last resort | 10-second button → factory reset | back up in ~30 s, auditable |
| Post-mortem evidence | Reboot-surviving logs | 63 KB recovered, verified twice |

## What Isn't Measured Yet

Honesty is the product here, so the closing section is the low-graded rows.

- The hardware watchdog. We verified it is shipped and running (30-second
  period), but we have never induced the hang it is supposed to catch.
  Grade: **armed**. Which is why nowhere in this post is the watchdog called
  a safety net.
- Remote management over the 5G path. Every management-session measurement
  was taken over Ethernet. It should work over 5G by design; it has never
  been demonstrated. Grade: **inference**.
- A bad config that blocks only the management server while the line stays
  up. That's the gap row from Case 2. Grade: gap.

We are spending next quarter erasing these three lines.

## What We Keep

"It works" is not an answer. The grade of evidence behind the claim is the
answer, and four grades have been enough for us. The only way to upgrade a
row is to actually break the thing, and a drill's real payoff is rarely the
number — it's the defect hiding inside something that everyone thought was
working. The rows I value most are the gap rows; writing down what you
can't do first is what makes the rest of your numbers believable.

Two smaller things worth keeping as well. Put a ceiling on your automation:
a reboot erases its own logs, so after three attempts we stop and preserve
the evidence instead. And the boundary of automatic fixes is discipline,
not technology. Fix only what is measurably wrong, never touch what the
customer chose, and write that discipline into the document.
