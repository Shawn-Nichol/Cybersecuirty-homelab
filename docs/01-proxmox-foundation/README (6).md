# Proxmox VE Node: Deployment, Fault Investigation & Stabilization

> **Status:** The node is stable on kernel `7.0.14-24-pve`. I re-enabled Turbo Boost and I'm running with the known risk, since this is a non-production lab. I confirmed the fault depends on turbo, but I did not isolate the failing component.

## Overview

I deployed a single-node Proxmox VE hypervisor as the foundation for a cybersecurity home lab. During setup I fixed repository authentication errors and then investigated a **reproducible hard reset** during a kernel package install. The investigation used a one-variable-at-a-time method, produced an A/B result, and ended with a compensating control and a documented decision on the remaining risk.

**Skills demonstrated:** Linux/Debian package management (`apt`, `dpkg`), hardware fault isolation, log analysis (`journalctl`, `dmesg`, SMART), Btrfs RAID1, UEFI boot management (`proxmox-boot-tool`), systemd, change control and risk acceptance.

## Environment

| Item | Value |
|---|---|
| CPU | Intel Core i7-14700 (8P + 12E cores) |
| Motherboard / BIOS | ASUS PRIME B760M-A AX6, AMI BIOS 1840 (09/18/2026) |
| CPU microcode | `0x133` (`intel-microcode 3.20251111.1~deb13u1`) |
| RAM | 64 GB DDR5 (4 x 16 GB, two different Corsair kits), 4200 MT/s, non-ECC |
| Storage | 2 x 1 TB NVMe, Btrfs RAID1 (data, metadata, system) |
| Boot | UEFI, two ESPs managed by `proxmox-boot-tool`, Secure Boot disabled |
| Base OS | Debian 13 (trixie), Proxmox VE |
| Kernels | `7.0.14-24-pve` (running, pinned), `7.0.14-25-pve` (installed, not booted) |
| PSU | `<make / wattage>` |
| Network | `<vmbr0 bridge, subnet, upstream>` |

## Problem 1: `401 Unauthorized` on `apt update`

**Symptom:** `apt update` failed with authentication errors.
**Cause:** The enterprise repositories (`pve-enterprise`, `ceph-squid`) require a paid subscription.
**Fix:** In the web UI under *Node > Updates > Repositories*, I disabled the enterprise repositories and confirmed `pve-no-subscription` was active.
**Verification:** `apt update` completed cleanly.

<!-- Uncomment after uploading images/02-repos.png
![Repositories tab](images/02-repos.png)
-->

## Problem 2: Hard reset during kernel install

### Symptom

Installing `proxmox-kernel-7.0.14-25-pve-signed` caused an instant power-cycle at about 95%. There was no kernel panic and nothing in the logs. The interrupted install left `dpkg` half-configured, and I applied `apt-mark hold` to stop further attempts.

### Investigation

| Hypothesis | Test | Result | Conclusion |
|---|---|---|---|
| Corrupt download | APT signature/hash verification on `apt-get download` | Passed | Ruled out |
| Postinst / initramfs / boot hooks | Staged `dpkg --unpack` then `--configure` | Reset during `--unpack`, before any hook ran | Ruled out |
| Disk write path | `dpkg-deb --fsys-tarfile \| wc -c` (decompress only, no writes) | Reset | Unlikely (reads still exercised) |
| NVMe media fault | `smartctl` on both drives, `btrfs device stats` | 0 media errors, 0 filesystem errors, 32-35 C | Ruled out |
| Software crash / panic | `journalctl -b -1`, `dmesg` | No MCE, panic or NVMe errors; log ends mid-routine | Consistent with an instantaneous hardware or firmware reset |
| Reset history | `journalctl --list-boots`, SMART `Unsafe Shutdowns` | Repeated short boots; 46 and 33 unsafe shutdowns | Corroborates reproducibility |
| Missing CPU mitigation microcode | `/proc/cpuinfo`, `dmesg`, package version | `0x133`, package 3.20251111 | Mitigation applied; does not exclude prior degradation |
| **Boost-dependent instability** | `no_turbo=1`, decompress reproducer | **Pass** | Supports hypothesis |
| Install under reduced power | Staged `--unpack` / `--configure` with turbo off | Completed; `dpkg --audit` clean | Supports hypothesis |
| **Turbo toggle control** | `no_turbo=0`, same decompress reproducer | **Reset** | Supports hypothesis |
| Sustained CPU load, turbo off | `stress-ng --cpu 0`, 10 min | 28/28 passed | Stable (short duration) |
| Mixed CPU/RAM/IO load, turbo off | `stress-ng --cpu --vm --hdd`, 10 min | 32/32 passed, no logged errors | Stable (30 GB RAM, light I/O) |
| Power limits, PSU swap, memtest86+ | Not performed | n/a | Open |

### A/B result

| Turbo state | Tests | Outcome |
|---|---|---|
| On | Original install, `--unpack`, decompress-only, control test | Reset every time (4+) |
| Off | Decompress-only, staged install, two 10-minute stress runs | Pass (4) |

> **Note on terminal output:** blocks below are transcribed from the real session. The green coloring is a Markdown `diff` rendering trick, so each line carries a leading `+` that was not in the original output.

