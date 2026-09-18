# NETWORK TROUBLESHOOTING LOG

## [INC-001]
**Date/Time:** 2026-09-17 18:36  
**Affected Host:** Proxmox Server (Host PVE)  
**Operator:** Marcell 
**Severity:** High (No network connection / Inaccessible)  

### 1. Description of the Problem (Symptom)
The newly installed Proxmox VE server does not respond to ping requests nor allow access to the web interface (GUI), even though the client device is on the same subnet.

### 2. Initial Diagnosis (Investigation)
* **Associated OSI Layer:** Layer 2 (Data Link) / Layer 3 (Network)
* **Physical Environment:** USB-A to RJ45 Ethernet adapter connected to the server's USB port.
* **Connectivity Check:** Verified physical link state and router reachability via local ICMP ping tests (`ping -c 4 <ROUTER_IP>`).

### 3. Plan of Action & Root Cause Analysis

1. **Host Access Recovery:** Gained root access via single-user mode to reset credentials since the system didn't acept my original account settings.

2. **Remount Filesystem and mount again the file system with write permission:** 

   ```
   mount -o remount,rw /
   passwd
   ```
   
**3. Access with my account as root**
**4. Execute ip link show** to see if it's connectet to the right interface 
**6. Activate promicuate mode** to receive all Mac address even tho they weren't destinated to the device soo it could see the traffic
**7 Check Mac** I Check if the MAC of vmbr0 (the bridge of proxmox that work as a virtual switch) is different than the MAC of the physical device
**8 The problem** 
The external USB Ethernet adapter failed to function reliably when bound directly as a port under the default Proxmox Linux bridge (vmbr0) meaning that the error was that my USB device doesn't work with the bridge (vmbr0) so I had to desactivate it.

**Solution:**

Bypassed vmbr0 for host management by assigning the primary IP directly to the USB interface (enx...), then set up an isolated internal bridge (vmbr1) with NAT masquerading to route traffic for virtual machines.

Topology Change:



Before: Router (192.168.1.1) → USB Adapter → vmbr0 (Virtual Switch) → Proxmox Host & VMs

After: Router (192.168.1.1) → USB Adapter (192.168.1.2) → Proxmox Host → vmbr1 (Isolated NAT Bridge) → VMs

### 4. Applied Network Configuration (/etc/network/interfaces)
I configured my /etc/network/interfaces so  enx00e04c574720 stay as my straight and stable connection and vmbr1 become my only point of conectivity to VMs:

(For security reasons, this is not my MAC nor my IP.)
```
auto lo
iface lo inet loopback

allow-hotplug enx000000000000
iface enx000000000000 inet static
    address 192.168.1.2/24
    gateway 192.168.1.1
    post-up /sbin/ethtool -K enx000000000000 tx off rx off
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -F POSTROUTING
    post-up iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o enx000000000000 -j MASQUERADE
'''
