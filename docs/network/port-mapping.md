# Port Mapping

This document defines the physical and logical switch-port mapping for the homelab.

It is the authoritative location for:

- Physical device-to-port connections
- Access and trunk roles
- Native VLANs
- Tagged VLANs on uplinks
- Disabled unused ports

It intentionally does **not** contain:

- Full IP addressing
- DHCP reservations
- MAC addresses
- ACL rules
- Credentials
- Application configuration

Related documentation:

- [`vlan-design.md`](vlan-design.md) — VLAN purpose and trust model
- [`ip-plan.md`](ip-plan.md) — addressing and DNS
- [`acl-policy.md`](acl-policy.md) — inter-VLAN policy
- [`../architecture/physical-topology.md`](../architecture/physical-topology.md) — overall physical topology

---

## Overview

The local wired network is built around two managed switches:

| Switch | Role |
|---|---|
| TP-Link SG3210X-M2 | Core switch |
| TP-Link ES216G | Secondary access switch |

The SG3210X-M2 connects the main infrastructure hosts, gateway, wireless access point, and the ES216G.

The ES216G provides additional access ports for local client and management devices.

Unused switch ports are intentionally **disabled**.

---

# SG3210X-M2

## Role

The SG3210X-M2 is the primary aggregation switch for the homelab.

It connects:

- Administration workstation
- Main Proxmox host
- Game-server Proxmox host
- EliteDesk Proxmox host
- ProDesk Proxmox host
- ES216G
- EAP670
- ER605 gateway

---

## Port Map

| Port | Connected Device | Port Role | Native VLAN | Tagged VLANs | Status |
|---:|---|---|---|---|---|
| 1 | Fractal administration PC | Access | `TRUSTED (20)` | — | Active |
| 2 | Main Proxmox — `pve-main` | Trunk | `INFRA (10)` | Required workload VLANs | Active |
| 3 | Game Server Proxmox — `pve-game-server` | Trunk | `INFRA (10)` | Required workload VLANs | Active |
| 4 | EliteDesk Proxmox — `elitedesk` | Trunk | `INFRA (10)` | Required workload VLANs | Active |
| 5 | ProDesk Proxmox — `prodesk` | Trunk | `INFRA (10)` | Required workload VLANs | Active |
| 6 | ES216G uplink | Trunk | `DEFAULT (1)` | `INFRA (10)`, `TRUSTED (20)`, `GAME-DMZ (25)`, `IOT (30)`, `GUEST (40)` | Active |
| 7 | EAP670 | Trunk | `INFRA (10)` | Wireless client VLANs | Active |
| 8 | ER605 V2 | Trunk / Gateway uplink | `DEFAULT (1)` | `INFRA (10)`, `TRUSTED (20)`, `GAME-DMZ (25)`, `IOT (30)`, `GUEST (40)` | Active |
| 9 | — | Disabled | — | — | Disabled |
| 10 | — | Disabled | — | — | Disabled |

---

## Port 1 — Administration Workstation

```text
SG3210X-M2 Port 1
        │
        ▼
Fractal / Fedora Workstation
```

Role:

```text
Access port
```

Native VLAN:

```text
TRUSTED (20)
```

The workstation is the primary administration client for the homelab.

It does not require trunking because the workstation itself belongs to the trusted client network.

---

## Port 2 — Main Proxmox

```text
SG3210X-M2 Port 2
        │
        ▼
pve-main
```

Role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

The physical Proxmox host uses `INFRA` for management.

Additional VLANs may be presented to virtual workloads hosted on the node.

This allows the physical management interface and virtual workloads to use different network zones without requiring multiple physical NIC connections.

---

## Port 3 — Game Server Proxmox

```text
SG3210X-M2 Port 3
        │
        ▼
pve-game-server
```

Role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

The Proxmox management interface resides on `INFRA`.

Game-server workloads can be attached to `GAME-DMZ` through the same physical uplink.

Conceptually:

```text
Physical Proxmox management → INFRA
Virtual game workload       → GAME-DMZ
```

This keeps the hypervisor management plane separate from the game workload network.

---

## Port 4 — EliteDesk Proxmox

```text
SG3210X-M2 Port 4
        │
        ▼
elitedesk
```

Role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

The EliteDesk currently hosts the secondary Pi-hole DNS instance.

The trunk keeps Proxmox host management on `INFRA` and leaves room for workload VLANs if required later.

---

## Port 5 — ProDesk Proxmox

```text
SG3210X-M2 Port 5
        │
        ▼
prodesk
```

Role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

Tagged VLANs:

```text
Required workload VLANs
```

The ProDesk hosts infrastructure and backup workloads including:

- Proxmox Backup Server
- Pi-hole DNS 1
- Home Assistant
- Omada Controller
- Homepage dashboard

