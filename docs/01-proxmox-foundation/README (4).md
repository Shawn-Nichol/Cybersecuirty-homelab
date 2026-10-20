# Proxmox VE Node: Deployment, Fault Investigation & Stabilization

> **Status:** Node stable on kernel `7.0.14-24-pve`. Turbo Boost re-enabled, risk accepted for a non-production lab. Fault is confirmed turbo-dependent; the root component was not isolated.

## Overview

I deployed a single-node Proxmox VE hypervisor as the foundation for a cybersecurity home lab. During setup I fixed repository authentication errors and then investigated a **reproducible hard reset** during a kernel package install. The investigation used a one-variable-at-a-time method, produced an A/B result, and ended with a documented compensating control and a risk decision.

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

<!-- Uncomment after uploading images/04b-unpack-crash.png
![Crash during unpack](images/04b-unpack-crash.png)
-->
<!-- Uncomment after uploading images/06-no-turbo-pass.png
![Turbo-off pass](images/06-no-turbo-pass.png)
-->
<!-- Uncomment after uploading images/14b-stress-cpu.png
![Stress test](images/14b-stress-cpu.png)
-->

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

<!-- Uncomment after uploading images/07-staged-install.png
![Staged install](images/07-staged-install.png)
-->

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

| Item | Detail |
|---|---|
| Decision | Re-enabled turbo and accepted the risk for a non-production lab. The compensating control was removed at the owner's discretion. |
| Residual risk | Resets are possible under burst load (package installs, large decompression, VM boot storms). No confirmed fix. |
| Mitigations | Disable turbo before `apt upgrade` or kernel work and re-enable afterward; keep `/etc/pve` backups off the node; keep kernel `-24` pinned; Btrfs RAID1 protects against single-drive loss (not a backup). |
| Exit criteria | Stable with turbo on at Intel-spec PL1/PL2 power limits. |
| If it recurs | BIOS power limits, PSU swap, `memtest86+` and matched-pair RAM test, then Intel's warranty process. |

**Notes on state:** the `proxmox-auto-removal` hook pruned the `7.0.2-6-pve` fallback kernel during the install, so the bootable set is `-24` and `-25`. Only `proxmox-kernel-7.0` remains under `apt-mark hold`.

## Lessons Learned

- A hold is a workaround, not a fix. It also blocks security patches, so it needs a review date.
- Change one variable at a time and write each result down, including negative ones.
- Split a failing operation into stages (download, unpack, configure) to find where it breaks.
- Runtime-only settings (`/sys`) vanish on reboot, so verify persistence before the next test.
- Kernel post-install hooks can prune fallback kernels. Check bootable entries after every install.
- Run crash tests over SSH with `journalctl -f` open on a second machine, because an instant reset leaves nothing in the local log.
- Pasting multi-line blocks into a web console can inject escape sequences. Verify that each command actually ran.

## Next Phase

Network segmentation: VLAN-aware bridge with management, attacker, victim and SIEM segments, plus an OPNsense/pfSense firewall VM.
