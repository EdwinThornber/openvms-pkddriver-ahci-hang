# VSI bug report (E9.2-4 field test, also V9.2-3): after one AHCI command time-out, SYS$PKDDRIVER never accepts the disk back; the system disk stays in mount verification for ever

Status: **published 19-NOV-2026** for VSI and the VSI forum. Written
16-NOV-2026, amended 17-NOV-2026 (cluster warning in section 2, question 5).
Tests from 18 and 19 November repeated it with a plain storage stall
(`dmsetup suspend`) on a data disk: the same hang, and the same INCONSTATE
crash at the same PC. The same stall behind virtio-scsi (SYS$VSPDRIVER, the
same SYS$DKDRIVER) did no harm, even for 11 hours. The results are in the
README; the files are listed in [MANIFEST.txt](MANIFEST.txt).

---

**Product:** OpenVMS x86-64 E9.2-4 (field test) and V9.2-3 Community.
SYS$PKDDRIVER (AHCI SATA port; modules PKDDRIVER, SATL, SATL_SUPPORT), with
SYS$DKDRIVER and mount verification.

**Reported by:** Edwin Thornber, VSI OpenVMS Ambassador, E9.2-4 field test
and OpenVMS V9.2-3.

**Systems:** X86924 (E9.2-4) and X86923 (V9.2-3 Community), x86-64 KVM
guests on a Dell Precision 7810. Details in section 3.

**Criticality:** major problem. The system disk hangs permanently and the node
must be crashed and rebooted. Our systems are test systems, not production.

**Repeatable:** yes, every time, in about 30 seconds (section 4): 3 of 3
attempts past the time limit on the two versions, plus 5 of 5 host suspends
with a write in flight.

## 1. Summary

If a single command to an AHCI SATA disk is outstanding for longer than the
driver's time limit (about 26 seconds in our tests), the command is timed out,
SYS$PKDDRIVER resets the port, and from then on the driver refuses every read
and write to that disk with SS$_DEVCMDERR without sending it. The disk itself
is healthy and answers every command it is given. Mount verification retries
once a second and never completes. When the disk is the system disk the node
is lost until it is crashed and rebooted.

We can reproduce this in about 30 seconds on a KVM guest, on E9.2-4 and on
V9.2-3 alike. On E9.2-4 we have traced the stuck state to one longword in the
port driver's per-disk block: a bit that is set when the disk is first
configured, cleared when a command times out, and not set again by the
recovery after the port reset. Module, offset and listing line for each of
those places are in section 6.

We first met the problem as "the system disk hangs when the KVM host resumes
from suspend". That is one way of producing the time-out (section 7); the
host suspend is not needed.

## 2. Impact

- **An ordinary storage event becomes a lost node.** A virtual disk that stops
  answering for half a minute and then recovers is routine for a guest: a
  storage path fail-over, an NFS or iSCSI reconnect, a host disk in error
  recovery, an overloaded host, a snapshot being committed, an I/O limit.
  OpenVMS's time-out and mount verification are the right response. The
  defect is that, once a disk is timed out, the disk is never accepted back.
  Thus a stall of 26 seconds or more means the timed-out disk is unusable
  until the next reboot. If the timed-out disk happens to be the OpenVMS
  system disk, then the whole node is lost.
- **Warning: in a cluster the damage does not stop at the one node.** If the
  timed-out disk is MSCP-served to other members of an OpenVMS Cluster, the
  serving node can no longer read or write it, so every member that mounts
  the disk through that node loses it too, until the serving node is
  rebooted. A SATA disk has only the one host, so there is no second direct
  path to fail over to. A storage stall of half a minute on one node can
  therefore put the viability of the whole cluster at risk. We expect this
  from the mechanism in sections 5 and 6; we have not yet tested it in a
  cluster.
- **Other guests are expected to ride such stalls out.** VMware Tools sets the
  disk time-out of Linux guests to 180 seconds (VMware KB 1009465), and
  VMware's storage documentation recommends 60 seconds for Windows guests "to
  avoid disruptions during a path failover". OpenVMS gave up after about 26
  seconds in our tests, and then did not recover.
- **It is in the released product.** V9.2-3 Community shows the same time
  limit, the same port reset sequence and the same permanent hang as the
  E9.2-4 field test.
- **It does not depend on our host.** The held-write test hangs the disk with
  the host's TSC stable or unstable; only the host-suspend route to the
  time-out needs the unstable TSC (section 7).
