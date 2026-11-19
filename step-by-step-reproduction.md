# Reproducing the problem step by step

**Companion to the bug report** "after one AHCI command time-out, SYS$PKDDRIVER
never accepts the disk back; the system disk stays in mount verification for
ever". This document only shows how to make the problem happen, with what you
should see at each step. The companion document `step-by-step-evidence.md`
follows the same steps and shows the log and SDA evidence for each one.

The whole sequence takes about five minutes. It shows two defects:

- **Defect A:** after a single write takes longer than about 26 seconds, the
  system disk goes into mount verification and **never comes out**, although
  the disk is working normally again.
- **Defect B:** cancelling that mount verification from the console
  (`IPC> C DKA0`) crashes the system with an **INCONSTATE** bugcheck.

All examples are from our node X86924 (OpenVMS x86-64 E9.2-4), which runs in
the KVM domain `vms-node2` with its system disk as `sda`. Replace these names
with yours.

---

## What you need

1. **A Linux KVM host with libvirt.** Ours: Ubuntu 24.04, QEMU 8.2.2, libvirt
   10.0.0. You need to be able to run `virsh` against the guest (root, or a
   member of the `libvirt` group).

2. **An OpenVMS x86-64 guest whose system disk is on the q35 machine's
   built-in AHCI (SATA) controller.** We have reproduced it on E9.2-4 and on
   V9.2-3 Community. To check the controller:

   On the host:
   ```
   virsh domblklist vms-node2 --details
   virsh dumpxml vms-node2 | grep -A4 "<disk"
   ```
   The system disk is target `sda`, with `bus='sata'`.

   On the guest:
   ```
   show device/full dka0
   ```
   This shows "device type ATA QEMU HARDDISK". In SDA, `CLUE SCSI/SUMMARY` and
   `CLUE SCSI/PORT` show port PKA0 with driver SYS$PKDDRIVER, "AHCI SATA
   controller".

3. **A console you can type on, preferably logged.** Several of the messages
   below appear only on the console, and once the system disk hangs you
   cannot log in any other way. We use the serial console (`virsh console
   vms-node2`) with its output saved to a file with timestamps.

---

## Step 1: make the guest write to its system disk all the time

Log in on the console as SYSTEM and create a small command procedure,
`SYS$LOGIN:H6WRITE.COM`, containing these lines:

```
$ set noon
$ loop:
$ copy sys$library:starlet.olb sys$login:h6_scratch.tmp;1
$ delete sys$login:h6_scratch.tmp;1
$ goto loop
```

Start it on the console:

```
@sys$login:h6write
```

**Why:** the problem starts when a write is in progress at the moment the
disk stalls. The loop copies a large file over and over, so a large write is
almost always outstanding. (Any large file will do; STARLET.OLB is about
36 MB.) We measured from the host about 160 writes a
second, of up to 128 KB each on E9.2-4 (about 62 KB each on V9.2-3), with a
write outstanding about 63% of the time.

**What should happen, and does:** nothing visible. The loop runs quietly. On
the host you can see the writes going through:

```
virsh domblkstat vms-node2 sda
```

Run it twice a few seconds apart; `wr_req` goes up by several hundred.

---

## Step 2: on the host, slow the guest's disk writes right down

```
virsh blkdeviotune vms-node2 sda --write-bytes-sec 4096 --live
```

**What this does:** QEMU now completes the guest's writes at 4,096 bytes a
second. Reads are not limited. The 128 KB write that is in progress therefore
takes about 32 seconds to complete. To the guest this looks like a disk that
has stopped answering for half a minute: the same thing a guest sees during a
storage path fail-over, a busy host, or a snapshot being committed.

On **V9.2-3** use `--write-bytes-sec 1024` instead. Its writes are about 62 KB,
so at 4,096 bytes a second each is held only about 15 seconds, which is below
OpenVMS's time limit; nothing then happens.

**What should happen:** OpenVMS waits for the write. If the wait passes its
time limit, it times the command out, resets the port, and carries on with
the disk as soon as the disk answers again.

**What happens:** nothing visible for about 26 seconds. The loop has simply
stopped making progress.

---

## Step 3: about 26 seconds later, mount verification starts

The console shows:

```
%SYSTEM-I-MOUNTVER, X86924$DKA0: is offline.  Mount verification in progress.
```

In our run of 16-NOV-2026 this came at 06:21:32, 26.5 seconds after the held
write was issued (06:21:05.87).

**Is this correct?** Yes. Up to this point OpenVMS is doing what it should:
the driver has given up on the write, completed it as "medium offline", and
mount verification has started. (The time limit itself is short: VMware
recommends 60 seconds for Windows guests and sets 180 seconds for Linux
guests, but that is a separate question.)

---

## Step 4: remove the limit, so the disk is fully available again

A few seconds after the MOUNTVER message (we waited about 5 seconds), on the
host:

```
virsh blkdeviotune vms-node2 sda --write-bytes-sec 0 --live
virsh blkdeviotune vms-node2 sda
```

The second command lists the limits; all must be 0. The guest's disk now runs
at full speed again.

**What should happen:** within a second or two mount verification succeeds.
The console shows "X86924$DKA0: has completed mount verification", and the
write loop carries on.

**What actually happens:** **nothing, ever.**

- No further message appears on the console.
- The write loop does not continue.
- Anything that needs the system disk hangs: an ssh login gets no greeting,
  and any DCL command that has to activate an image never completes.
