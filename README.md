# OpenVMS x86-64: SYS$PKDDRIVER never accepts an AHCI disk back after one command time-out

We have found a bug in SYS$PKDDRIVER, the AHCI (SATA) port driver, on OpenVMS x86-64 V9.2-3 and E9.2-4.

- **The hang:** if the host holds a write to an AHCI disk for about 26 s, the driver times out and resets the port. The disk then goes into mount verification and never comes out, even after the storage is back.
- **The crash:** cancelling that mount verification at IPC (`C <disk>`) crashes the system with an INCONSTATE bugcheck in SYS$DKDRIVER, the disk class driver.

This repository holds the full report, the step-by-step reproduction and evidence, and two system dumps, for VSI and for anyone who wants to reproduce it. The rest of our evidence (SDA output, write-watchpoint traces, host-side AHCI register traces, console logs and further dumps) is listed in [MANIFEST.txt](MANIFEST.txt) and available on request.

## Where we have seen it

| | Hang after a ~26 s stall | INCONSTATE on `IPC> C` |
|---|---|---|
| **E9.2-4** | yes: system disk and data disk; host suspend, QEMU write throttle, device-mapper suspend | yes, 4 of 4 (3 on the system disk, 1 on a data disk) |
| **V9.2-3 Community** | yes: port reset at 25.9 s | not tried |
| **Control: virtio-scsi** (SYS$VSPDRIVER, same SYS$DKDRIVER) | no: a 214 s stall and an 11 h stall did no harm | — |

Real events that can hold a write that long include a host suspend with a write in flight (how we first hit it), an LVM snapshot, multipath with `queue_if_no_path`, or a busy NAS.

## Documents

| File | What it is |
|---|---|
| [report.md](report.md) | The bug report: summary, impact, system, reproduction, the failure step by step, where the state is lost, questions for VSI |
| [step-by-step-reproduction.md](step-by-step-reproduction.md) | How to make it happen on the system disk with a QEMU write throttle, what should happen and what does |
| [step-by-step-evidence.md](step-by-step-evidence.md) | The same steps with log, trace and SDA extracts, down to the code that loses the state |
| [MANIFEST.txt](MANIFEST.txt) | What is here, and every further file and dump available on request |

## Our stack

- **Host:** Ubuntu 24.04.5 LTS, kernel 7.0.0-43-generic, on a Dell Precision 7810 (2 × Xeon E5-2690 v3). This is not one of VSI's tested KVM hosts (Oracle Linux 9.7, RHEL 9.8).
- **Hypervisor:** QEMU 8.2.2 (Ubuntu 1:8.2.2+ds-0ubuntu1.18), libvirt 10.0.0, OVMF 2024.02 (`OVMF_CODE_4M.fd`).
- **Guests:** machine type `pc-q35-noble` (q35), built-in ICH9 AHCI controller, CPU `host-passthrough`, 2 vCPUs, 8 GiB. Node X86924 runs E9.2-4; node X86923 runs V9.2-3 Community.
- **Disks:** `cache=writethrough`. The data disks are raw and fully allocated, each behind a device-mapper linear device so the host can freeze it with `dmsetup suspend`.

## Quick reproduction on a data disk (KVM)

> **⚠ WARNING: run this only on a test node, against a scratch data disk.**
>
> - The AHCI disk under test stays in mount verification until the node is crashed or rebooted. Any disk SYS$PKDDRIVER controls does the same if its storage stalls for ~26 s, the system disk included.
> - The whole guest freezes for as long as the storage is held.
> - `C <disk>` at IPC crashes the node with INCONSTATE.
> - Expect SHUTDOWN to hang on the stuck disk. Crash the node instead: Ctrl/P, Ctrl/P, `y`.
> - Don't try it under a nested hypervisor: stalling our nested ESXi host's disk gave ESXi a purple screen.

**1. Host, as root: create a disk you can freeze.**