- **Nothing of the incident is recorded by OpenVMS.** The device's error
  count stays 0, CLUE SCSI shows no resets, retries or events, and with the
  system disk stuck ERRFMT cannot write the error log. Afterwards the
  operator sees only "Mount verification in progress". Cancelling the mount
  verification with `IPC> C DKA0` gives an INCONSTATE bugcheck.
- **What we have and have not tested.** Tested: KVM/QEMU guests with the disk
  on the AHCI SATA controller, where every attempt past the limit failed (2
  of 2 on E9.2-4 and 1 of 1 on V9.2-3 with a held write, and 5 of 5 host
  suspends with a write in flight on E9.2-4). Not tested: VMware's AHCI
  controller, and disks on other port drivers (our ESXi guests have their
  system disks on LSI Logic Parallel, SYS$PKMDRIVER). **We are continuing
  the investigation** to find out whether other port drivers and other
  controllers have the same problem, and will add the results to this
  report.

## 3. System

- Node X86924, OpenVMS x86-64 **E9.2-4**, no patches. All internal detail in
  sections 5 and 6 (stacks, addresses, listing lines) is from this node.
- Node X86923, OpenVMS x86-64 **V9.2-3 Community**, in a second guest on the
  same host with the same definition (section 4 only).
- KVM guest: QEMU 8.2.2, libvirt 10.0.0, Ubuntu 24.04 host, Linux 7.0.0-43
  (Dell Precision 7810, 2 x Xeon E5-2690 v3). Machine pc-q35-noble, OVMF
  (UEFI), host-passthrough CPU, 2 vCPUs, 8 GiB. Ubuntu is not one of VSI's
  tested KVM hosts (Oracle Linux 9.7, RHEL 9.8).
- System disk: qcow2, `cache='writethrough'`, on the q35 built-in AHCI
  controller (PCI 8086:2922), seen as DKA0 on PKA0. One port, one disk.

## 4. How to reproduce (no host suspend)

1. On the guest, keep the disk busy with large writes. We use a DCL loop:

   ```
   $ set noon
   $ loop:
   $ copy sys$library:starlet.olb sys$login:h6_scratch.tmp;1
   $ delete sys$login:h6_scratch.tmp;1
   $ goto loop
   ```

   This gives about 160 writes a second of up to 128 KB each.

2. On the host, limit the guest disk's write rate so that one 128 KB write is
   held for about 32 seconds:

   ```
   virsh blkdeviotune <domain> sda --write-bytes-sec 4096 --live
   ```

3. About 26 seconds later the guest's console prints
   `%SYSTEM-I-MOUNTVER, X86924$DKA0: is offline.  Mount verification in progress.`

4. Remove the limit (`--write-bytes-sec 0`). The disk is (or should be)
   fully available again, but the affected guest disk never leaves mount
   verification.

| Run (16-NOV-2026) | Longest time a write was outstanding | Result |
|---|---|---|
| E9.2-4, limit 65,536 B/s, then 8,192 B/s | 2.0 s (9 times), then 16.0 s (3 times) | no effect; no time-out, no port reset |
| E9.2-4, limit 4,096 B/s | port reset 26.52 s after the held write was issued | mount verification, permanent |
| E9.2-4, limit 4,096 B/s, repeated | stopped by our trap 27 s after the limit was set | mount verification, permanent |
| V9.2-3, limit 4,096 B/s (its writes are about 62 KB) | 15.5 s (4 times) | no effect; no time-out, no port reset |
| V9.2-3, limit 1,024 B/s | port reset 25.87 s after the held write was issued | mount verification, permanent |

So the limit lies between 16.0 and about 26 seconds on both versions. On
V9.2-3 the console shows the same MOUNTVER message (and the OPCOM message
"Device X86923$DKA0: is offline. Mount verification is in progress."), the
AHCI register sequence of the port reset is the same step for step as in
section 5, including PxTFD read twice as 00000130, and afterwards only CHECK
POWER MODE reaches the disk.

## 5. What happens in detail, step by step

All of this is observed, either in a host-side trace of the guest's AHCI
register accesses or in memory images and dumps taken at the points named.

**Time-out and the two failures.** The once-a-second routine EXE$TIMEOUT calls
the port driver, which resumes the SYS$DKDRIVER thread of the held write. Stack
at that moment (identical in six captures) is as follows:

```
EXE$TIMEOUT_C+0037E                    (SYSTEM_PRIMITIVES, image offset 80061FDE)
  PKDDRIVER + 00003B3D, line 92741     in the routine at PKDDRIVER + 00003998, line 92631
    PKDDRIVER + 00005394, line 92830   in the routine at PKDDRIVER + 000052A8, line 92778; resumes the class driver's thread
      DKDRIVER_C + 000047C7, line 77598      in the routine at DKDRIVER_C + 00004310, line 77431
        DKDRIVER_C + 0000606C, line 76652    in the routine at DKDRIVER_C + 00006050, line 76704
          DKDRIVER_X86 + 000021B7, line 20309  in the routine at DKDRIVER_X86 + 00001920, line 20232
            DKDRIVER_X86 + 0000DCE1, line 27579  in the routine at DKDRIVER_X86 + 0000DBB0, line 27514
```

(Module, offset and line from SDA `MAP`; routine entry points from the images'
unwind tables. The last line is the instruction pointer, the others are return
addresses. The PKDDRIVER lines are in image SYS$PKDDRIVER, the DKDRIVER lines
in SYS$DKDRIVER.) At DKDRIVER_X86 + 0000DCE1 UCB$L_DK_MEDOFL is incremented
and the write is completed through IOC_STD$REQCOM with a "medium offline"
status. That starts mount verification
(EXE_STD$MOUNT_VER, MNTVER_STD$PACKACK_VOL, EXE$MNTVERSIO,
IOC_STD$INITIATE_NEW_IO). Its IO$_PACKACK (function 0808) runs synchronously,
still inside the same call from EXE$TIMEOUT. SYS$DKDRIVER builds an INQUIRY
(CDB 12 00 00 00 FF 00), the port driver refuses it without issuing any ATA
command, and the PACKACK ends with IRP$L_IOST1 = SS$_MEDOFL (UCB$L_DK_MEDOFL
is now 2).

**Port reset.** Next the port driver resets the AHCI port. From the trace
(times from the first access):

| Time | Register access |
|---|---|
| 0.000 ms | PxCMD read C017, written C016 (ST cleared), read 4016 |
| +0.14 ms | PxIE 7D000053 -> 3D000013 |
| +0.35 ms | PxSCTL 300 -> 301 (COMRESET asserted) |
| +22.2 ms | PxSCTL 301 -> 300 |
| after the interrupt | PxSSTS read 113 (three times); PxSERR and PxIS cleared; PxIE restored |
| | **PxTFD read twice: 00000130** (status 30, error 01) |
| | PxCMD 4016 -> 4017 (ST set) |
| then | one PIO data-in command (PxIS = 3), one non-data command (PxIS = 1) |

**The hang.** After that the guest issues exactly one command a second, CHECK
POWER MODE (E5), and each completes with status 50. No read or write command
is issued again. In a memory image taken 2.5 minutes into the hang:

- DKA0: `online,bsy,valid,unload,mntverip`; UCB$L_IRP 0; 12 IRPs in the I/O
  request queue; UCB$L_DK_MEDOFL still 2.
- Each second mount verification issues a one-block read with IO$M_INHERLOG
  (function 080C) of LBN 31 hex. SYS$DKDRIVER builds READ(10)
  (28 00 00 00 00 31 00 00 01). It is not sent to the disk, and the IRP ends
  with **IRP$L_IOST1 = 0000032C, SS$_DEVCMDERR**.
- CLUE SCSI: connection open, nothing outstanding, all error counters 0, no
  bus resets counted.

`IPC> C DKA0` in this state gives an INCONSTATE bugcheck in SYS$DKDRIVER
(seen three times; details in the earlier draft).

## 6. Where the state is lost

SCDT$L_PORT_SPECIFIC points to a per-disk block owned by the port driver. Its
longword at offset +10 is **0000000A on a healthy system and 00000002 in the
hang**. Apart from running statistics in the SCDT, it is the only field that
differs between a healthy and a hung system in the SCDT, the STDT and that
block (compared in the same boot, 20 minutes apart).

We put a hardware write trap on that longword, across a boot and across a
time-out. These are all the writes the driver makes to it. Module, offset and
line are from SDA `MAP` on image SYS$PKDDRIVER. The shipped image has no
symbol table, so we cannot give routine names; each routine's entry point is
taken from the image's unwind tables instead, and its first listing line
should identify it.

| When | What is written | Routine entry point | The write itself |
|---|---|---|---|
| Boot, disk configured | field = 8 | SATL_SUPPORT + 00000360, line 46978 (image offset 80029680, 117 bytes) | SATL_SUPPORT + 000003C9, line 46999 |
| Boot, disk identified | 8 -> A (adds 2) | SATL + 00000590, line 40488 (image offset 80027EF0, 799 bytes) | SATL + 00000812, line 40540 |
| Command time-out | A -> 0 | SATL_SUPPORT + 000003E0, line 47018 (image offset 80029700, 221 bytes) | SATL_SUPPORT + 00000404, line 47025 |
| About 1 s after the port reset | 0 -> 2 (adds 2) | SATL + 00000590, line 40488 (as at boot) | SATL + 00000812, line 40540 |
| Afterwards | nothing writes it again | | |