- On the host, the guest's disk counters stop moving altogether. Run
  `virsh domblkstat vms-node2 sda` a few times: `rd_req` and `wr_req` do not
  change. (In our run they stood at 1,190,310 reads and 637,191 writes from
  06:21:37 onwards.)
- The virtual disk itself is fine. A host trace shows OpenVMS sending it
  exactly one command a second, ATA CHECK POWER MODE, and the disk answering
  each one at once with "ready" (status 50). No read or write is ever sent to
  it again.
- Mount verification never gives up. We once left a hang of this kind (one
  produced by the host-suspend route at the end of this document) for 74
  minutes, well past MVTIMEOUT (3600 seconds): nothing changed and nothing
  appeared on the console.

**This is defect A.** The disk is healthy and answering, but OpenVMS never
uses it again until the node is rebooted. Because this is the system disk,
the whole node is lost.

---

## Step 5: try to cancel mount verification from the console

On the console press **Ctrl/P**. The IPC menu appears:

```
Interrupt Priority C

Commands:
  C device   Cancel Mount Verification
  Q          Adjust Quorum
  CTRL-P     Prompt for Crash
  CTRL-Z     Exit IPC

IPC>
```

Type:

```
C DKA0
```

and then press **Ctrl/Z** to leave IPC. (On E9.2-4 IPC also leaves by itself,
echoing `^Z`, after about 45 seconds without input. The cancel takes effect
when IPC is left, either way.)

**What should happen:** mount verification is abandoned and the requests
waiting for DKA0 are completed with an error. With its system disk gone the
system may not be usable afterwards, but we would expect an orderly outcome,
not a bugcheck saying that the I/O database is inconsistent.

**What actually happens:** the console shows

```
%SYSTEM-I-MOUNTVER, X86924$DKA0: has aborted mount verification.
```

and at once the system crashes:

```
**** OpenVMS x86_64 Operating System E9.2-4   - BUGCHECK ****

** Bugcheck code = 0000019C: INCONSTATE, Inconsistent I/O data base
```

The crash is always at the same place, SYS$DKDRIVER+80001E44 (module
DKDRIVER_X86 + 00001E44, listing line 20250), whatever process happens to be
current (we have seen ERRFMT, JOB_CONTROL and CPT$performance).

**This is defect B.** We have seen it 3 times out of 3 (14-NOV-2026 04:17,
14-NOV-2026 21:53, 15-NOV-2026 10:23). As a control, on the third hang we
first pressed Ctrl/P and left IPC **without** typing `C DKA0` (15-NOV-2026
10:18): there was no crash, and the system simply stayed hung. So the cancel
is what triggers the crash, not entering or leaving IPC.

Those three hangs were produced by the host-suspend route described at the
end of this document, not by the write limit of step 2. The hung state is the
same in both cases (step 4 of `step-by-step-evidence.md`).

---

## Step 5, alternative: take a dump of the hang instead

To keep the hung state for analysis, press **Ctrl/P** on the console, then at
`IPC>` press **Ctrl/P** again and answer `y`:

```
IPC> ^P
Crash (y/n): y
Starting Crash...
```

The system writes an OPERCRASH dump. The dumps we attach were all taken this
way. Two of them (16-NOV-2026 06:27 and 06:46) are of hangs produced by steps
1 to 4; for the second, our write watchpoint (see `step-by-step-evidence.md`)
was armed during the run.

---

## Step 6: reboot

Boot the guest as usual. It boots normally and the disk works normally.
Nothing is wrong with the disk; only the OpenVMS session that hung is lost.

---

## Where the time limit lies

We ran step 2 with different limits. A write held for 16 seconds is
tolerated; a write held for about 26 seconds is not, and the disk is then lost
for good.

| Run (16-NOV-2026) | Longest a write was outstanding | Result |
|---|---|---|
| E9.2-4, 65,536 then 8,192 bytes/s | 2.0 s, then 16.0 s | no effect |
| E9.2-4, 4,096 bytes/s | port reset 26.52 s after the write was issued | hang, as in step 4 |
| E9.2-4, 4,096 bytes/s, repeated | stopped by our trap 27 s in | hang, as in step 4 |
| V9.2-3, 4,096 bytes/s | 15.5 s | no effect |
| V9.2-3, 1,024 bytes/s | port reset 25.87 s after the write was issued | hang, as in step 4 |

---

## The other way we met this: a host suspend

We first saw the hang when the KVM host was put to sleep (S3) and woken while
the guest had a write outstanding (5 times out of 5). On a host whose TSC the
Linux kernel has marked unstable, KVM moves the guest's TSC forward by the
length of the sleep at the wake. OpenVMS then sees the outstanding write as
hundreds of seconds old and times it out at the next once-a-second check. From
that time-out on, everything is as in steps 3 to 5. The host suspend is only
one way of producing the time-out; steps 1 to 4 produce it with no suspend at
all.

---

## Summary

| Step | What should happen | What happens |
|---|---|---|
| 1. Write loop | Writes go through | As expected |
| 2. Write limit on the host | OpenVMS waits, or times out and recovers | Waits about 26 s |
| 3. Time-out | Mount verification starts | As expected |
| 4. Limit removed | Mount verification completes within seconds | **Never completes**; no read or write is sent to the disk again; node lost |
| 5. `IPC> C DKA0` | Mount verification is cancelled | **INCONSTATE bugcheck** in SYS$DKDRIVER |
| 6. Reboot | Node boots | As expected; the disk is fine |
