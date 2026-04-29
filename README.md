# my-appleRAID

**The Swiss Army knife for disks and RAIDs on macOS.**

One single-file POSIX/dash script that pulls together everything you
actually want when you live with external drives and software RAIDs
on a Mac:

- **Disks** — list, identify (LED blink), spin-down + safe-eject,
  per-disk SMART drill-down (history + temperature + error log +
  self-tests), wipe / verify-wipe, hot-add a replacement.
- **AppleRAID** — list, status, create (mirror / stripe / concat),
  guided repair of a degraded set with size-based replacement guess,
  rename, destroy, AutoRebuild policy management, non-destructive
  test of a set.
- **SoftRAID interop** — migrate a SoftRAID mirror to AppleRAID,
  release a SoftRAID volume, pre/post-flight checks for uninstalling
  SoftRAID.
- **Always-on watchdog** — a launchd-driven `health` monitor that
  catches "set went Degraded", "member returned but won't auto-rebuild",
  "drive disappeared", "SMART attribute crossed threshold",
  "temperature crit", and emails you (rate-limited) + plays an
  audible alarm.
- **One-shot config** — `my-appleRAID create-config <path>` writes a
  fully-commented config; the only fields you really must edit are
  the email-recipient ones, highlighted at the top.

It wraps the low-level `diskutil appleRAID`, `diskutil`, `smartctl`,
`ioreg`, and `launchctl` interfaces in safer, friendlier verbs — and
makes the things macOS hides (e.g. `AppleRAID-AutoRebuild` and
`AppleRAID-SetTimeout`, which `diskutil` does not surface at all)
actually visible and controllable.

The script itself documents every flag via `my-appleRAID --help` and
per-command `my-appleRAID <cmd> --help`. **This README is the empirical
knowledge base** that informs *why* the script behaves the way it does.
Most of what's below is not in Apple's documentation; it was learned
the hard way through controlled testing on real external storage.

---

## tl;dr — what to actually configure