Callers, as return address (routine entry point):

| Routine called | Called from |
|---|---|
| Configuration, SATL_SUPPORT line 46978 | PKDDRIVER + 0000240F, line 89682 (routine at PKDDRIVER + 000022B8, line 89589), itself called from PKDDRIVER + 00006E3D, line 86787 (routine at + 00006C78, line 86706) |
| Time-out clear, SATL_SUPPORT line 47018 | PKDDRIVER + 00006101, line 92952 (routine at PKDDRIVER + 00005F98, line 92900) |
| Identify-time, SATL line 40488 | SATL + 00000C52, line 40614 (routine at + 00000BA0, line 40603); SATL + 000002B2, line 40165 (routine at + 00000190, line 40148); SATL + 0000012F, line 40128 (routine at + 000000F0, line 40095) |

So bit 3 is set only by the configuration routine (SATL_SUPPORT line 46978,
write at line 46999). The time-out routine that follows it in the same module
(line 47018) clears the whole field at line 47025, and the recovery after the
port reset runs only the identify-time routine (SATL line 40488), which
restores bit 1 at line 40540. Bit 3 stays clear, and that coincides exactly
with the driver refusing all reads and writes.

The clear at line 47025 happens after both "medium offline" completions and
before the first register access of the port reset.

## 7. How a host suspend produces the same time-out

On our host the TSC was marked unstable in one boot (the kernel's TSC
synchronisation check measured a 56-cycle warp between two CPUs). KVM then
makes the guest's TSC jump forward by the length of the sleep when the host
resumes from S3, and OpenVMS's uptime (EXE$GL_ABSTIM) includes the sleep. A
write that was outstanding when the host froze the guest then looks hundreds
of seconds old, and the first EXE$TIMEOUT run after the wake times it out. In
that case the device had already completed the write (all bytes transferred,
D2H status 50), and the trace shows the guest's interrupt routine
acknowledging that completion (PxIS read and cleared) before the time-out was
declared. From the time-out on, everything is as in section 5. With a write
loop running this failed in 5 host suspends out of 5.

In a later boot of the same host the TSC passed the check (clocksource tsc;
KVM did not report an unstable TSC). Then the guest's TSC did not jump at the
wake, EXE$GL_ABSTIM did not include the sleeps (OpenVMS still set its wall
clock correctly from KVM's clock), and a write that was outstanding across a
9-minute suspend completed 2.4 ms after the wake with no time-out: the node
carried on normally. In that same boot the held-write test of section 4 still
hung the disk after 25 seconds. So the host suspend is only a way of producing
the time-out on a host with an unstable TSC; the recovery defect is
independent of it.

## 8. Questions

1. Is the clear at SATL_SUPPORT line 47025 meant to drop bit 3 of that field,
   and which code is meant to set it again once the port reset has succeeded?
2. After COMRESET, QEMU's AHCI model presents a disk with task-file status 30
   (DRDY clear) and error 01 until the first command completes; the driver
   reads PxTFD twice at that point. Does the recovery depend on that value?
3. What is the time limit for a read or write on this port, and can it be
   changed? Is there a workaround meanwhile, such as a supported controller
   type whose driver does recover?
4. Is the same recovery path used on other configurations (VMware's AHCI
   controller, physical AHCI controllers)? We could not test VMware: our ESXi
   host is itself a KVM guest, and stalling its storage brought ESXi down
   before OpenVMS's limit was reached.
5. Do the other port drivers (SYS$PKMDRIVER, for example) recover from a
   command time-out through the same logic? And has this recovery been
   exercised with an MSCP-served disk, or in shared-disk cluster testing?

## 9. Evidence

- **Two system dumps,** made with SDA COPY, are in the repository's release
  [`dumps-2026-11`](https://github.com/EdwinThornber/openvms-pkddriver-ahci-hang/releases/tag/dumps-2026-11):
  the hang after a host suspend (15-NOV-2026 22:38) and the INCONSTATE crash
  from a data disk (18-NOV-2026 11:05).
- **Everything else is available on request:** SDA output, trap logs, throttle
  test logs, AHCI register traces, the 18/19-NOV tests, console logs, the other
  system dumps, and the VSI$SUPPORT.COM reports (to VSI only; they hold licence
  data). [MANIFEST.txt](MANIFEST.txt) lists every file.

Edwin Thornber, VSI OpenVMS Ambassador, E9.2-4 field test
