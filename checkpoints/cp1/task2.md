## Allocation

Right now my C: drive only has 20.73 GiB (22.26 GB) free, so it cannot fit two full-size VMs at the same time without freeing up ~15-20 GB first or using a stripped Windows 11 image. Also need to verify VT-x in BIOS or `systeminfo` since `VirtualizationFirmwareEnabled` returned `False`.

Closest working setup for current free space:

| Resource | Guest 1: Ubuntu Server | Guest 2: Windows 11 (minimal) |
|---|---|---|
| vCPUs | 2 vCPUs | 2 vCPUs |
| Memory | 1.5 GB (1536 MB) | 4 GB (4096 MB) |
| Disk size and type | 12 GB, dynamically allocated (~3.5 GB actual) | 32 GB, dynamically allocated (~12.5 GB actual with CompactOS, or 64 GB if I free ~15 GB on C: first) |
| Network | Adapter 1: NAT, Adapter 2: Host-Only | Adapter 1: NAT |

## Arithmetic

- **Host needs:** ~4 logical processors, ~5.5 GB RAM (OS + hypervisor), and at least 5-8 GB free disk space so Windows stays usable.
- **CPU:** 12 threads - 2 - 2 = 8 threads left for the host (more than enough).
- **Memory:** 15.7 GB total - 1.5 GB - 4 GB = 10.2 GB left for the host. Right now only 4.81 GB is free because of open apps, so I would close browser/IDE tabs first to free up ~2-3 GB.
- **Disk:**
  - Standard sizes (20 GB + 64 GB = 84 GB) do not fit on 20.73 GB free (-63.27 GB).
  - Normal dynamic installs (~4 GB Ubuntu + ~22 GB Win11 = 26 GB) still exceed 20.73 GB free by ~5.3 GB.
  - Trimmed setup (~3.5 GB Ubuntu + ~12.5 GB compact Win11 = 16 GB) leaves ~4.7 GB free on C:.

## Justification

- **What this setup gives up:** To fit into 20.7 GB free space without deleting host files, Windows 11 has to be installed below Microsoft's 64 GB minimum (using CompactOS and no hibernation/snapshots), leaving little room for Windows updates.
- **Ubuntu network (NAT + Host-Only vs Bridged):** Adapter 1 (NAT) gives Ubuntu internet access for `apt` updates. Adapter 2 (Host-Only) gives it a static local IP (`192.168.56.x`) so I can SSH from Windows anytime. Bridged mode often fails on university/office Wi-Fi because of MAC filtering or 802.1X.
- **Dynamic disks vs Fixed:** Fixed disks (12 + 32 = 44 GB) would fail immediately since I only have 20.7 GB free. Dynamic disks only use the space actually written.
- **Windows 11 RAM/CPU:** Kept at 2 vCPUs and 4 GB RAM because going below Microsoft's minimum makes Windows 11 almost unusable.

## Verification plan

1. **Check virtualization on host (PowerShell):**
   - Command: `systeminfo | findstr /I "Hyper-V"`
   - Expected: shows hypervisor detected or `Yes` for virtualization requirements.
2. **Check Ubuntu resources and internet (Ubuntu terminal):**
   - Command: `nproc && free -h && df -h / && ping -c 3 1.1.1.1`
   - Expected: `2` CPUs, `~1.4Gi` RAM, `~12G` disk, `0% packet loss`.
3. **Check SSH from host to Ubuntu (Host PowerShell):**
   - Command: `ssh student@192.168.56.101`
   - Expected: logs into `student@ubuntu-srv:~$` prompt.
4. **Check Windows 11 guest specs (Guest PowerShell):**
   - Command: `(Get-CimInstance Win32_Processor).NumberOfLogicalProcessors; [math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 1)`
   - Expected: `2` and `4`.