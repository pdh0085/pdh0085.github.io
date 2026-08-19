---
title: "It Was Never the Hardware"
date: 2026-08-17T09:51:00+09:00
draft: false
tags: ["gnss", "debugging", "5g", "firmware", "root-cause"]
summary: "A 5G CPE in the field reported zero GPS satellites. Every value we could read matched a healthy unit, so the case was closed as a hardware fault. The problem was the values we couldn't read: one AT command later, the 'dead' board was tracking 22 satellites."
---

## Zero Satellites

The report from the field was short: GPS doesn't work. Zero satellites.

The same model on our bench was fine. Same application firmware, same modem
firmware version, same antenna configuration. So we did the diligent,
textbook thing: we built a comparison table, healthy board and failed board
side by side, and went looking for the difference.

| | Healthy board | Failed board |
|---|---|---|
| Application firmware | same | **same** |
| Modem firmware version | same | **same** |
| Antenna port configuration | same | **same** |
| GNSS status files | same | **same** |
| Positioning session | normal, updating every second | **session error** |
| Result | 19 satellites | **0 satellites** |

Every software value we could read was identical, and only the outcome
differed. So the case was closed the way that evidence seemed to demand:
hardware fault. All that remained was to determine whether it was the
antenna or the module, so the plan was to swap antennas between the two
units. That's what the report said.

That conclusion was wrong.

## The Row the Table Didn't Have

A few days later we started interrogating the failed board's modem with AT
commands, one by one. Not because of some clever plan; it was closer to
"let's exhaust everything we can do before tearing the antennas off."

Then one line came back different.

```
Healthy board:  AT$GPSLOCK?  →  0
Failed board:   AT$GPSLOCK?  →  3
```

`$GPSLOCK` is the modem's GNSS lock setting, and 3 means fully locked. The
board didn't have zero satellites. It had an engine it was forbidden to
start.

```
AT$GPSLOCK=0
```

One command, and a little while later the "broken" board was tracking
**22–25 satellites** with HDOP 1.99 and horizontal accuracy ±9.9 m, the same
quality as the healthy bench unit. After a reboot it acquired a fix again
entirely on its own. The first five minutes of 4 satellites and HDOP 52 were
just a cold start.

### Why That Row Was Missing

The more interesting question is why our table never had that row in the
first place.

`$GPSLOCK` lives in the modem's internal NV memory. It does not appear in
anything we had compared: not the config files, not the antenna port
settings, not the status files, not the firmware versions. No amount of
diligent `diff`ing on the Linux side would ever have shown it.

Our comparison table could say "everything is the same" only because that
table didn't have that row. The moment you believe the table is complete,
whatever lies outside it stops existing.

### And Our Own Docs Threw Away the Clue

The painful part is that the decisive clue had been in our hands from the
beginning.

On the failed board, `AT$GPSP=1` (GNSS on) returned `ERROR`. And our
internal technical documentation said: *"this command is rejected on
healthy units too."*

So that observation was classified as normal and discarded.

When we later checked on an actual healthy bench unit, the doc was wrong. On
an unlocked board, `AT$GPSP=1` is accepted with `OK`. That `ERROR` had been
the lock symptom itself all along.

One wrong line of documentation disguised the most decisive piece of
evidence as known-normal behavior. As a result, we had judged a perfectly
good board defective and were planning to start pulling antennas.

## Fixing One Line Would Have Been Whack-a-Mole

We could have stopped there: write `AT$GPSLOCK=0` into the service
procedure, apply it whenever a report comes in, done.

The problem is who set the lock in the first place.

It wasn't our code, that much was certain. There isn't a single line in our
source tree that sets a GNSS lock. Which means the modem did it to itself,
and indeed this modem has a separate setting (`$GNSSDISACFG`) that disables
GNSS when certain host events occur. And that setting's factory default
allows the lock.

Which means the lock can come back at any time, and since it's the factory
default, every unit in the field is exposed to the same condition.

Fixing one unit and preventing recurrence are different jobs. We adopted
`$GNSSDISACFG=0` (no host event may disable GNSS) as the shipping default.
The cost is power consumption, and our positioning daemon already holds a
continuous session anyway, so most of that power saving had been forfeited
long ago.

## Then We Made It Fix Itself

Writing it into a service procedure isn't enough. A procedure runs when a
report comes in, and a report comes in only after a customer notices their
GPS is dead.

So we built a self-healing check into the firmware. It does something
simple:

- Check once at boot; after that, check periodically only while there is no
  fix.
- If the lock is set, clear it, and normalize the lock-permission setting at
  the same time.
- Record what was fixed and when, in storage that survives reflashing.

We verified it by fault injection: deliberately set `$GPSLOCK=3` again,
hands off. **The unit cleared it by itself in 23 seconds.**

### Two Traps That Only Measurement Caught

Both are about how you decide "is GNSS healthy," and neither would ever have
been caught by reasoning alone.

① The positioning daemon keeps serving the last fix forever, even after
GNSS dies. We set the lock and watched for four minutes: `fix=true`,
11 satellites, unchanged. The only thing frozen was the timestamp. So a
health check that looks at the `fix` flag will never fire precisely when
you need it. Liveness must be judged by the age of the fix.

② The fix timestamp always runs about one second ahead of the system
clock, so computing the age gives you a negative number. Our first rule was
"negative = clock broken = untrusted," and suddenly every healthy unit read
as sick. When a clock is actually wrong, it's off by years, not seconds. A
tolerance of about 60 seconds settled it.

Neither trap is visible in the code, no matter how long you stare. You see
them by sitting in front of a deliberately locked board for four minutes.

## What's Still Open

Our side of the story changed conclusions, but one question remains
unanswered: we still don't know which host event set the lock in the first
place. Today's mitigation is "clear it when it happens," not "make it never
happen." A follow-up inquiry is filed with the vendor; if an answer comes,
the polling check can become event-driven.

## What We Took Away

Mostly habits, not conclusions. "Everything is the same" is only true if
the comparison table is complete; a row the table doesn't have can't even
show up as different. Internal documentation can manufacture evidence, too.
One wrong line becomes "it's always been like that" and reclassifies the
decisive clue as normal, which is why judgment criteria written in docs
need to be re-measured against real hardware once in a while. Fixing one
unit and preventing recurrence are separate problems; once you find root
cause, the next question is always why it was set and whether it can be set
again. And a self-healing feature's health criteria have to be measured on
real hardware, because whether a value keeps updating or freezes is written
nowhere in the code.

And one last thing. Our original plan was to swap the antennas. Had we done
that first, the result would have been "still broken," so we'd have
replaced the module next. Also "still broken."

**Replacing hardware never turns a software problem into a fixed one.**
Fortunately, we typed one more command before pulling the antennas off.
