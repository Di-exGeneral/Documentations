# Swap Configuration Fix — Kali Linux
 
**System:** `weirdo_iv@geek` | Kali Linux | 7.1GB RAM | NVMe SSD
 
---
 
## Problem
 
The Linux kernel's OOM (Out of Memory) killer was terminating Google Chrome with the following error:
 
```
The background service app-google\x2dchrome@<uuid>.service has been terminated
by the Linux kernel because the system is low on memory.
```
 
### Root Cause
 
The system had **zero swap space configured**. When RAM usage spiked temporarily (primarily due to Chrome's multi-process architecture), the kernel had no fallback. Instead of gracefully offloading memory, it immediately killed the highest-scoring process — Chrome.
 
Additionally, the default `vm.swappiness` value of `60` meant the kernel was configured to start swapping aggressively even when RAM was not critically low — a poor default for a system that had no swap to begin with.
 
### System State Before Fix
 
| Metric | Value |
|---|---|
| Total RAM | 7.1GB |
| Swap | 0B |
| vm.swappiness | 60 |
| Disk free (/) | 17GB |
 
Chrome was running 8+ separate processes consuming roughly 2.5GB of RAM at idle.
 
---
 
## Fix
 
### Step 1 — Create a 4GB swapfile
 
```bash
sudo fallocate -l 4G /swapfile
```
 
**Why:** `fallocate` instantly allocates a contiguous block of disk space. 4GB was chosen as a safe balance between providing adequate overflow capacity and preserving the remaining 17GB of free disk space.
 
---
 
### Step 2 — Restrict permissions
 
```bash
sudo chmod 600 /swapfile
```
 
**Why:** Swap files must only be accessible by root. Leaving them world-readable is a security risk as they can contain sensitive data from memory.
 
---
 
### Step 3 — Format as swap
 
```bash
sudo mkswap /swapfile
```
 
**Why:** This writes the swap header to the file, making it recognisable to the kernel as a valid swap area.
 
---
 
### Step 4 — Activate swap
 
```bash
sudo swapon /swapfile
```
 
**Why:** Activates the swapfile immediately without requiring a reboot. Confirmed with:
 
```bash
swapon --show
```
 
Output:
 
```
NAME      TYPE SIZE USED PRIO
/swapfile file   4G   0B   -2
```
 
---
 
### Step 5 — Make swap permanent
 
```bash
sudo vim /etc/fstab
```
 
Added the following line at the bottom of the file:
 
```
/swapfile none swap sw 0 0
```
 
**Why:** Without this entry, the swapfile would not be mounted after a reboot. `/etc/fstab` is read at boot to determine what gets mounted and activated.
 
Verified with:
 
```bash
cat /etc/fstab | grep swapfile
# Output: /swapfile none swap sw 0 0
```
 
---
 
### Step 6 — Lower swappiness
 
```bash
sudo sysctl vm.swappiness=10
```
 
**Why:** The default value of `60` tells the kernel to start pushing memory to swap when RAM is still relatively available. Lowering it to `10` instructs the kernel to exhaust RAM first and only use swap as a genuine last resort — better behaviour for a machine with 7GB of physical RAM.
 
Made permanent by appending to `/etc/sysctl.conf`:
 
```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```
 
Verified with:
 
```bash
grep swappiness /etc/sysctl.conf
# Output: vm.swappiness=10
```
 
---
 
## Final System State
 
| Metric | Before | After |
|---|---|---|
| Swap | 0B | 4GB (active + permanent) |
| vm.swappiness | 60 | 10 (permanent) |
| OOM kill risk | High | Low |
 
---
 
## Notes
 
- Swap is **not** extra RAM. It is slower disk-backed overflow. Accessing data in swap is significantly slower than RAM due to NVMe latency vs RAM latency.
- If RAM usage is consistently near 7GB during heavy sessions, the long-term solution is a physical RAM upgrade to 16GB.
- Tab discipline in Chrome is recommended. Each tab runs as a separate process and contributes meaningfully to RAM consumption.
