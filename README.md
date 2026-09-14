# Multi-Site Enterprise Core Network with Remote Pi-hole & Automated SNMP Monitoring

A homelab project simulating a multi-site enterprise network using virtualization, site-to-site VPN tunnels, a remote DNS sinkhole (Pi-hole), and automated SNMP-based monitoring built entirely on consumer hardware.

## Overview

This project recreates the core pieces of an enterprise WAN(World Area network) connected over encrypted tunnels, centralized monitoring, and a remote edge location running its own services. Instead of racks of hardware, the sites are virtualized on a single host and connected to real physical nodes.

## Architecture

"""
Proxmox VE host (i3, 8GB)
├── OPNsense/VyOS — Site A router
├── OPNsense/VyOS — Site B router
└── LibreNMS VM
    └── SNMP + alerts

WireGuard tunnels
├── Laptop, 4GB
│   └── WireGuard client
└── Raspberry Pi — Site C router
    ├── Pi-hole DNS
    └── WireGuard client + snmpd agent
"""
## Hardware

| Role | Device | Specs |
|---|---|---|
| Core hypervisor | Primary PC | i3 10th Gen, 8 GB RAM |
| Site C node | Secondary laptop | Low-spec, 4 GB RAM |
| Remote edge | Raspberry Pi | <1 GB RAM |

## Stack

- **Hypervisor:** Proxmox VE
- **Routing/Firewall:** OPNsense or VyOS (virtualized per site)
- **Site-to-site connectivity:** WireGuard
- **DNS filtering:** Pi-hole (on Raspberry Pi)
- **Monitoring:** LibreNMS (SNMP polling, alerting)
- **SNMP agent:** net-snmp (`snmpd`) on all endpoints

## Build phases

1. **Virtualization foundation** — Proxmox install, virtual bridges/VLANs per site
2. **Core routing** — deploy per-site router VMs, configure NAT/firewall rules
3. **Site-to-site VPN** — WireGuard tunnels linking all sites
4. **Remote edge** — Pi-hole + WireGuard client + snmpd on the Raspberry Pi
5. **Monitoring** — LibreNMS VM, SNMP-enabled devices, automated alerts
6. **Hardening** — VLAN segmentation, IDS (Suricata), config backups, syslog

## Setup

### 1. Proxmox host
```bash
# After installing Proxmox VE, create a Linux bridge per simulated site
# Datacenter > <node> > System > Network > Create > Linux Bridge
```

### 2. Router VMs
- Deploy OPNsense/VyOS ISO as a VM per site (512MB–1GB RAM each)
- Assign each to its site's virtual bridge + a shared "WAN" bridge

### 3. WireGuard tunnels
```bash
# On each router/node
wg genkey | tee privatekey | wg pubkey > publickey
```
Add peer configs referencing each site's public key and endpoint.

### 4. Pi-hole on Raspberry Pi
```bash
curl -sSL https://install.pi-hole.net | bash
```

### 5. SNMP agent (all nodes)
```bash
sudo apt install snmpd
sudo nano /etc/snmp/snmpd.conf   # set community string, agentAddress
sudo systemctl restart snmpd
```

### 6. LibreNMS
Deploy via the [official Docker/VM install guide](https://docs.librenms.org/), then add each router/Pi as a device using its SNMP community string.

## Status / Roadmap

- [ ] Site A & B router VMs deployed
- [ ] WireGuard mesh established
- [ ] Pi-hole live and resolving for remote clients
- [ ] LibreNMS polling all nodes
- [ ] Alerting configured (email/Discord webhook)
- [ ] VLAN segmentation added
- [ ] IDS/IPS enabled

## License

MIT
