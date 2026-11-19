## 1a. The computer in front of you

| Figure | This computer | Command it came from |
|---|---|---|
| CPU model | 13th Gen Intel(R) Core(TM) i5-13420H | `Get-CimInstance Win32_Processor` |
| Cores / logical processors | 8 cores / 12 logical processors | `Get-CimInstance Win32_Processor` |
| Hardware virtualization: supported? enabled in firmware? | Supported (Intel VT-x), shows `False` in PowerShell (either disabled in BIOS or hidden because Windows Hyper-V is active) | `Get-CimInstance Win32_Processor` (`VirtualizationFirmwareEnabled`) |
| Memory: total / available now | 15.7 GB (16,457,592 KB) total / 4.8 GB (5,044,572 KB) available | `Get-CimInstance Win32_OperatingSystem` |
| System disk: size, free space, SSD or HDD | 512 GB (476.9 GiB) physical / 475.8 GiB C: drive, 20.7 GiB (22.26 GB) free, SSD (`KINGSTON OM8PGP4512Q-AA`) | `Get-PhysicalDisk` and `Get-Volume -DriveLetter C` |
| Partition style of the system disk | GPT | `Get-Disk` |
| OS and version | Windows 11 Pro, Version 10.0.26200 (Build 26200) | `Get-CimInstance Win32_OperatingSystem` |

## 1b. A machine you have never seen: a thin-and-light laptop

### 1. Both guests?
No, it cannot run both guests from Labs 1-2 at the same time. Even if we give Ubuntu Server 1 vCPU, 2 GB RAM, 20 GB disk and Windows 10/11 2 vCPUs, 4 GB RAM, 40 GB disk, the laptop runs out of RAM and storage:

- **Host assumption:** Windows 11 Home needs around 1 core (2 threads), 3.5-4 GB RAM, and at least 15 GB free disk space for pagefile and updates.
- **CPU:** 4 threads total - 3 vCPUs (1 + 2) = 1 thread left for the host. At 1.1 GHz base clock it will lag badly.
- **Memory:** 8 GB total - 4 GB host - 6 GB guests (2 + 4) = -2 GB. It will swap constantly to slow eMMC storage.
- **Disk:** 31 GB free - 60 GB guests (20 + 40) = -29 GB. Not enough space even if disks are dynamically allocated.

### 2. Upgrade first
Nothing can actually be upgraded on this laptop because both the LPDDR4X RAM and the 128 GB eMMC storage are soldered to the motherboard, and there are no extra slots.

Instead, I would switch Windows 11 out of S mode first (through Microsoft Store / Settings) so it allows installing VirtualBox or VMware, and then run only one lightweight VM at a time (for example, Ubuntu Server with 1 vCPU, 1 GB RAM, and a 10 GB dynamic disk).
