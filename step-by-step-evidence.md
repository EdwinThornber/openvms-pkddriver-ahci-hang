# The evidence, step by step

**Companion to** `step-by-step-reproduction.md`. That document says what to do
and what you will see. This one follows the same steps and shows, for each,
the short pieces of our logs, traces and SDA output that prove what happened,
with the values that matter marked `<--`. It ends with how we traced the
failure to one longword in SYS$PKDDRIVER.

All values are from node X86924 (OpenVMS x86-64 E9.2-4) unless stated. The
file each extract comes from is named in brackets. Those files are not in this
repository; [MANIFEST.txt](MANIFEST.txt) lists them, and they are available on
request. Long lines in the extracts
are shortened with `...`; nothing else is changed.

**Two notations for the same code address.** SDA's `SHOW STACK` and our trap
logs write addresses as `SYS$PKDDRIVER+800xxxxx`. SDA's `MAP` command gives
image offsets that are 0x1B00 higher for this image, plus the module, offset
and listing line. Both are given where it matters; the `MAP` form is the one
to look up in the listings.

---

## The one field to watch

SYS$PKDDRIVER keeps a small block for each disk. You reach it from the disk's
UCB in three steps. In SDA (`ANALYZE/SYSTEM` on a running system, or
`ANALYZE/CRASH` on a dump):

```
show device dka0                       ! gives the UCB address
format <ucb>                           ! UCB$PS_DK_SCDT gives the SCDT address
format/type=scdt <scdt>                ! SCDT$L_PORT_SPECIFIC gives the block
examine <block>;20                     ! the longword at offset +10
```

On E9.2-4 the offsets are UCB$PS_DK_SCDT = UCB+2B0, SCDT$L_PORT_SPECIFIC =
SCDT+178 and UCB$L_DK_MEDOFL = UCB+3CC.

`EXAMINE` prints longwords from right to left: the rightmost longword is at
offset +00, so offset +10 is the fifth from the right.

**Healthy** (live SDA on our E9.2-4 ESXi guest, the same SYS$PKDDRIVER image;
16-NOV-2026 10:05) [`sda/esxi2-healthy-base2.txt`]:

```
SCDT$L_PORT_SPECIFIC     82506B40

00000000 04000000 00000000 0000000A 00000000 00000100 00000000 00000000   FFFFFFFF.82506B40
                           ^^^^^^^^
                           +10 = 0000000A   <-- healthy
```

**Hung** (the hang dump of 15-NOV-2026 22:38, `EXAMINE 820EEBC0;100`)
[`sda/node2-h5_sda1.txt`]:

```
00000000 01000000 00000000 00000002 00000000 00000100 00000000 00000000   FFFFFFFF.820EEBC0
                           ^^^^^^^^
                           +10 = 00000002   <-- hung: bit 3 (value 8) is gone
```

On node X86924 itself the field read 0000000A at the start of every test
run and after every boot we checked (step 6 below), and 00000002 in every
hang. In one boot we compared the healthy system
with the hung one 20 minutes later, field by field: apart from running
statistics, this longword was the only difference in the SCDT, the STDT and
this block.

---

## Step 1: the write loop keeps a write in flight

Our test script reads the guest's disk counters from the host before it
starts [`stalltests/stalltest-E924-run2-4096.log`]:

```
06:21:05.833 loop check: 815 reads, 815 writes in 5 s
```

That is about 160 writes a second. The same script also reads two values
from the guest's memory at every sample: UCB$L_DK_MEDOFL ("medofl") and the
port driver's field at +10. Both are healthy at the start:

```
06:21:00.763 baseline: state running; medofl 0x00000000; port per-disk field 0x0000000a; ...
                                             ^^^^^^^^^^                      ^^^^^^^^^^
                                             no "medium offline" yet         healthy value
```

---

## Step 2: one write is held for 32 seconds

```
06:21:05.860 throttle write-bytes-sec=4096: ok
06:21:16.375 phase 1 (+10 s): ... medofl 0x00000000; port per-disk field 0x0000000a; ... writes 637190
06:21:26.892 phase 1 (+20 s): ... medofl 0x00000000; port per-disk field 0x0000000a; ... writes 637190
                                                                                              ^^^^^^
                                             the write counter has stopped: one write is held
```

The host trace of the guest's AHCI register accesses confirms it
[`traces/summary-E924-run2-4096.txt`]:

```
   issued 06:21:05.866  outstanding  31.907 s          <-- the held write
first reset-type access (P0CMD) at 06:21:32.386
command outstanding at that moment was issued at 06:21:05.866: 26.519 s before the reset
                                                               ^^^^^^^^
                                                               OpenVMS's time limit here
```

---

## Step 3: the time-out, two "medium offline" verdicts, and the port reset

**On the console** at 06:21:32:

```
%SYSTEM-I-MOUNTVER, X86924$DKA0: is offline.  Mount verification in progress.
```

**In the test log**, 27 seconds after the limit was set:

```
06:21:33.048 CHANGE 27 s into phase 1
06:21:37.915 end of phase 1: ... medofl 0x00000002; port per-disk field 0x00000002; ...
                                       ^^^^^^^^^^                      ^^^^^^^^^^
                       two "medium offline" completions          bit 3 lost (was 0x0a)
```

**What the time-out path looks like.** Stack at the moment the port driver
hands the held write back to the class driver (identical in six captures;
module, offset and line from SDA `MAP`):

```
EXE$TIMEOUT_C+0037E                         the once-a-second time-out scan
  PKDDRIVER + 00003B3D, line 92741
    PKDDRIVER + 00005394, line 92830        resumes the class driver's thread
      DKDRIVER_C + 000047C7, line 77598
        DKDRIVER_C + 0000606C, line 76652
          DKDRIVER_X86 + 000021B7, line 20309
            DKDRIVER_X86 + 0000DCE1, line 27579   <-- UCB$L_DK_MEDOFL incremented,
                                                      write completed "medium offline"
```

**Mount verification's first PACKACK is refused at once.** In the early dump
(taken 1.8 s after the two verdicts, 15-NOV-2026 21:57) the PACKACK request
has already finished [`sda/node2-h4_sda1.txt`, `FORMAT 824418C0`]:

```
FFFFFFFF.824418CC   IRP$L_PID     8B1114B0   EXE$ILLIOFUNC_C+00180   <-- issued by mount verification
FFFFFFFF.82441AB8   IRP$L_IOST1   000001A4                            <-- SS$_MEDOFL
FFFFFFFF.82441AC8   IRP$L_FUNC    00000808                            <-- IO$_PACKACK
```

The PACKACK's INQUIRY (CDB `12 00 00 00 FF 00`) was refused by the port driver
without any ATA command reaching the disk. It ran inside the same call from
EXE$TIMEOUT, before the port reset.

**Then the port reset** [`traces/summary-E924-run2-4096.txt`; t = 0 at the
first reset access]:

```
       +0.000 ms  read P0CMD 0xc017
       +0.029 ms  write P0CMD 0xc016          <-- port stopped (ST cleared)
       +0.346 ms  write P0SCTL 0x301          <-- COMRESET asserted
      +22.250 ms  write P0SCTL 0x300          <-- COMRESET released
    +5387.763 ms  MSI vec 207                 <-- (see note)
    +5391.216 ms  read P0SSTS 0x113           <-- link up, device present
    +5391.652 ms  read P0TFD 0x130            <-- status 30 (DRDY clear), error 01
    +5391.703 ms  read P0TFD 0x130                read twice
    +5391.818 ms  write P0CMD 0x4017          <-- port started again
    +5392.644 ms  write P0CI 0x1              <-- first command after the reset
    +5392.809 ms  read P0IS 0x3                   (PIO set-up + D2H: an IDENTIFY-type command)
    +5394.441 ms  write P0CI 0x1              <-- second command
    +6395.104 ms  write P0CI 0x1              <-- from here on, one command a second
```

Note: QEMU completed the reset only once the held write had drained, at the
moment we removed the limit (+5.39 s). The reset itself then went normally.

**V9.2-3 does the same, step for step** [`traces/summary-V923-runs1-2.txt`]:
port stopped 25.87 s after its held write was issued, COMRESET, P0TFD read
twice as 0x130, port started.

---

## Step 4: the limit is gone, the disk is fine, and OpenVMS never uses it again

**No I/O at all** [`stalltests/stalltest-E924-run2-4096.log`]:

```
06:21:37.939 throttle write-bytes-sec=0: ok
06:21:48.122 phase 3, throttle off (+10 s): ... medofl 0x00000002; port per-disk field 0x00000002; reads 1190310 writes 637191
06:22:18.599 phase 3, throttle off (+40 s): ... medofl 0x00000002; port per-disk field 0x00000002; reads 1190310 writes 637191
                                                                                                  ^^^^^^^        ^^^^^^
                                                                         not one read or write in 40 seconds
```

