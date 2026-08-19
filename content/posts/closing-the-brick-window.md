---
title: "We Pulled the Plug Mid-Flash"
date: 2026-08-17T10:00:00+09:00
draft: true
tags: ["ota", "firmware-update", "reliability", "openwrt", "recovery", "embedded"]
summary: "If power fails mid-flash, the device is a brick. We had no room for A/B partitions: a second slot needs 226MB and the 1024MB of storage had none to spare. So we closed the window with the recovery partition that ships from the factory anyway, and then we actually pulled the plug to prove it. Back to a healthy boot in about 2 minutes 30 seconds, no human hands."
---

## A Two-Minute Window

Our 5G CPE lives in a customer's home or office, somewhere we cannot reach.

When you push a firmware update to a device like that, exactly one stretch of
the process is frightening. Download the new image, verify it, erase
storage, write it back. If power fails in the few minutes between the erase
and the end of the write, the device has no rootfs to boot. It's a brick.

And a brick requires a human to go there in person. Every other automatic
recovery feature we've built stands on one assumption: the device boots. If
it doesn't boot, all of it is void.

The textbook answer is A/B partitioning: keep two copies of the system,
write to one while you can still boot the other. If the write dies, you
simply boot the old slot, and the brick window never exists in the first
place. This blog has covered [rollback drills on a product that has A/B
slots](/posts/ab-rollback-drills/); this product couldn't afford that
luxury. There is no room. A second slot would need 226MB, storage is exactly
1024MB, and none of it is spare. Every partition we might have raided had a
job. Changing the partition table is only an option for future production
runs, which means units already sold would never benefit anyway. And the
window we wanted to close was precisely on those units.

## The Thing That Was Already There

So we changed the question. If we can't create a second system, is there a
second system that already exists?

There was: the recovery partition. Flashed at the factory, untouched during
normal boots, a complete Linux with its own kernel and filesystem. And the
bootloader already contained, compiled in, a branch that boots recovery
instead of the main system when it reads a particular value.

Put together, the change looks like this:

```
[before]  the running system erases and rewrites itself
          └ from the first erase to the last write = the brick window

[after]   stage the image on a separate volume → set a cookie → reboot
          boot into recovery → recovery erases and writes the main system
          → clear the cookie → reboot
          └ if power fails anywhere in between, the cookie is still there,
            so the unit boots back into recovery
```

The whole thing rests on one piece of ordering:

> **The cookie is cleared last.**

Only a finished flash clears the cookie. So if power dies mid-flash, the
cookie is still there on the next boot, the unit lands in recovery again, and
it finishes the job it started. The staged image lives on a separate volume
the flash never touches, so it survives intact.

### When to Give Up, When to Hold On

One more judgment call goes into the design. Things can fail inside recovery
too, and you have to decide when to abort and when to dig in.

| Situation | Action | Why |
|---|---|---|
| Failure before erasing (image verification fails, etc.) | Abort cleanly. Clear the cookie, return to the original system | The main system is still intact. Going back to an intact system is always right |
| Failure after erasing | Retry 5 times; if it still fails, anchor in recovery. Keep the cookie, do not reboot | There is no system to go back to. Rebooting here turns the box into something that power-cycles itself |

The second row matters. A box that keeps power-cycling itself is worse than
a box that sits still. A box that sits still can be rescued over a serial
console; a box that reboots every three seconds can't even be caught.

## But Recovery Collapsed First

With all of this built, we ran the first power-cut test. The unit fell into
an infinite reboot loop. We counted 24 iterations before cutting it off over
the serial console.

Our updater never ran. Not once.

The cause was in the very early stage of the recovery boot. An interrupted
flash leaves the main system partition in an unmountable state. And a vendor
script that runs at recovery startup was written to trigger a reboot when
its attempt to mount that partition failed. Power back on, the partition is
in the same state, the mount fails again, it reboots again. Our updater ran
much later in that sequence, so its turn never came.

You have to laugh at the shape of this: recovery was rebooting itself in
exactly the situation that is its reason to exist. Whoever wrote that script
presumably thought "if we wait, it'll attach eventually." Usually a fine
assumption. But it was a design that never contemplated a partition that can
permanently fail to attach.

### The Fix

All we did was take the two lines in that script consisting solely of
`/sbin/reboot` and replace each with a single kernel-log line. The mount
attempts themselves stayed exactly as they were.

There was a reason we couldn't skip the attempts. Inside that function the
partition numbers increase in order, and one of the volumes attached later
holds the marker file that decides whether our updater is selected at all.
Skip the mounting wholesale and the marker can't be read, and the boot
branches into the vendor's default path instead of our updater. The only
thing that needed fixing was the reboot on failure. Nothing else.

This is modifying someone else's code, and the first file that runs in the
fallback system at that, so we layered safeguards:

- The regex is a full-line match: only a line that is exactly
  `/sbin/reboot` gets replaced. A reboot with arguments, or in any other
  context, is untouched.
- We verify the replacement count matches expectations and that zero
  `reboot` calls remain.
- The modified file is syntax-checked again, and if anything is off we
  automatically roll back to the vendor original. A syntax error in this
  file means losing the fallback system itself.
- The vendor original is always preserved separately.
- If this guard is not in place, the flash does not start at all.

## And Then We Actually Pulled the Plug

The machinery was complete, but machinery is preparation, not verification.
The real question was singular:

> With the guard in place, if we pull the plug mid-flash, does the box come
> back on its own and finish?

So we pulled it, for real, on the exact line where the erase begins. After
that no human hand touched the unit; plugging it back in was the only
intervention.

```
--- first boot after the power cut ---
attempt to attach the system partition → fails
  "... not yet created, rebooting"          ← the vendor's log text remains as-is
  "system volume did not attach -- continuing instead of rebooting"   ← this, instead of a reboot
cache partition attaches                     ← the marker is readable
--- the marker selects our updater ---
  recovery updater starting (cookie=...)
  reboot-loop guard: present
  attempt 2 (committed=1)                    ← the interrupted flash is recognized
  checksums verified
  erasing and writing ... -- the point of no return
  rootfs written / boot image written
  finished: OK (flashed from recovery on attempt 2)
```

| | |
|---|---|
| Power cut → healthy boot | **about 2 min 30 s**, hands off |
| Vendor reboot loops | 0 (24 consecutive in the previous test) |
| State after return | cookie cleared · staging auto-deleted · 0% WAN packet loss |
| Automated acceptance checks | 53 passed / 0 failed |

The brick window is closed. Without A/B partitions, and on units already
sold.

> A note on reading these logs: the vendor's `... rebooting` message remains
> verbatim. What we changed is the actual reboot call beneath it. Whether a
> reboot really happened must be judged from the next line. Miss this, and
> you will read a successful log as a failure.

## The Trap Only Real Hardware Could Catch

There is one thing we learned only after burning an entire upgrade run.

Attach a UBI volume on Linux and a device node like `/dev/ubi7_0` appears.
Or so we believed. In reality the kernel only creates the kernel object; the
node under `/dev` is created by a userspace hotplug daemon.

But by the time our code runs on the upgrade path, every service has already
been shut down. There is no daemon left to create the node. It never
appears.

The symptom was vicious. The kernel log prints, clear as day:

```
ubi7: attached mtd28 (name "recoveryfs")
```

And on the very next line, our code dies:

```
could not attach recoveryfs (mtd28)
```

Attached, says the kernel; could not attach, says our code as it exits. No
amount of staring at the code reveals it, and no amount of ordinary Linux
reasoning derives it. We ended up reading the major:minor numbers straight
out of sysfs and creating the node ourselves.

Where to hook in had the same flavor. We naturally attached our logic to
`platform_do_upgrade` (the function that performs the actual upgrade), but
by then the system has already pivoted to a ramdisk holding only a fixed
list of binaries. No our-script, no `cmp`, no `df`. It only worked after
moving one stage earlier, to `platform_pre_upgrade`.

## Limitations

The feature is finished, but it is not a cure-all. These belong in the post.

1. The *first* upgrade of old units already in the field is still
   dangerous. Those images don't contain this updater, so they flash the
   old way. Only after that upgrade succeeds are they protected. This is
   structural; there is no way around it.
2. A clean flash that doubles as factory reset does not take this path
   yet. Clean flashing signals config wipe through a separate flag, and we
   have not yet measured whether that flag survives the detour through
   recovery. "Clean install that silently keeps old settings" is a worse
   failure than a slow one. Until it's measured, clean flashes take the old
   path.
3. We modify the recovery partition. The vendor original is backed up, but
   the partition now differs from the factory image, a matter that needs to
   be worked out with the production process.
4. On failure, we fall back to the old method. In fact, the very first live
   run was rejected because of the hotplug trap above, and the fallback
   worked, completing the upgrade without damage. We ended up field-proving
   the fallback path by accident.

## Lessons

A/B partitioning is the answer, but it isn't the only answer; it's worth
looking first for a second system already on the board, which in our case
had been there since the factory. The atomicity here is made entirely out of
ordering. Clearing the cookie last is the single decision the whole design
stands on. Recovery logic also has to decide when to give up, because
retrying a doomed operation forever isn't recovery, it's a new failure mode.
And two things about process: before you modify a fallback system, build the
rollback first (we added syntax checking and automatic rollback because that
file is the last safety net, not out of timidity), and at the end you have
to actually pull the plug. Building all the machinery is not verification.
Until we pulled it, we did not know whether any of this worked.
