# my-appleRAID

A management helper for **macOS AppleRAID** (mirror, stripe, concat) that
wraps the low-level `diskutil appleRAID` interface in safer, friendlier
verbs and adds a launchd-driven health monitor.

The script itself documents every flag via `my-appleRAID --help` and
per-command `my-appleRAID <cmd> --help`. **This README is the empirical
knowledge base** that informs *why* the script behaves the way it does.
Most of what's below is not in Apple's documentation; it was learned
the hard way through controlled testing on real external storage.

---

## tl;dr — what to actually configure

| Knob | Recommended | Reason |
|---|---|---|
| `AutoRebuild` | **`1` (automatic)** for every set | The only effective recovery knob. With `0`, a returning member stays `Failed` until you manually run `repairMirror` — which is a full rebuild anyway. |
| `SetTimeout` | Leave at Apple's default `30 s` | **No observable effect** on real physical disconnects of these enclosures. Changing it doesn't buy you anything. |

`my-appleRAID`'s defaults follow this advice: it auto-enables
`AutoRebuild=1` on every set it sees, and it does not modify
`SetTimeout` (only displays it, for diagnostic visibility). The
behavior is governed by a single config key:

```sh
# DEFAULT_AUTO_REBUILD
#   Controls whether my-appleRAID enforces an AutoRebuild value on
#   every set during CREATE, HEALTH, and LIST calls.
#
#     1   (default, recommended) — set AR=1 on every set the script
#         touches. Sets created via `my-appleRAID create` are flipped
#         from Apple's default of 0 to 1 right after creation. Health
#         passes ensure no set has drifted off. List calls print the
#         current value and re-apply if drifted.
#     0   force AR=0 on every set the script touches. Same enforcement
#         points; opposite target value. Useful only for the narrow
#         "I want hard alerts / explicit control" cases below.
#     ""  (empty) disable enforcement entirely. The script reads and
#         displays whatever AR each set currently has, but never
#         writes the field. Use this if you want to manage AR by
#         hand per-set via `my-appleRAID autorebuild <set> on|off`.
DEFAULT_AUTO_REBUILD=1
```

Enforcement points:

- **`create`** — after `diskutil appleRAID create` succeeds, the script
  applies `DEFAULT_AUTO_REBUILD` to the new set (skipped if empty).
- **`health`** — every monitoring pass walks the monitored sets and
  re-applies the configured value if any have drifted (skipped if
  empty). Triggered by the launchd daemon, so the enforcement is
  silent and continuous.
- **`list`** — every `my-appleRAID list` invocation checks each set's
  current AR against `DEFAULT_AUTO_REBUILD` and re-applies if drifted
  (skipped if empty). May prompt for `sudo` if a write is needed.

---

## "Why would I ever want `AutoRebuild=0`?"

Short answer: **you usually wouldn't**, and the script's default is `1`.

Long answer — narrow cases where `0` makes sense:

1. **You want a hard alert that requires action.** With `AutoRebuild=0`,
   a returning member sits in `Failed` status until you act. The set
   stays `Degraded`. That's a persistent, visible alert. With
   `AutoRebuild=1`, the rebuild starts automatically and the set is
   back to `Online` after a few hours — silent self-healing, but you
   may miss the fact that there was an incident at all.
2. **You want explicit control over when a rebuild runs.** A full
   rebuild on a multi-TB mirror is hours of sustained I/O on both
   members and degrades read/write performance for everything else
   on the host. With `0`, you can defer the rebuild to off-hours
   (e.g. start it manually overnight via `my-appleRAID repair`).
3. **You suspect the returning member itself is bad.** If a disk is
   flapping, you may not want the kernel to keep re-rebuilding onto
   a dying drive. With `0`, you decide whether to repair it back in
   or replace it with a known-good disk via `my-appleRAID repair`.

For ordinary home / SOHO use, none of these outweigh the
self-healing benefit of `AutoRebuild=1`. That's the script's default
recommendation.

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

- Mark the returning member `Failed` (AR=0) and require a full
  `repairMirror` rebuild to bring it back, OR
- Auto-trigger that full rebuild itself (AR=1).

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
| `Failed` | plist | Disk reattached after timeout with AR=0 — permanent until manual `repairMirror` |
| `Rebuilding` | plist (with `<N>% (rebuilding)` text in the human-readable list) | AR=1 triggered a rebuild; full rebuild in progress |

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
my-appleRAID wipe N                                        # zero first/last 100 MB
sudo diskutil appleRAID repairMirror <set-uuid> /dev/diskN # repartition + add + sync
```

`my-appleRAID repair` runs all of this interactively, with a single
"yes" confirmation. The wipe is necessary because `repairMirror`
does its own GPT layout and refuses members with leftover Apple_RAID
metadata.

### 13. `diskutil eject` of an active RAID member

Plain `diskutil eject /dev/diskN` **fails** for a disk that's still
a member of a live AppleRAID set ("kernel claim held"). To eject
cleanly you must `appleRAID remove` the member first. `my-appleRAID
eject` does this in a single guided step (with a confirmation
warning that removing a member degrades the set).

If you only want to **temporarily disconnect** a member (e.g. test
a cable), do **not** use `eject` — just physically yank the cable.
The set will go `Degraded`, the volume stays mounted, writes go to
the survivor, and on reconnect AR=1 will rebuild while AR=0 leaves
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

The launchd-driven health monitor acts on user-visible state
transitions:

- Set goes `Degraded` → alert (sound + email + log).
- Member becomes `Failed` → alert (this is the "permanent" state
  where manual recovery is needed).
- Member becomes `Rebuilding` → info, not an alert (kernel is
  healing).

It does **not** rely on `SetTimeout` for any decision. It does
honor `AutoRebuild`: with `AUTO_ENABLE_AUTOREBUILD=yes` (the
default), the daemon ensures every monitored set has
`AutoRebuild=1`, so transient disconnects auto-heal via rebuild
without operator intervention.

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
