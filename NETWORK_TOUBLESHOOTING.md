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
```










# NETWORK TROUBLESHOOTING LOG

## [INC-002]

**Date/Time:** 2026-10-01 12:27
**Affected Host:** Proxmox Server (Host PVE)
**Operator:** Marcell
**Severity:** Medium (VMs/containers have no direct LAN access; host management unaffected)

---

### 1. Description of the Problem (Symptom)

While running a Proxmox helper script to create an LXC container, the *Network Bridge* selector only offered `vmbr1`. That bridge was the isolated NAT bridge created in INC-001 (`bridge-ports none`), so any container attached to it would be unreachable from the LAN and would get no DHCP from the router. The goal was to attach the USB Ethernet adapter (`enx000000000000`) so containers could share the physical network.

### 2. Initial Diagnosis (Investigation)

- **Associated OSI Layer:** Layer 2 (Data Link) / Layer 3 (Network)
- **Physical Environment:** USB-A to RJ45 Ethernet adapter, same as INC-001.
- **Findings:**
  - Containers and VMs attach to Linux bridges (`vmbr*`), never directly to a physical interface, so `enx000000000000` could not be selected in the script.
  - `vmbr1` had `bridge-ports none`: a virtual switch with no uplink, hence no external connectivity.

### 3. Plan of Action & Root Cause Analysis

1. In *Node → System → Network*, edited `vmbr1` and set **Bridge ports** to `enx000000000000`.
2. Proxmox rejected the change with:
   `iface enx000000000000 - ip address can't be set on interface if bridged in vmbr1 (500)`
   - **Root cause:** the physical interface still had its own IP/gateway. An interface enslaved to a bridge acts only as a port of the virtual switch and cannot hold an IP.
   - **Secondary issue:** `vmbr1` still had `10.10.X.1/24` (internal NAT range) together with gateway `192.168.X.X`. The gateway must be in the same subnet as the bridge IP.
3. **Fix applied:**
   - Cleared IPv4/CIDR and Gateway on `enx000000000000`.
   - Moved the host IP and default gateway to `vmbr1` (see Figure 1): IPv4/CIDR `192.168.X.X/24`, Gateway `192.168.X.X`, Bridge ports `enx000000000000`.
   - Only one default gateway should remain on the system (the one on `vmbr1`).
4. **Pending:** press **OK**, then **Apply Configuration**, and verify (`ip a`, `bridge link`, `ping -c 4 192.168.X.X`, and DHCP on a test container).

> **Figure 1:** *Edit: Linux Bridge* dialog for `vmbr1` with the final values.
>
> <img width="629" height="269" alt="image" src="https://github.com/user-attachments/assets/17bc502f-42a9-46f1-999d-f1886feb0f0e" />


**Known risk (from INC-001):** the USB adapter previously failed to work reliably as a port of `vmbr0`. If connectivity drops again after applying this change, check the MAC of the bridge vs. the adapter, promiscuous mode, and offloading (`ethtool -K enx000000000000 tx off rx off`). If it still fails, **rollback** to the INC-001 configuration (IP on the USB interface + isolated NAT `vmbr1`).

**Topology Change:**

- **Before:** Router → USB Adapter (host IP) → Proxmox Host; `vmbr1` isolated (NAT) → VMs
- **After (proposed):** Router (192.168.1.1) → USB Adapter (no IP, bridge port) → `vmbr1` (192.168.1.2, host + VMs/CTs on the LAN)

### 4. Applied Network Configuration (`/etc/network/interfaces`)

*(For security reasons, this is not my real MAC nor my real IP.)*

```
auto lo
iface lo inet loopback

iface enx000000000000 inet manual

auto vmbr1
iface vmbr1 inet static
    address 192.168.1.2/24
    gateway 192.168.1.1
    bridge-ports enx000000000000
    bridge-stp off
    bridge-fd 0
```
