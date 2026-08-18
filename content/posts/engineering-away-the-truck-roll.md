---
title: "Engineering Away the Truck Roll"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: [5g, cpe, reliability, self-healing, tr-069, fleet-operations]
summary: "We worked backwards from every situation that forces a field visit and built five guards into our 5G CPE. The hard part was never the fixing logic — it was the stopping conditions. A recovery feature that doesn't know when to stop isn't recovery; it's a new outage."
---

## What One Visit Costs

The most expensive event in the CPE business is not a technical failure. It
is **a human being getting into a truck.**

One truck roll, counting only labor and vehicle costs, is worth several
units of hardware. Add the hours the customer spent waiting and the call
center time on top. And most of those visits are made to do something a
remote hand could have done: **reboot the box, or roll back a setting.**

So on our 5G CPE we approached this not as feature development but as a
**problem to work backwards from.**

> List every situation that forces a person to physically go to the site,
> and for each path ask: "can the box do this by itself?"

Five paths came out of that exercise. And the thing we learned along the
way is that **the hard part was never the fixing logic.**

## The Five Paths

### 1. Nobody knows why it rebooted

The most common call-center conversation goes like this: "the internet
suddenly dropped yesterday and then came back."

At that point we know nothing. Did the unit reboot? If so, was it a power
outage, a kernel panic, the watchdog, or an operator pressing a button? The
logs vanished with the reboot.

So the first guard is not recovery — it is **record-keeping.** On every
boot, the unit writes one line saying **why the previous shutdown
happened.** Intentional reboots carry a tag: operator command, firmware
upgrade, or one of the other guards. A death with no tag means a power loss
or a crash.

It is a boring feature, but **the four guards that follow all stand on top
of this record.** You cannot count "failed boots" until you can tell them
apart from "intentional shutdowns."

### 2. One config line severs the management path

It is easy, when changing configuration remotely, to close the door you
came in through. Firewall zones, routing metrics, the WAN interface — one
wrong change and the device is unreachable from that moment on. And the
only way to undo it is also remote.

The second guard **starts a clock whenever a watched setting changes.**
Within a fixed window, the unit must actually confirm "is this device still
reachable?" If confirmation never arrives, it **reverts to the
configuration as it was just before the change.**

The judgment criterion is the whole point. The question is not "is the
configuration valid?" but **"can we actually reach the box?"** A
syntactically perfect configuration can isolate a device just as
thoroughly as a broken one.

### 3. Cellular is attached but no data flows

There is a state where the modem reports "registered," the interface is
up, and no packets move. To the user this is simply "the internet is
down."

Our original watchdog had exactly **one move** here: bounce the interface.
If that didn't work, it had nothing left.

The third guard turned that one move into a **ladder.**

| Level | Action | Limit |
|---|---|---|
| L1 | Restart the interface | up to 3 times |
| L2 | Network reselection (`AT+COPS` re-registration) | 2 per 6 hours |
| L3 | Reboot | **once per day**, at least 30 minutes after boot, only when the clock is trustworthy |

**Designing L2, the hardware taught us something.** The original plan was
an RF cycle — turn the radio off and back on, the textbook move. But on
this modem, sending that command was **irreversible.** Once off, every
attempt to bring the radio back failed, and a reboot was the only way to
recover. A measure meant to fix "no data" would have put the entire box to
sleep instead.

So L2 became **network reselection**, not an RF cycle. **We were lucky not
to have assumed that the action written in the plan would work as-is on
this hardware.**

Look at L3's restrictions and the character of this ladder becomes clear.
Once per day, 30 minutes after boot, trusted clock — every one of them is
**not a condition for rebooting, but a condition for not becoming a reboot
loop.**

### 4. It boots, but it can't be managed

There is a state where the unit boots but the management surfaces — the
web UI, the remote management daemon — never come up. A box that is alive
but untouchable.

The fourth guard: if a boot fails to reach a "usable state" **three times
in a row**, revert to the last known-good configuration and reboot.

Two places where the design deliberately deviates from the plan:

**We did not use "previous boot lasted under 60 seconds counts as a
failure."** We use a **health marker** instead. When a boot first reaches
a manageable state it writes the marker; the next boot judges the previous
one by that marker's absence. It is simply the better question — **a unit
that stayed up for 10 minutes without its management daemon is exactly as
unreachable as a unit that panicked after 4 seconds.** Both need the same
prescription.

**If the configuration already matches the baseline, do nothing.** This
safeguard was not in the plan. In that case the configuration is not what
broke the boot, so reverting changes nothing — and rebooting anyway would
be **precisely the loop we ourselves built.**

### 5. The data path is dead entirely

Every guard above matters only **while the box can still be reached in
some form.** When cellular fails completely, remote management dies with
it.

The fifth guard is a **text message.** Even when cellular data is dead,
SMS often survives on its separate path. A single signed text can trigger
a small set of remote actions.

The security design is most of this feature.

- **The sender's number is not authentication.** It can be spoofed.
  Authentication is an **HMAC signature** made with a per-unit secret key.
- **Replay attacks are blocked.** Every command carries a counter, and
  numbers already used are rejected.
- **There is an hourly processing limit** — and **rejected commands count
  against it too.**