The physical Proxmox management interface remains on `INFRA`, while hosted workloads can use additional VLANs as required.

---

## Port 6 — ES216G Uplink

```text
SG3210X-M2 Port 6
        │
        ▼
ES216G Port 1
```

Role:

```text
Trunk
```

Native VLAN:

```text
DEFAULT (1)
```

Tagged VLANs:

```text
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

This is the primary uplink between the core switch and the secondary access switch.

The ES216G management interface resides on `INFRA`, which is transported tagged across this uplink.

> **Important:** This uplink is a known management-critical path. Changes to its native VLAN, tagged VLAN membership, or management behavior should be treated as controlled changes with a rollback plan.

The previous ES216G incident is documented in:

[`incidents/es216g-recovery.md`](incidents/es216g-recovery.md)

---

## Port 7 — EAP670

```text
SG3210X-M2 Port 7
        │
        ▼
EAP670
```

Role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

The access point management interface uses `INFRA`.

Wireless client networks are transported using tagged VLANs.

This allows separate wireless networks to place clients directly into the appropriate security zones.

---

## Port 8 — ER605 Gateway

```text
SG3210X-M2 Port 8
        │
        ▼
ER605 V2
```

Role:

```text
Gateway trunk
```

Native VLAN:

```text
DEFAULT (1)
```

Tagged VLANs:

```text
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

This link carries routed VLAN traffic between the core switch and the gateway.

The ER605 provides:

- VLAN gateways
- DHCP
- Routing
- NAT
- Gateway ACL enforcement

---

# ES216G

## Role

The ES216G is the secondary access switch.

It connects downstream wired devices and uplinks to the SG3210X-M2.

Unused ports are disabled.

---

## Port Map

| Port | Connected Device | Port Role | Native VLAN | Tagged VLANs | Status |
|---:|---|---|---|---|---|
| 1 | SG3210X-M2 port 6 | Trunk | `DEFAULT (1)` | `INFRA (10)`, `TRUSTED (20)`, `GAME-DMZ (25)`, `IOT (30)`, `GUEST (40)` | Active |
| 2 | — | Disabled | — | — | Disabled |
| 3 | PlayStation 5 | Access | `TRUSTED (20)` | — | Active |
| 4 | — | Disabled | — | — | Disabled |
| 5 | Samsung TV | Access | `IOT (30)` | — | Active |
| 6 | — | Disabled | — | — | Disabled |
| 7 | — | Disabled | — | — | Disabled |
| 8 | — | Disabled | — | — | Disabled |
| 9 | — | Disabled | — | — | Disabled |
| 10 | — | Disabled | — | — | Disabled |
| 11 | — | Disabled | — | — | Disabled |
| 12 | — | Disabled | — | — | Disabled |
| 13 | — | Disabled | — | — | Disabled |
| 14 | — | Disabled | — | — | Disabled |
| 15 | — | Disabled | — | — | Disabled |
| 16 | JetKVM | Access | `INFRA (10)` | — | Active |

---

## Port 1 — Core Uplink

```text
ES216G Port 1
      │
      ▼
SG3210X-M2 Port 6
```

Role:

```text
Trunk
```

Native VLAN:

```text
DEFAULT (1)
```

Tagged VLANs:

```text
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

The switch management interface itself uses `INFRA`.

This means management traffic crosses the uplink as tagged VLAN 10 traffic.

---

## Port 3 — PlayStation 5

```text
ES216G Port 3
      │
      ▼
PlayStation 5
```

Role:

```text
Access
```

Native VLAN:

```text
TRUSTED (20)
```

The console is treated as a trusted personal client rather than as infrastructure or IoT.

---

## Port 5 — Samsung TV

```text
ES216G Port 5
      │
      ▼
Samsung TV
```

Role:

```text
Access
```

Native VLAN:

```text
IOT (30)
```

The TV belongs to the IoT security zone.

It receives Internet and DNS access according to the IoT policy without unrestricted access to private internal systems.

---

## Port 16 — JetKVM

```text
ES216G Port 16
      │
      ▼
JetKVM
```

Role:

```text
Access
```

Native VLAN:

```text
INFRA (10)
```

JetKVM is treated as infrastructure because it provides remote management access to physical hardware.

Its network access therefore belongs on the management/infrastructure network rather than on a normal client VLAN.

---

# Disabled Ports

All currently unused physical switch ports are disabled.

This applies to:

### SG3210X-M2

```text
Port 9
Port 10
```

### ES216G

```text
Port 2
Port 4
Ports 6–15
```

The unused ports have not been repurposed or assigned temporary client profiles.

---

## Why Unused Ports Are Disabled

Disabling unused switch ports provides several benefits:

- Prevents accidental connection to an unintended VLAN
- Reduces unauthorized physical access paths
- Makes active topology easier to understand
- Reduces ambiguity during troubleshooting
- Avoids stale access/trunk profiles on unused interfaces

A port should be explicitly configured for its intended role before being enabled.

---

# Trunk Relationships

The main trunk relationships are:

```text
ER605 V2
    │
    │ VLAN 1, 10, 20, 25, 30, 40
    ▼
