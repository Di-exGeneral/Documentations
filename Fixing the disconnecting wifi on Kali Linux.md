# WiFi Random Disconnects on Kali Linux
 
## Problem
 
WiFi was disconnecting at random intervals on a fresh Kali Linux install with a built-in WiFi card.
 
## Cause
 
Kali enables WiFi power management by default. The card was being aggressively suspended mid-connection to save power.
 
## Fix
 
### 1. Confirm power management is on
 
```bash
iwconfig 2>/dev/null | grep -i "power management"
```
 
Expected output:
```
Power Management:on
```
 
### 2. Disable it temporarily
 
```bash
sudo iwconfig wlan0 power off
```
 
### 3. Make it permanent via NetworkManager
 
```bash
sudo vim /etc/NetworkManager/conf.d/wifi-power-management.conf
```
 
Add the following:
 
```ini
[connection]
wifi.powersave = 2
```
 
### 4. Restart NetworkManager
 
```bash
sudo systemctl restart NetworkManager
```
 
### 5. Verify
 
```bash
iwconfig 2>/dev/null | grep -i "power management"
```
 
Expected output:
```
Power Management:off
```
 
## Result
 
WiFi card stays fully powered and no longer drops the connection at random intervals. Fix persists across reboots.
 