| Knob | my-appleRAID default | Notes |
|---|---|---|
| `AutoRebuild` | **`0` (manual)** for every set — full operator control. The set stays Degraded with a Failed member until you investigate and explicitly start the rebuild via `repair`. **Motivation: any rebuild is a multi-hour, full-disk-read operation that should be a deliberate decision** — was the cable loose, or is the disk truly bad? Is the surviving member healthy enough to drive a rebuild without surfacing latent bad sectors? Did *anything else* on the host glitch at the same time? Auto-rebuild on every reconnect denies you the chance to ask these questions. | Flip to `1` if you'd rather the kernel auto-rebuild silently; the health daemon still emails on every transition and every 6h during the rebuild. |
| `SetTimeout` | Untouched (Apple's default `30 s`) | **No observable effect** on real physical disconnects of these enclosures. Changing it doesn't buy you anything. `my-appleRAID` only reads + displays this, never writes it. |

The behavior is governed by a single config key:

```sh
# DEFAULT_AUTO_REBUILD
#   Controls whether my-appleRAID enforces an AutoRebuild value on
#   every set during CREATE, HEALTH, and LIST calls.
#
#     0   (default) force AutoRebuild=0 on every set the script
#         touches. A returning member stays 'Failed' until you
#         manually run 'my-appleRAID repair'. Motivation: any
#         rebuild is a multi-hour, full-disk-read operation that
#         should be a deliberate decision (was the cable loose?
#         is the disk truly bad? is the survivor healthy enough to
#         drive a rebuild without surfacing latent bad sectors?).
#         Auto-rebuild on every reconnect denies you the chance to
#         ask these questions.
#     1   force AutoRebuild=1 on every set. Returning members
#         auto-rebuild silently. The health daemon still emails on
#         every transition (Degraded onset / Rebuilding start /
#         progress every 6h / resolution), so visibility is
#         preserved - but the kernel will start the rebuild without
#         your input.
#     ""  (empty) disable enforcement entirely. The script reads
#         and displays whatever AutoRebuild value each set
#         currently has, but never writes the field. Use this if
#         you want to manage AutoRebuild by hand per-set via
#         `my-appleRAID autorebuild <set> on|off`.
DEFAULT_AUTO_REBUILD=0
```

Enforcement points:

- **`create`** — after `diskutil appleRAID create` succeeds, the script
  applies `DEFAULT_AUTO_REBUILD` to the new set (skipped if empty).
- **`health`** — every monitoring pass walks the monitored sets and
  re-applies the configured value if any have drifted (skipped if
  empty). Triggered by the launchd daemon, so the enforcement is
  silent and continuous.
- **`list`** — every `my-appleRAID list` invocation checks each set's
  current AutoRebuild against `DEFAULT_AUTO_REBUILD` and re-applies if drifted
  (skipped if empty). May prompt for `sudo` if a write is needed.

---

## Why is the default `AutoRebuild=0`?

A rebuild on AppleRAID is a **full block-by-block copy of the
entire member partition** (no incremental resync — see section 3
below). On a TB-class drive that's many hours of sustained I/O on
both the returning disk and the survivor, with the mirror in
`Degraded` state the entire time. **It is not a casual operation.**
Auto-rebuild on every reconnect lets the kernel start that work
without giving you a chance to ask:

1. **Is the cable loose, or is the disk truly bad?** If a USB or
   Thunderbolt cable wiggle caused the disconnect, plugging it
   back in and immediately initiating a multi-hour rebuild is a
   sledgehammer for what could be a 3-second fix.
2. **Is the surviving member healthy enough to drive a rebuild?**
   A rebuild reads every block of the survivor; latent bad sectors
   that have never been read in the lifetime of the mirror will
   surface NOW, potentially turning a one-disk problem into a
   two-disk problem. Run `my-appleRAID smart` on the survivor first.
3. **Did anything else on the host glitch at the same time?**
   Power supply hiccup, kernel panic, unrelated controller reset?
   You may want to investigate before stressing both disks for
   hours.
4. **Should the returning disk be repaired, or replaced?** If a
   drive is flapping or has prior-failure SMART signs, you'd
   rather swap in a known-good drive than rebuild onto the
   suspect one.

`my-appleRAID`'s default of `AutoRebuild=0` keeps the set in a
**persistent visible Degraded state** with the missing/Failed
member loudly displayed in `list` and `status`, plus emails from
the health daemon, until *you* explicitly run `repair`. The price
of self-healing convenience is too high relative to the size of
the question "should this rebuild run?" — so the default favors
control.

If you want auto-heal anyway (e.g. an unattended off-site mirror
where the rebuild's I/O cost is negligible compared to the
operator's response time), set `DEFAULT_AUTO_REBUILD=1`. The
health daemon's transition emails (Degraded onset, Rebuilding
start, 6 h progress, resolution) will still surface every incident.

---

## Empirically-verified AppleRAID behavior

All findings below come from controlled tests on Darwin 24.6.x
(macOS 2026 hardware) with real external SATA enclosures (HGST
HUH721010ALN604, WDC WD100PURZ-85W86Y0, etc.) plus `hdiutil`-attached
disk images for control comparisons. Test harnesses live under
`tests/raid-behavior.sh` (disk images) and `tests/raid-behavior-real.sh`
(real disks; physical cable yank required).

### 1. The kernel state machine on real disks

When a member disk is **physically yanked** from a 2-disk APFS
mirror:

```
t=0           cable disconnected
t≈0..1s       SetStatus: Online → Degraded (immediate; no Offline transient)
              member slot in plist becomes <missing> (BSD Name=null,
              MemberStatus=null)
              /Volumes/<name> stays MOUNTED, ls succeeds
              writes succeed (rc=0, full content, md5 matches expected)
t≈whenever    cable reconnected
t+15..30s     member slot re-populates with the original partition name,
              MemberStatus depends on AutoRebuild:
                AutoRebuild=0: -> Failed   (set stays Degraded forever
                                            until manual repair)
                AutoRebuild=1: -> Rebuilding  (full block-by-block rebuild
                                              starts; takes hours on
                                              TB-class drives)
```

This contradicts a naive reading of Apple's man page, which suggests
`SetTimeout` provides a "patience window" during which a returning
member can rejoin without rebuild. **In practice, on real hardware,
the kernel goes straight to `Degraded` regardless of `SetTimeout`,
and a returning member is treated as failed regardless of how
briefly it was missing.**

### 2. Disk-image vs real-disk discrepancy

`hdiutil`-attached disk images detached with `hdiutil detach -force`
show a *different* state machine than real disks:

```
Real disk:    Online → Degraded (immediate, on physical disconnect)
Disk image:   Online → Offline (transient, lasting SetTimeout seconds)
                       → Degraded (after SetTimeout fires)
```

The disk-image `Offline` window also kills the mounted volume (`ls`
returns "No such file or directory") and rejects writes. **Neither
of these happens on real hardware.** Disk-image testing is only
useful for confirming the kernel's transition rules between states,
not for predicting real-world I/O availability or mount behavior.

### 3. The AppleRAID storage model — Model B confirmed

When one member is missing during writes, AppleRAID does **Model B**:
writes go to the surviving member only. The returning member's
content is necessarily stale. Because AppleRAID has **no
write-intent bitmap**, the kernel cannot know which blocks
diverged — so on reattach it must either:

- Mark the returning member `Failed` (AutoRebuild=0) and require a full
  `repairMirror` rebuild to bring it back, OR
- Auto-trigger that full rebuild itself (AutoRebuild=1).

**There is no incremental resync mechanism in AppleRAID.** Compare:
mdadm has write-intent-bitmap → fast partial resync; ZFS has
resilver → resync only modified records; SoftRAID has Quick Rebuild
→ resync only changed regions. **AppleRAID has none of these.** Any
rebuild is a full block-by-block copy of the entire member partition.

### 4. `SetTimeout` empirical range

Direct testing on a live mirror, sweeping values from `-1` through
`2147483647` and back:

| Value | Result |
|---|---|
| `-1` | Accepted, persisted as `-1` in ioreg. Semantics undefined by Apple. |
| `0` | Accepted, persisted as `0`. Semantics undefined by Apple. |
| `1` .. `2147483647` (INT32_MAX) | All accepted, all round-trip cleanly through ioreg. |
| `≥ 2147483648` (UINT32_MAX) | Rejected: `Couldn't modify RAID (-69848)`. |

The field is a signed 32-bit integer. Anything in that range is
accepted. But changing the value has **no observable runtime effect**
on physical disconnect behavior. SetTimeout *may* still matter for
soft I/O timeouts (where the kernel isn't sure if the disk is gone),
but that case is hard to synthesize for testing.

### 5. Reading and writing the AppleRAID configurable fields

| Field | Read source | Write source |
|---|---|---|
| `AutoRebuild` | ioreg (`AppleRAIDSet` → `AppleRAID-AutoRebuild`, boolean) | `diskutil appleRAID update AutoRebuild 0\|1 <set-uuid>` |
| `SetTimeout` | ioreg (`AppleRAIDSet` → `AppleRAID-SetTimeout`, int) | `diskutil appleRAID update SetTimeout <N> <set-uuid>` |
| `Name` (set name) | plist (`Name`), human-readable list (`Name:`), ioreg (`AppleRAID-SetName`) | `diskutil appleRAID update SetName <new> <set-uuid>` (undocumented but works) |

`diskutil appleRAID list` (both text and `-plist`) does **not**
expose `SetTimeout` at any output level. **ioreg is the only
userspace source for SetTimeout.** Round-trip parity for all three
fields is empirically confirmed: any value written via
`diskutil appleRAID update` is reflected in ioreg within 1 second.

### 6. `update` on freshly-created mirrors — `-69848` workaround

`diskutil appleRAID update` on a mirror that was created within the
same session sometimes fails with `Couldn't modify RAID (-69848)`,
but only when the value being written equals the current default.
**Workaround:** read the current value from ioreg first, only call
`update` if the new value actually differs. (`my-appleRAID` does
this automatically.)

### 7. Member status vocabulary

| `MemberStatus` | Where it appears | Meaning |
|---|---|---|
| `Online` | plist + ioreg | Healthy, actively serving the mirror |
| `<missing>` (null in plist) | plist | Disk just disappeared; kernel is still adjusting (transient) |
| `Standby` | ioreg | Kernel-internal stub state for a slot whose disk is gone (corresponds to `<missing>` in plist) |
| `Failed` | plist | Disk reattached after timeout with AutoRebuild=0 — permanent until manual `repairMirror` |
| `Rebuilding` | plist (with `<N>% (rebuilding)` text in the human-readable list) | AutoRebuild=1 triggered a rebuild; full rebuild in progress |

The plist and ioreg can disagree during transitions (plist says
`Failed`, ioreg says `Standby` for the same member). `my-appleRAID`
parses both and treats `Failed` / `Standby` as equivalent failure
states for repair-detection purposes.

### 8. Set-level status

| `Status` | Meaning |
|---|---|
| `Online` | All members healthy |
| `Degraded` | One or more members missing/failed; mirror still serving from survivors |
| `Offline` | Set unavailable. Observed only on disk-image tests, not real hardware. |

### 9. AppleRAID partition-table requirements

AppleRAID requires **GPT** (`GUID_partition_scheme`) on each member
disk. It will refuse / silently malfunction on a legacy
`Apple_partition_scheme` (APM):

- `hdiutil create -layout SPUD ...` produces APM → `appleRAID create`
  succeeds nominally but the resulting set is born `Offline` and
  rejects all subsequent `update` calls with `-69848`.
- `hdiutil create -layout GPTSPUD ...` produces GPT → works.

When AppleRAID creates a member out of a whole disk it lays out three
partitions: `EFI` (~210 MB), `Apple_RAID` (the data), `Apple_Boot`
(~134 MB). After a set is destroyed, the data partition's type
changes to `Apple_RAID_Offline` — those former members are not
useful as fresh members until wiped (`my-appleRAID wipe <N>`).

### 10. APFS-on-AppleRAID layered architecture

A typical setup is **APFS volume → AppleRAID synthesized device →
two physical members**. The user-visible mount lives on a separate
synthesized disk identifier (e.g. `disk10s1`) than the AppleRAID
virtual device (e.g. `disk9`). `diskutil` and `mount` show:

```
/dev/disk9 (external, virtual):       ← AppleRAID synthesized device
   0:                Apple_APFS                +10.0 TB    disk9
/dev/disk10s1 on /Volumes/<name> (apfs, local, journaled)  ← user mount
```

This means there is **no single-`diskutil-info` lookup** that gives
you both "is the AppleRAID set healthy?" and "is its volume
mounted?" — you have to walk the chain.

### 11. AppleRAID supports Mirror, Stripe, and Concat only

The kernel only knows three layouts: `Mirror`, `Stripe`, `Concat`.
No RAID 5/6/10/JBOD; no parity, no nested RAIDs except
"AppleRAID-on-AppleRAID" via UUID nesting (rarely used).

### 12. Failed-member recovery — what `repair` actually does

There is no in-place "rejoin" path for a `Failed` member. The full
recipe is:

```
sudo diskutil appleRAID remove <member-uuid> <set-uuid>   # release the kernel claim
sudo diskutil unmountDisk force /dev/diskN                 # belt-and-braces unmount
[ my-appleRAID wipe N ]                                    # ONLY if SoftRAID metadata present
sudo diskutil appleRAID add member /dev/diskN <set-uuid>   # repartition + add + START rebuild
```

`my-appleRAID repair` runs all of this interactively, with a single
"yes" confirmation. The pre-wipe step is **conditional**: it fires
only when `list_softraid_disks` finds SoftRAID metadata on the disk
(SoftRAID's signature is sticky enough that the GPT-layout step in
`add member` can fail or produce a confused layout otherwise). For
vanilla AppleRAID-leftover or fresh disks the wipe is skipped —
`diskutil` does its own GPT layout and a 100 MB pre-wipe is wear
without benefit.

**Why `add member` and NOT `repairMirror`.** Empirically (verified on
real disks April 2026), `diskutil appleRAID repairMirror <set>
/dev/diskN` adds the new disk as a hot `Spare`, not a Member. With
`AutoRebuild=No` the Spare sits inert; the kernel only promotes a
Spare to a Rebuilding member while `AutoRebuild=Yes`. Apple's man
page describes `repairMirror` as adding-as-member, but the observed
behavior is "added as Spare", and `diskutil appleRAID list` shows the
set as `Status: Online` even while the new Spare is doing nothing —
which is misleading and dangerous (you think the rebuild is running
but the mirror is still effectively single-member).

`diskutil appleRAID add member <device> <set>` does NOT have that
problem. It adds the disk as a Member and starts the rebuild as part
of the same operation, regardless of AutoRebuild. Apple's man page
explicitly contrasts the two `add` subtypes:
> "Adding a hot spare to a mirror will enable autorebuilding for that
>  mirror." (i.e. only the spare path depends on AutoRebuild)

`my-appleRAID repair` therefore uses `add member`, not `repairMirror`,
in step 3 of the flow above.

**The `AutoRebuild=1 nudge` is NOT a viable shortcut.** An earlier
version of `my-appleRAID repair` offered a path that flipped
`AutoRebuild` to `Yes` and then waited for the kernel to start the
rebuild on the same disk in place. Empirical testing on a real
Degraded set with a logically-dissociated Failed member (disk19s2
of `weg`, after a yank+reattach cycle) showed three failure modes
that make this path unsafe:

1. **`diskutil appleRAID update AutoRebuild 1 <set>` blocks for
   minutes** instead of the sub-second return seen on healthy sets.
   The kernel reacts to the metadata write by trying to evaluate
   the failed slot, and the syscall doesn't return until that
   evaluation completes.
2. **The userspace disk daemons get wedged.** Once `diskutil update`
   is in that long-blocked state, even read-only `diskutil appleRAID
   list` from another shell blocks indefinitely — `diskmanagementd`
   / `diskarbitrationd` are stuck waiting on the same kernel call.
   With SIP enabled, `launchctl kickstart -k` is refused
   ("Operation not permitted while System Integrity Protection is
   engaged"), so there is **no userspace recovery**: a reboot is
   the only way out.
3. **Even when AR=Yes is committed, the rebuild does NOT start.**
   `ioreg -rc AppleRAIDSet` confirmed the metadata write reached
   the kernel (`AppleRAID-AutoRebuild = Yes`), but the set stayed
   `Degraded` and the failed member never transitioned to
   `Rebuilding`. Apparently the kernel had already removed the
   failed slot from its rebuild-candidate list when the member
   went Failed, and re-enabling AR doesn't re-evaluate that slot.

The remove + wipe + repairMirror flow avoids all three: it
explicitly tears down the failed slot (clearing the kernel's
candidate list), wipes the disk so `repairMirror` doesn't refuse
the leftover metadata, and re-adds the disk as a fresh member
which the kernel definitely *will* rebuild. It's also the
documented Apple flow (man page + Apple-Community thread #7650392).

**Important: `repairMirror` adds a Spare, not a Member — `AutoRebuild=1`
is required to promote it.** Apple's man page is precise about this
("Adds a hot spare device to a (pre-existing) mirror set"), but the
implication is easy to miss: after `repairMirror /dev/diskN` the new
disk shows up as `Spare` (not `Rebuilding`), and the kernel only
promotes it to a `Rebuilding` member while `AutoRebuild=Yes`. With
`AutoRebuild=No`, the Spare sits inert; the mirror stays a single-
member set with one orphan Spare attached, and `diskutil appleRAID
list` will misleadingly report it as `Status: Online`.

**Consequence for `DEFAULT_AUTO_REBUILD=0` users:** if the health
daemon's `enforce_default_ar_for_uuid` runs between `repairMirror`
finishing and the kernel finishing the Spare-to-Rebuilding promotion,
it will silently abort the rebuild. `my-appleRAID` therefore skips
the AR=1→0 enforcement flip whenever ANY member is in a `Spare`,
`Stale`, or `Rebuilding` state; the next health pass re-evaluates
once the rebuild has completed and the set is fully Online.

### 13. `diskutil eject` of an active RAID member

Plain `diskutil eject /dev/diskN` **fails** for a disk that's still
a member of a live AppleRAID set ("kernel claim held"). To eject
cleanly you must `appleRAID remove` the member first. `my-appleRAID
eject` does this in a single guided step (with a confirmation
warning that removing a member degrades the set).

If you only want to **temporarily disconnect** a member (e.g. test
a cable), do **not** use `eject` — just physically yank the cable.
The set will go `Degraded`, the volume stays mounted, writes go to
the survivor, and on reconnect AutoRebuild=1 will rebuild while AutoRebuild=0 leaves
it `Failed`.

### 14. Identifying physical members

The `disks` command's compact view (`my-appleRAID disks`) shows one
line per whole disk with `DEV / LOC / KIND / SIZE / MODEL / ROLE`.
The `ROLE` column says `RAID member (<setname>)` for active
members, `RAID virtual (<setname>)` for the synthesized RAID
device, `APFS` for plain APFS volumes, or `-` for free disks. With
`-V`, the table adds SERIAL / TEMP / SMART / ROT / TRIM / PROTOCOL.

To physically identify which drive in an enclosure is `diskN`, use
`my-appleRAID blink N` (or with no arg, an interactive picker).
This generates a read-only I/O loop that pulses the drive's
activity LED — the same trick SoftRAID's "Identify" button uses.
macOS exposes no API to toggle an enclosure LED directly; bus
activity is the only available signal.

### 15. SMART drill-down

`my-appleRAID smart <disk>` (or interactive picker) prints:

1. Overall health (`smartctl -H`)
2. Pre-failure attribute table with non-zero "killer" highlight
3. Self-test log with calendar dates derived from
   `Power_On_Hours - LifeTime`
4. Firmware error log with same date annotation
5. **Summary highlights + verdict** — one-screen recap of overall
   health, age (POH → years), the five killer attributes, helium
   level, self-test count + last result, and error-log count;
   followed by a single-line verdict: `HEALTHY`, `WATCH`, or
   `RETIRE-CANDIDATE`. The verdict logic mirrors the priority
   ordering used by `smart <a> <b>` comparison (health → self-test
   failure → killers 5/197/198 → error-log entries → 187 → UDMA_CRC
   → age threshold), so a single-drive verdict and a two-drive
   verdict use the same evidence weighting.

The killer attributes whose RAW value should always be `0`:

| ID | Name |
|---|---|
| 5 | Reallocated_Sector_Ct |
| 187 | Reported_Uncorrect |
| 197 | Current_Pending_Sector |
| 198 | Offline_Uncorrectable |
| 199 | UDMA_CRC_Error_Count |

For NVMe drives: `Critical Warning != 0x00`,
`Media and Data Integrity Errors > 0`, `Available Spare < 100%`.

`smartctl -d <type>` is auto-detected per enclosure and cached in
`$STATE_DIR/smart-dtype-cache` (most external SATA bridges need
`-d sat`).

### 16. SetTimeout and AutoRebuild are NOT exposed in `diskutil appleRAID list`

This was a load-bearing finding. Both fields are completely absent
from `diskutil appleRAID list` (text format) and `diskutil
appleRAID list -plist` (plist format), regardless of what value is
configured. The kernel persists them somewhere we don't have a
direct file path for, but ioreg surfaces them as
`AppleRAID-AutoRebuild` and `AppleRAID-SetTimeout` properties on
the `AppleRAIDSet` IOService entries. **ioreg is therefore the only
userspace read path** for these two fields.

### 17. Why "auto-heal in RAM" is not a thing

If AppleRAID buffered writes in RAM during a member-missing window
and replayed them on reattach (Model C in the analysis), a power
loss during that window would lose the writes silently — even
though the kernel had already acknowledged them to the application.
No durable RAID implementation does this. AppleRAID is no
exception. Hence Model B (write to survivor only) is the correct
mental model.

---

## Health monitor (`my-appleRAID health`)

The launchd-driven health monitor emits an email **on every state
transition**, so even with `AutoRebuild=1` (silent self-heal) you
never lose visibility into incidents:

- Set Online → **Degraded** (member missing) — `crit` email + sound
  on first detection, then `EMAIL_INTERVAL` (default 4 h) recurring.
- Member → **Failed** (returning member after AutoRebuild=0 timeout, or hard
  failure) — `crit` email + sound; same recurring cadence as Degraded.
- Member → **Rebuilding** — `warn` email on first detection, then
  `REBUILD_EMAIL_INTERVAL` (default 6 h = 4×/day) recurring while
  the rebuild is in progress.
- State clears (member back to Online, set back to Online) — one
  resolution email (`[appleRAID/OK] Resolved: ...`).

Why two separate intervals: a multi-day full rebuild on a TB-class
mirror would generate ~30+ recurring emails at the regular 4 h
cadence. `REBUILD_EMAIL_INTERVAL=21600` (6 h) keeps progress
visibility without spamming. Set it to a value larger than the
typical rebuild duration (e.g. `48h`) if you only want the
start + completion emails.

The monitor does **not** rely on `SetTimeout` for any decision. It
honors `DEFAULT_AUTO_REBUILD`: by default `0` (manual rebuild) so
you get the persistent visible Degraded+Failed state until you
manually run `repair`. Flip to `1` if you'd rather the kernel
auto-rebuild silently — the daemon's emails above will still
surface every transition.

---

## Test harnesses

Two scripts live in `tests/`:

- **`raid-behavior.sh`** — builds 200 MB hdiutil disk-image mirrors,
  runs three timing scenarios using `hdiutil detach -force` to
  simulate member disconnects. Useful for verifying the kernel's
  plist / ioreg state machine and exit codes. **Does not touch real
  disks.** Be aware of the disk-image-vs-real-disk discrepancy
  documented in section 2 — this test is *less* faithful to
  real-world behavior than its disk-image basis would suggest.

- **`raid-behavior-real.sh`** — operates **only** on `/dev/disk7`
  and `/dev/disk8` (with explicit confirmation) and requires the
  operator to physically yank/replug the disk7 cable at prompted
  times. This is the only way to reproduce real-world disconnect
  semantics; software detach of an active RAID member requires
  `appleRAID remove` first, which is destructive (membership-
  severing), not a transient-missing simulation.

Both write timestamped logs to `/tmp/raid-behavior-*.log`.