SG3210X-M2
    │
    ├── pve-main
    ├── pve-game-server
    ├── elitedesk
    ├── prodesk
    ├── EAP670
    │
    │ VLAN 1, 10, 20, 25, 30, 40
    ▼
ES216G
```

Each trunk has a different purpose:

| Trunk | Purpose |
|---|---|
| ER605 ↔ SG3210X-M2 | Routed VLAN uplink |
| SG3210X-M2 ↔ ES216G | Carry VLANs to secondary switch |
| SG3210X-M2 ↔ Proxmox hosts | Carry management and VM workload networks |
| SG3210X-M2 ↔ EAP670 | Carry AP management and wireless client VLANs |

---

# Access-Port Relationships

Access ports place normal endpoints into a single VLAN.

Current examples:

```text
Fractal       → TRUSTED
PlayStation 5 → TRUSTED
Samsung TV    → IOT
JetKVM        → INFRA
```

Endpoints connected to these ports do not need to understand VLAN tagging.

---

# Management Traffic

Management devices are generally placed on `INFRA`.

Examples include:

- Proxmox hosts
- SG3210X-M2
- ES216G
- EAP670
- JetKVM
- Omada Controller

The ER605 is the exception and retains its management interface on `DEFAULT`.

The management design is documented further in [`vlan-design.md`](vlan-design.md).

---

# Omada Client Display Caveat

Omada may display a VM or LXC as the apparent connected client on a physical Proxmox switch port.

For example, a port physically connected to a Proxmox host may show the name of a VM currently generating traffic.

This does **not** change the physical port mapping.

The authoritative physical connections are:

```text
SG3210X-M2 P2 → pve-main
SG3210X-M2 P3 → pve-game-server
SG3210X-M2 P4 → elitedesk
SG3210X-M2 P5 → prodesk
SG3210X-M2 P6 → ES216G
```

The switch-port map in this document should therefore be treated as the physical source of truth.

---

# Change-Control Guidelines

Switch-port changes can affect both client connectivity and management access.

Before modifying an active port:

1. Identify the physical device connected to the port.
2. Confirm whether the port is access or trunk.
3. Record the current native VLAN.
4. Record all required tagged VLANs.
5. Determine whether the port carries management traffic.
6. Prepare a rollback path.
7. Make one change at a time.
8. Verify the connected device immediately afterward.

Additional caution is required for:

- Gateway uplinks
- Inter-switch trunks
- Proxmox trunks
- AP trunks
- Management VLAN changes

---

## Enabling a Currently Disabled Port

A disabled port should not simply be enabled with an arbitrary existing profile.

Before enabling it:

1. Identify the intended endpoint.
2. Determine the correct VLAN.
3. Decide whether the endpoint requires access or trunk mode.
4. Apply only the VLANs required by that device.
5. Enable the port.
6. Verify addressing and connectivity.
7. Update this document.

---

# Current State

Current physical switch state:

- Core gateway uplink: active
- Main Proxmox trunk: active
- Game-server Proxmox trunk: active
- EliteDesk Proxmox trunk: active
- ProDesk Proxmox trunk: active
- ES216G uplink: active
- EAP670 trunk: active
- Fractal access port: active
- PlayStation 5 access port: active
- Samsung TV access port: active
- JetKVM access port: active
- All unused ports: disabled

The port map should be treated as implemented current state rather than a planned topology.

---

# Public Repository Notes

This document intentionally excludes:

- MAC addresses
- management credentials
- Wi-Fi credentials
- API tokens
- WAN/public addressing
- Tailscale addressing
- cable identifiers that provide no architectural value

The goal is to document the topology and VLAN behavior without exposing unnecessary device-specific information.

---

# Scope of This Document

This file owns:

- Physical switch-port mapping
- Access/trunk roles
- Native VLAN assignments
- Tagged VLANs on important trunks
- Disabled-port state
- Inter-switch relationships

It does not own:

- VLAN design rationale
- IP addressing
- DHCP
- DNS
- ACL rules
- Incident details

---

## Related Documentation

- [`README.md`](README.md) — network overview
- [`vlan-design.md`](vlan-design.md) — VLAN design and trust model
- [`ip-plan.md`](ip-plan.md) — addressing and DNS
- [`acl-policy.md`](acl-policy.md) — inter-VLAN policy
- [`incidents/es216g-recovery.md`](incidents/es216g-recovery.md) — ES216G management incident
- [`../architecture/physical-topology.md`](../architecture/physical-topology.md) — full homelab physical topology
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — VM and service topology