```diff
+ root@pve:~# dpkg --unpack ./proxmox-kernel*.deb
+ (Reading database ... 60724 files and directories currently installed.)
+ Preparing to unpack .../proxmox-kernel-7.0.14-25-pve-signed_7.0.14-25_amd64.deb ...
+ Unpacking proxmox-kernel-7.0.14-25-pve-signed (7.0.14-25) over (7.0.14-25) ..
```
*Transcribed terminal output. The output stops here: the system reset during `dpkg --unpack`, before any post-install hook ran.*
```diff
+ root@pve:~# echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
+ root@pve:~# cat /sys/devices/system/cpu/intel_pstate/no_turbo
+ 1
+ root@pve:~# cd /root && dpkg-deb --fsys-tarfile proxmox-kernel-7.0.14-25-pve-signed_7.0.14-25_amd64.deb | wc -c
+ 1033103360
```
*Transcribed terminal output. With turbo disabled, the decompress-only reproducer that previously reset the system completed.*
```diff
+ root@pve:~# stress-ng --cpu 0 --timeout 10m --metrics-brief
+ stress-ng: info:  [2383] setting to a 10 mins run per stressor
+ stress-ng: info:  [2383] dispatching hogs: 28 cpu
+ stress-ng: metrc: [2383] stressor       bogo ops real time  usr time  sys time   bogo ops/s     bogo ops/s
+ stress-ng: metrc: [2383]                           (secs)    (secs)    (secs)   (real time) (usr+sys time)
+ stress-ng: metrc: [2383] cpu            13225050    600.00  16759.94      0.20     22041.74         789.08
+ stress-ng: info:  [2383] skipped: 0
+ stress-ng: info:  [2383] passed: 28: cpu (28)
+ stress-ng: info:  [2383] failed: 0
+ stress-ng: info:  [2383] metrics untrustworthy: 0
+ stress-ng: info:  [2383] successful run completed in 10 mins
```
*Transcribed `stress-ng` summary: 28 CPU workers for 10 minutes with turbo disabled.*

### Root cause (suspected)

Instability under boost, with the failing component **not isolated**. Candidates remain CPU boost-voltage instability or degradation (the i7-14700 belongs to the 13th/14th-gen family affected by Intel's Vmin Shift issue), VRM or power-limit configuration, and PSU headroom. A secondary candidate is the mixed four-DIMM DDR5 configuration. The sample size is small and I did not control for every confounder.

### Recovery

```bash
# Run with turbo disabled for the duration
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
sync
dpkg --unpack ./proxmox-kernel-7.0.14-25-pve-signed_7.0.14-25_amd64.deb
dpkg --configure proxmox-kernel-7.0.14-25-pve-signed
dpkg --audit                                   # no output = clean
proxmox-boot-tool kernel pin 7.0.14-24-pve     # keep booting the known-good kernel
```

```diff
+ root@pve:~# cat /sys/devices/system/cpu/intel_pstate/no_turbo
+ 1
+ root@pve:~# dpkg --unpack /root/proxmox-kernel-7.0.14-25-pve-signed_7.0.14-25_amd64.deb
+ Unpacking proxmox-kernel-7.0.14-25-pve-signed (7.0.14-25) over (7.0.14-25) ...
+ root@pve:~# dpkg --configure proxmox-kernel-7.0.14-25-pve-signed
+ Setting up proxmox-kernel-7.0.14-25-pve-signed (7.0.14-25) ...
+ update-initramfs: Generating /boot/initrd.img-7.0.14-25-pve
+ Running hook script 'zz-proxmox-boot'..
+ [... ESP copy output trimmed ...]
+ Generating grub configuration file ...
+ done
+ root@pve:~# proxmox-boot-tool kernel pin 7.0.14-24-pve
+ Set kernel '7.0.14-24-pve' in /etc/kernel/proxmox-boot-pin.
```
*Transcribed terminal output, trimmed where marked. The staged install ran to completion with turbo disabled.*

### Compensating control (later retired)

While investigating, I disabled turbo persistently with a systemd oneshot unit, verified it survived a reboot, and kept the unit on disk (disabled) so it can be re-enabled in one command.

```ini
[Unit]
Description=Disable Intel Turbo Boost (stability workaround)
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo'
[Install]
WantedBy=multi-user.target
```

## Decision and Residual Risk

After the stress tests passed with turbo off, I chose to re-enable turbo and move on to building the lab rather than keep chasing the root cause. This is a non-production lab, and the remaining tests (BIOS power limits, PSU swap, `memtest86+`) would take time without blocking the work I want to do next. I'm documenting the trade-off so it's clear this was a deliberate call, not an oversight.

| Item | Detail |
|---|---|
| Decision | Turbo re-enabled. I removed the compensating control and accepted the risk. |
| Residual risk | Resets are possible under burst load (package installs, large decompression, VM boot storms). I don't have a confirmed fix. |
| Mitigations | I disable turbo before `apt upgrade` or kernel work and turn it back on afterward. I keep `/etc/pve` backups off the node and keep kernel `-24` pinned. Btrfs RAID1 covers a single-drive failure, but it is not a backup. |
| Exit criteria | Stable with turbo on at Intel-spec PL1/PL2 power limits. |
| If it recurs | Check BIOS power limits, swap the PSU, run `memtest86+` and a matched-pair RAM test, then look at Intel's warranty process. |

**Notes on state:** the `proxmox-auto-removal` hook pruned the `7.0.2-6-pve` fallback kernel during the install, so my bootable kernels are now `-24` and `-25`. Only `proxmox-kernel-7.0` is still under `apt-mark hold`.

## Lessons Learned

- A hold is a workaround, not a fix. It also blocks security patches, so it needs a review date.
- Change one variable at a time and write each result down, including negative ones.
- Split a failing operation into stages (download, unpack, configure) to find where it breaks.
- Runtime-only settings (`/sys`) vanish on reboot, so verify persistence before the next test.
- Kernel post-install hooks can prune fallback kernels. Check bootable entries after every install.
- Run crash tests over SSH with `journalctl -f` open on a second machine, because an instant reset leaves nothing in the local log.
- Pasting multi-line blocks into a web console can inject escape sequences. Verify that each command actually ran.