```sh
img=/var/lib/libvirt/images/stall-ahci.img
qemu-img create -f raw -o preallocation=full $img 32G
loop=$(losetup --find --show --direct-io=on $img)
dmsetup create stall-ahci --table "0 $(blockdev --getsz $loop) linear $loop 0"
```

**2. Host, as root, guest shut down: attach it as SATA unit 1, which OpenVMS sees as DKA100.** Set `vm` to your guest's libvirt name.

```sh
vm=vms-test
virsh attach-disk $vm /dev/mapper/stall-ahci sdb --targetbus sata --driver qemu --subdriver raw --cache writethrough --config
virsh start $vm
```

**3. Guest, as SYSTEM: initialise and mount it.**

```
initialize/structure_level=5/system/cluster_size=16 DKA100: STALLTEST
mount/system DKA100: STALLTEST
create/directory DKA100:[TEST]
```

**4. Guest, as SYSTEM: keep a write in flight.** Save this as `SYS$LOGIN:WRITELOOP.COM`, then start it from an ssh session, so the console stays free for IPC.

```
$ set noon
$ disk = p1
$ loop:
$ copy sys$library:starlet.olb 'disk'[test]scratch.tmp;1
$ delete 'disk'[test]scratch.tmp;1
$ goto loop
```

```
@sys$login:writeloop DKA100:
```

**5. Host, as root: freeze the disk, wait about 40 s, release it.**

```sh
dmsetup suspend stall-ahci
```

About 26 s later the console shows `%SYSTEM-I-MOUNTVER, <node>$DKA100: is offline. Mount verification in progress.` and the guest freezes. Wait another 10 s, then:

```sh
dmsetup resume stall-ahci
```

- **Expected:** mount verification completes and the loop carries on.
- **What we see:** the guest unfreezes, but DKA100 stays in mount verification for ever, with error count 0. Other disks are unaffected.

**6. Guest console, optional: the crash.** Press Ctrl/P and type `C DKA100` at `IPC>`, then Ctrl/Z. The console shows `has aborted mount verification`, then at once `Bugcheck 0000019C INCONSTATE`. On E9.2-4 the PC is `SYS$DKDRIVER+80001E44`, with R0 = `0000032C` (SS$_DEVCMDERR).

**Without device-mapper:** a QEMU write throttle on any SATA disk does the same. Use this on E9.2-4, or 1024 on V9.2-3, and set it back to 0 to release:

```sh
virsh blkdeviotune <vm> <target> --write-bytes-sec 4096 --live
```

## System dumps

Two dumps, made with SDA COPY on the node, are in the release [`dumps-2026-11`](https://github.com/EdwinThornber/openvms-pkddriver-ahci-hang/releases/tag/dumps-2026-11), gzipped (about 86 MB each). The release also holds the SHA-256 sums of the compressed and uncompressed files.

| Dump | What it shows |
|---|---|
| `node2-SYSDUMP-20261115-2238-H5HANG-SDACOPY.DMP` | E9.2-4, forced crash (OPERCRASH) of a system disk hung in mount verification after a host suspend; IPC not used, so it shows the hung state itself |
| `node2-SYSDUMP-20261118-1105-T1-INCONSTATE-DKA100-SDACOPY.DMP` | E9.2-4, INCONSTATE after `IPC> C DKA100` on a hung AHCI data disk |

The files lose their RMS attributes on the way through Linux. Before you analyse one on OpenVMS, restore them:

```
set file/attributes=(rfm=fix,lrl=512,mrs=512,rat=none) node2-SYSDUMP-*-SDACOPY.DMP
analyze/crash node2-SYSDUMP-20261118-1105-T1-INCONSTATE-DKA100-SDACOPY.DMP
```

Eighteen more dumps from the investigation are listed in [MANIFEST.txt](MANIFEST.txt), available on request.

## Status

- **19 Nov 2026:** published, and reported on the VSI forum.
- Results from other hosts and controllers are welcome. We would especially like them from VSI's tested hosts, VMware's SATA controller and VirtualBox's ICH8 AHCI.

Edwin Thornber, VSI OpenVMS Ambassador