- **There is no shell-command capability.** Commands come from a fixed
  list only: reboot, APN reset, temporary SSH opening (auto-closed after
  24 hours), management-server address restore.
- **Factory reset is excluded from the default list.** An operator has to
  enable it explicitly.

## The Actually Hard Part: Knowing When to Stop

That was the feature list. But it is not where most of the time in this
work went.

Automatic recovery features share one trap: **building the fixing logic is
easy; building the stopping conditions is hard.** And a recovery feature
that doesn't know when to stop is not a recovery feature — it is a new
outage.

Nowhere was this sharper than in the **second stage** of guard 4: "if
reverting the configuration still doesn't produce a good boot, factory
reset."

It is the logically natural next step — and we could not build it. Here is
why.

> On this platform, factory reset erases the configuration area **from
> outside Linux.** But our record saying "we already tried a reset" lives
> inside that same area. The moment the unit resets, **it forgets that it
> reset.** What cannot remember cannot stop, and **a guard that cannot
> stop is a guard that reinstalls the factory image forever.**

So before writing any code for this stage, we ran an **experiment.** We
planted markers in seven locations, ran one factory reset, and measured
what survived.

The result: the reset erased less than folklore said. It wiped the
overlay's **contents**, not the volume itself — and a separate volume
holding factory provisioning data **survived untouched.** That was where
the counter belonged.

And as a side effect, that experiment **exposed a problem we did not know
we had.**

### Self-Reset Was Invalidating the Call Center's Key

The SMS secret key from guard 5 lived in the area the reset erases. Which
means: **the moment a unit factory-resets itself, the key the call center
holds for that unit becomes invalid.**

At precisely the moment when the lifeline is the last resort. The box
believes it has "healed itself" — when in fact it has **cut its own last
remaining remote path.**

We moved the key to the volume the reset cannot erase. This key is not
customer data; like a MAC address, it is a **device credential** — never
something a factory reset should have been erasing in the first place.

### Two Records That Answer Two Different Questions

One more from the same thread. At first we kept a single record meaning
"a reset has been attempted." Live testing exposed the flaw.

When a reset **succeeds**, that record gets cleared (the unit booted
healthy, so returning the permission is correct). But the reset also
erases the boot log that contained the reset event. Net result: **when a
reset succeeds, every trace of it disappears.**

That is the state in which you tell a customer whose configuration
vanished overnight: "nothing happened."

So we split the record in two.

| Record | Question it answers | When it is cleared |
|---|---|---|
| A | **May we do this again?** | one healthy boot → cleared |
| B | **Has this box ever reset itself?** | **never** |

**They are different questions.** Merge them into one record and one of
the two answers is guaranteed to be wrong.

### And This Stage Ships Disabled

We built it — and set the shipping default to **OFF.**

Here is the reasoning. This action discards all customer configuration and
cannot be undone. Meanwhile its trigger — "the management daemon didn't
come up" — is **a condition that one bad release can create across the
entire fleet at the same time.**

Worst case, a single firmware we got wrong factory-resets every device in
the field simultaneously. That risk belongs to the **fleet operator**, not
to us — so the decision to turn it on must be the operator's too.

## The Guard That Watches the Guards

Finally, the most instructive incident of this whole series.

Alongside the guards above, we built a **pre-flash firmware check**: verify
the image **before** burning it, so a truncated download or a corrupted
file gets rejected before storage is wiped. It is the prevention-side
answer to the bricking problem an earlier post covered.

We took the first image carrying this check to the bench to flash it — and
**it was rejected.**

The image was fine. Byte-for-byte it had the same structure as the
previous, already-verified version. The fault was in the check itself. It
looked for a particular entry inside the archive; that entry was a
directory, so the listing showed its name with a trailing slash. The check
demanded an **exact match**, so it never matched.

In other words, this guard was **rejecting every valid image.** A feature
built to prevent bricks had nearly blocked all firmware updates instead.

More important than the embarrassing cause is **why we didn't catch it.**
We had tested this check diligently. Truncated files, missing members,
insufficient storage, low battery — it rejected every one of them,
correctly.

**We had only tested the rejection paths.** We never tested acceptance.

> **When you test a guard, checking only that it blocks what it should
> block is seeing half the picture. You must also check that it passes
> what it should pass — and feed it real production artifacts to prove it.**

Since that incident, our release procedure includes **running two real
built images through their own image's checker.**

## Lessons

- Approach truck rolls as **paths**, not features. Start not from "what
  feature should we add" but from "what situations force a person to go."
- **All automatic recovery stands on records.** If you can't tell
  intentional shutdowns from accidents, you can't count anything.
- **Judge by reachability, not by state.** Not "is the configuration
  valid" but "can we actually reach the box."
- **The hard part is not the fixing logic but the stopping conditions.**
  Recovery that doesn't know when to stop is a new outage.
- **Ship irreversible actions disabled by default.** Especially when the
  trigger can fire across the whole fleet at once.
- **Test guards in both directions.** What they block, and what they let
  through.

And all of this is visible from the remote management server. Which unit
fixed what by itself, when and why it rebooted, whether it has ever
reverted its configuration — an operator can see it **before deciding
whether to dispatch.**

**The surest way to reduce truck rolls is to give operators, remotely, the
evidence to decide whether the truck needs to go at all.**