**The disk answers every question it is asked.** The only command OpenVMS
still sends is CHECK POWER MODE, once a second, and the disk answers "ready".
From the AHCI command slot read during a hang (V9.2-3, 08:07:47)
[`traces/node1-hang-ahci.txt`]:

```
command FIS: 27 80 e5 00 ...          <-- ATA command E5 = CHECK POWER MODE
D2H FIS:     34 40 50 00 ... ff ...   <-- status 50 = ready, no error; count FF = active
```

On 14-NOV-2026 a trace of one hang showed 126 such commands at 1.000 to
1.002 s intervals and nothing else: no read, no write, no reset.

**What mount verification gets each second.** We took this from a memory
image of the hung guest, not from an SDA dump: each second mount verification
issues a one-block read of LBN 31 hex (function 080C), SYS$DKDRIVER builds
READ(10) `28 00 00 00 00 31 00 00 01`, and the request comes back with

```
IRP$L_IOST1 = 0000032C        <-- %SYSTEM-F-DEVCMDERR, device command error
```

without the read ever being sent to the disk (no read appears in the host
trace).

**SDA on the hang dump** (15-NOV-2026 22:38) [`sda/node2-h5_sda1.txt`]:

`SHOW DEVICE DKA0`:
```
X86924$DKA0                ATA QEMU HARDDISK   UCB: 821082C0
Device status:   08005910 online,bsy,valid,unload,mntverip,exfunc_supp    <-- stuck in mount verification
DK Class Busy    00000009 nocmdq,mntverip
Error count              0                                               <-- nothing recorded
```
followed by 13 requests waiting in the I/O request queue.

`CLUE SCSI/PORT`:
```
PKA0:                                       Driver         SYS$PKDDRIVER
SPDT Address                820F80C0        Port Type      AHCI SATA controller
Target inited Bus Resets           0        Number of Events                   0   <-- the reset is
Retry Attempts                     0        Curr I/Os on all Ports             0       not counted
```

`CLUE SCSI/CONNECTION`:
```
Connection State            open
Total Outstanding I/Os             0        Number of Commands sent            0
Count of Controller Errors         0        Count of Retries                   0
```

So OpenVMS has recorded nothing of the incident: error count 0, no resets, no
events, no retries. The error log is empty too, because ERRFMT cannot write
to the hung system disk.

`EXAMINE 820EEBC0;100` (the port driver's block): +10 = 00000002, as shown at
the top.

---

## Step 5: `IPC> C DKA0` and the INCONSTATE bugcheck

**Console** (15-NOV-2026, the third occurrence; one CPU active):

```
IPC> C DKA0                    <-- Ctrl/P at 10:22:39, C DKA0 typed at 10:22:46, nothing more
IPC> ^Z                        <-- IPC left by itself at 10:23:31, about 45 s later
%SYSTEM-I-MOUNTVER, X86924$DKA0: has aborted mount verification.
VSI Dump Kernel - Handles Shutdown/Reboot/Crash Dumps
** Bugcheck code = 0000019C: INCONSTATE, Inconsistent I/O data base
** Crash Time:            15-NOV-2026 10:23:31.56       <-- the same second as the abort
** Current Process:       "CPT$performance"
```

**Control, four minutes earlier on the same hang:**

```
IPC> ^Z                        <-- Ctrl/P at 10:18:15, no input; IPC left by itself at 10:19:00
```

There was no crash and no output of any kind; the system stayed hung.

**`CLUE CRASH` on the first occurrence** (14-NOV-2026 04:17)
[`sda/node2-inc_sda1.txt`; the dump itself is available on request, see MANIFEST.txt]:

```
Bugcheck Type:     INCONSTATE, Inconsistent I/O data base
Current Process:   ERRFMT
Failing PC:        FFFF8300.06E89F44    SYS$DKDRIVER+80001E44   (DKDRIVER_X86 + 00001E44 / line 20250)
Module:            SYS$DKDRIVER    (Link Date/Time:  4-SEP-2026 11:49:13.12)
RAX = 00000000.00009700   RDI = 00000000.00100002   RSI = 00000000.0000019C
RCX = FFFF8300.06E9CB80   R8  = FFFFFFFF.82108830   R13 = 00000000.82108700
                                ^^^^^^^^^^^^^^^^^^         ^^^^^^^^^^^^^^^^
                                UCB+130                    DKA0's UCB in that boot
```

| | 14-NOV 04:17 | 14-NOV 21:53 | 15-NOV 10:23 |
|---|---|---|---|
| Active CPUs | 2 | 2 | 1 |
| Current process | ERRFMT | JOB_CONTROL | CPT$performance |
| Failing PC | SYS$DKDRIVER+80001E44 | the same | the same |
| R13 | DKA0's UCB | DKA0's UCB | DKA0's UCB |
| RAX, RDI, RSI | 9700, 100002, 19C | the same | the same |

The failing PC and registers are the same every time; the process is whatever
happened to be current. These three hangs came from the host-suspend route;
the hung state they show is the one in step 4.

---

## Step 6: following the field to the code that loses it

The field at +10 is 0000000A when the system is healthy and 00000002 for ever
after a time-out. To find which code writes it, we put a hardware write
watchpoint on its address, through QEMU's gdb stub, across a boot and across
a time-out. The guest stops on every write, and we log the instruction and
its callers. Any write watchpoint would do the same.

**Across a boot** [`traps/boottrap.log`]. The first 79 stops are other users
of that memory before the port driver owns it. Then:

```
stop  81  fields 0x00000008 ... RIP SYS$PKDDRIVER+80027BE9  callers: SYS$PKDDRIVER+80014917, SYS$PKDDRIVER+80019345, ...
                 ^^^^^^^^^^           set-up code writes 8: bit 3 set
stop  83  fields 0x0000000a ... RIP SYS$PKDDRIVER+80026672  callers: SYS$PKDDRIVER+80026AB2, SYS$PKDDRIVER+80026112, ...
                 ^^^^^^^^^^           identify-time code adds 2: 8 -> A
...
06:50:05 a field reads 0xA and no stop for 20 s: done
```

**Across a time-out** [`traps/stop-01.txt`, `stop-03.txt`]:

```
stop 01 captured 06:41:19                      <-- 27 s after the limit was set
UCB medofl 00000002                                both verdicts already given
AHCI P0: ... CMD 0000C017 ... SCTL 00000300        port not yet reset
CPU 0: RIP FFFF830006E76924 SYS$PKDDRIVER+80027C24   <-- the field is cleared: A -> 0
   FFFFFFFF8E243E38  FFFF830006E67309  SYS$PKDDRIVER+80018609   (caller)

stop 03 captured 06:42:20                      <-- about 1 s after the port reset
CPU 0: RIP FFFF830006E75372 SYS$PKDDRIVER+80026672   <-- identify-time code adds 2: 0 -> 2
   FFFFFFFF8E297E18  FFFF830006E757B2  SYS$PKDDRIVER+80026AB2
```

Nothing writes the field again: it stays 00000002, and from then on every read
and write is refused.

**The same places in the listings** (`MAP <address>` on the running system)
[`sda/node2-map1.txt`, `node2-map3.txt`]:

| When | Value | Trap address | MAP: module + offset, listing line |
|---|---|---|---|
| Boot, disk configured | = 8 | SYS$PKDDRIVER+80027BE9 | SATL_SUPPORT + 000003C9, line 46999 (routine at SATL_SUPPORT + 00000360, line 46978) |
| Boot, disk identified | 8 -> A | SYS$PKDDRIVER+80026672 | SATL + 00000812, line 40540 (routine at SATL + 00000590, line 40488) |
| Command time-out | A -> 0 | SYS$PKDDRIVER+80027C24 | **SATL_SUPPORT + 00000404, line 47025** (routine at SATL_SUPPORT + 000003E0, line 47018) |
| 1 s after the port reset | 0 -> 2 | SYS$PKDDRIVER+80026672 | SATL + 00000812, line 40540 (as at boot) |
| Afterwards | none | | |

Callers, as `MAP` gives them:

- the time-out clear is called from PKDDRIVER + 00006101, line 92952;
- the set-up routine is called from PKDDRIVER + 0000240F, line 89682, itself
  called from PKDDRIVER + 00006E3D, line 86787.

---

## Conclusion

Bit 3 of the field is set only by the set-up code (SATL_SUPPORT line 46999),
when the disk is first configured. At a command time-out the routine that
follows it in the same module clears the whole field (line 47025). The
recovery after the port reset runs only the identify-time code (SATL line
40540), which restores bit 1 and leaves bit 3 clear. From that moment the
driver refuses every read and write with DEVCMDERR without sending it, while
its readiness check (CHECK POWER MODE) still succeeds. Mount verification
therefore retries for ever, and cancelling it ends in the INCONSTATE bugcheck
of step 5.

Whether the time-out path should not clear bit 3, or the recovery should set
it again, is for VSI to judge; the four places are exact.
