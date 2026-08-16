# IP Plan

This document defines the internal IP-addressing plan for the homelab.

It is the authoritative location for:

- VLAN subnets
- Default gateways
- DHCP pools
- DNS servers
- Fixed infrastructure addresses
- Addressing conventions

It intentionally does **not** contain:

- Public/WAN addresses
- Upstream provider DNS
- MAC addresses
- Wi-Fi SSIDs
- Tailscale addresses
- Credentials
- API tokens
- Private keys
- Webhook URLs

Only RFC1918 private addressing required to understand and operate the internal architecture is documented here.

For the reasoning behind the network zones, see [`vlan-design.md`](vlan-design.md).

For inter-VLAN access policy, see [`acl-policy.md`](acl-policy.md).

---

## Addressing Strategy

Each VLAN uses its own `/24` IPv4 subnet.

The third octet generally follows the VLAN ID:

```text
VLAN 10  → 192.168.10.0/24
VLAN 20  → 192.168.20.0/24
VLAN 25  → 192.168.25.0/24
VLAN 30  → 192.168.30.0/24
VLAN 40  → 192.168.40.0/24
```

This makes the relationship between a device address and its security zone easy to identify during administration and troubleshooting.

`DEFAULT` is the exception and retains the original `192.168.0.0/24` network.

---

## VLAN and Subnet Plan

| VLAN | Name | Subnet | Gateway | DHCP |
|---:|---|---|---|---|
| 1 | `DEFAULT` | `192.168.0.0/24` | `192.168.0.1` | Disabled |
| 10 | `INFRA` | `192.168.10.0/24` | `192.168.10.1` | Enabled |
| 20 | `TRUSTED` | `192.168.20.0/24` | `192.168.20.1` | Enabled |
| 25 | `GAME-DMZ` | `192.168.25.0/24` | `192.168.25.1` | Enabled |
| 30 | `IOT` | `192.168.30.0/24` | `192.168.30.1` | Enabled |
| 40 | `GUEST` | `192.168.40.0/24` | `192.168.40.1` | Enabled |

---

## DHCP Plan

DHCP is provided by the ER605 gateway.

The active VLANs currently use the following pools:

| VLAN | Network | DHCP Pool |
|---:|---|---|
| 10 | `INFRA` | `192.168.10.100 – 192.168.10.199` |
| 20 | `TRUSTED` | `192.168.20.100 – 192.168.20.199` |
| 25 | `GAME-DMZ` | `192.168.25.100 – 192.168.25.199` |
| 30 | `IOT` | `192.168.30.100 – 192.168.30.199` |
| 40 | `GUEST` | `192.168.40.100 – 192.168.40.199` |

Current lease time:

```text
120 minutes
```

`DEFAULT` does not provide DHCP.

---

## Address Allocation Model

Addresses are divided conceptually into two categories.

### Infrastructure / Predictable Addresses

Important infrastructure systems use fixed or reserved addresses.

These addresses are documented because they are referenced by:

- DNS configuration
- ACL rules
- Monitoring
- Management tools
- Service dependencies
- Backup configuration

### Dynamic Client Addresses

Normal client devices should primarily use DHCP.

Individual transient client addresses are not documented in this public repository unless they are architecturally important.

This keeps the IP plan focused on infrastructure rather than becoming a complete device-tracking database.

---

# INFRA — VLAN 10

Subnet:

```text
192.168.10.0/24
```

Gateway:

```text
192.168.10.1
```

Primary role:

- Hypervisor management
- Internal servers
- Storage
- DNS
- Backup
- Network management
- Infrastructure devices

---

## Core Infrastructure Addresses

| System | Address | Role |
|---|---|---|
| Main Proxmox | `192.168.10.10` | Primary hypervisor |
| EliteDesk Proxmox | `192.168.10.20` | Secondary DNS host |
| ProDesk Proxmox | `192.168.10.21` | Infrastructure and backup hypervisor |
| Game Server Proxmox | `192.168.10.30` | Game-host hypervisor management |
| TrueNAS | `192.168.10.40` | NAS / ZFS storage |
| Proxmox Backup Server | `192.168.10.50` | Backup platform on `prodesk` |
| Home Assistant | `192.168.10.70` | Home automation on `prodesk` |
| Immich | `192.168.10.71` | Photo/video service |
| Pi-hole 1 | `192.168.10.90` | Primary DNS on `prodesk` |
| Pi-hole 2 | `192.168.10.91` | Secondary DNS on `elitedesk` |
| Omada Controller | `192.168.10.100` | Network controller on `prodesk` |
| SG3210X-M2 | `192.168.10.102` | Core switch management |
| ES216G | `192.168.10.103` | Access switch management |
| EAP670 | `192.168.10.104` | Access-point management |
| JetKVM | `192.168.10.110` | Remote physical console |

These addresses are infrastructure anchors rather than a complete inventory of every host that may exist in `INFRA`.

---

## INFRA Addressing Pattern

The current addressing layout loosely groups systems by function:

```text
.1        Gateway

.10–.39   Hypervisors / physical compute
.40–.59   Storage and backup
.60–.79   Core applications
.90–.99   DNS / supporting infrastructure
.100+     Network management and reserved infrastructure
```

This is a convention rather than a hard technical requirement.

New infrastructure should preferably follow the existing structure when practical.

---

# TRUSTED — VLAN 20

Subnet:

```text
192.168.20.0/24
```

Gateway:

```text
192.168.20.1
```

Primary role:

- Main administration workstation
- Trusted personal devices
- Approved client devices

Most trusted clients use DHCP.

The public IP plan intentionally does not maintain a complete list of personal client devices or their fixed addresses.

This avoids turning the repository into a detailed inventory of personal endpoints.

---

# GAME-DMZ — VLAN 25

Subnet:

```text
192.168.25.0/24
```

Gateway:

```text
192.168.25.1
```

Primary role:

- Game-server workloads
- Externally reachable game services
- Playit.gg-connected systems

Important logical distinction:

```text
Game Proxmox management → INFRA
Game workload           → GAME-DMZ
```

This keeps the Proxmox management interface out of the workload DMZ.

---

## Game Infrastructure Addresses

| System | Address | Role |
|---|---|---|
| Game Server VM | `192.168.25.10` | Primary dedicated game-server VM |
| Legacy Linux Game Server | `192.168.25.20` | Existing/migrating game workload |

The legacy address may be retired once the remaining workloads have been fully migrated to the dedicated game-server platform.

---

# IOT — VLAN 30

Subnet:

```text
192.168.30.0/24
```

Gateway:

```text
192.168.30.1
```

Primary role:

- Smart-home devices
- TVs
- Appliances
- Vendor-managed consumer IoT

Most IoT devices use DHCP reservations or dynamic DHCP.

Individual device addresses are intentionally not listed in the public IP plan unless they become required for infrastructure policy.

This avoids publishing a complete inventory of personal smart-home devices.

Home Assistant itself does **not** reside in this subnet. It remains on `INFRA` and receives explicit ACL access toward `IOT`.

---

# GUEST — VLAN 40

Subnet:

```text
192.168.40.0/24
```

Gateway:

```text
192.168.40.1
```

Primary role:

- Temporary guest devices

Guest devices use dynamic DHCP.

No individual guest addresses are documented.

---

# DEFAULT — VLAN 1

Subnet:

```text
192.168.0.0/24
```

Gateway:

```text
192.168.0.1
```

Primary role:

- Restricted legacy/management network

DHCP:

```text
Disabled
```

The network is not intended for ordinary clients.

The ER605 management interface remains on this network by design.

No normal endpoint inventory is maintained for `DEFAULT`.

---

## DNS

Two Pi-hole instances provide DNS to the active VLANs.

| DNS Server | Address | Host |
|---|---|---|
| Pi-hole 1 | `192.168.10.90` | `prodesk` |
| Pi-hole 2 | `192.168.10.91` | `elitedesk` |

The active VLANs are configured to use both addresses.

Conceptually:

```text
INFRA     ─┐
TRUSTED   ─┤
GAME-DMZ  ─┼──► Pi-hole 1 / Pi-hole 2
IOT       ─┤
GUEST     ─┘
```

Restricted zones receive only the DNS access required by the ACL policy rather than unrestricted access to `INFRA`.

The ACL implementation is documented in [`acl-policy.md`](acl-policy.md).

---

## DNS Redundancy

The two DNS servers are hosted on separate physical Proxmox nodes.

```text
Pi-hole 1 → prodesk
Pi-hole 2 → elitedesk
```

This provides host-level separation.

Maintenance or failure of one Proxmox node should therefore not automatically remove both DNS services.

---

## Address Reservations

Important infrastructure should use predictable addresses through either:

- Static configuration
- DHCP reservation

The public repository documents the resulting IP address, but **not the corresponding MAC address**.

MAC-address mappings are operational details and are intentionally excluded from the public documentation.

---

## Addressing Rules

New infrastructure should follow these rules where practical.

### 1. Use the Correct VLAN

A device's network should reflect its role.

Examples:

```text
Hypervisor management → INFRA
Trusted workstation   → TRUSTED
Game workload         → GAME-DMZ
Smart device          → IOT
Guest client          → GUEST
```

---

### 2. Infrastructure Gets Predictable Addresses

Services referenced by:

- DNS
- ACLs
- monitoring
- mounts
- API integrations
- backup jobs

should not depend on unpredictable dynamic addresses.

---

### 3. Clients Prefer DHCP

Personal devices, guest devices, and ordinary clients should use DHCP unless a fixed address provides a clear operational benefit.

---

### 4. Do Not Reuse Addresses Casually

An address previously associated with infrastructure should not immediately be reassigned to an unrelated device.

Stable addressing makes:

- Logs
- troubleshooting
- documentation
- ACL rules
- monitoring

easier to reason about.

---

### 5. Keep Management Addresses Stable

Addresses for systems such as:

- Proxmox
- Omada
- managed switches
- access points
- TrueNAS
- PBS
- Pi-hole

should be treated as infrastructure anchors.

Changes to these addresses can affect multiple downstream integrations and should therefore be planned rather than performed casually.

---

## Public Repository Sanitization

This document intentionally includes RFC1918 internal addresses because they are part of the technical network design.

The following information is deliberately excluded:

| Excluded Information | Reason |
|---|---|
| WAN/public address | Unnecessary external exposure |
| Upstream/provider addressing | Not part of the internal architecture |
| MAC addresses | Device-identifying operational detail |
| Wi-Fi SSIDs | Personal/environment-specific information |
| Tailscale addresses | Remote-access inventory |
| Personal endpoint inventory | Unnecessary device disclosure |
| Credentials/tokens/keys | Secrets |
| Webhook URLs | Secrets / externally actionable endpoints |

The goal is to document the architecture without publishing information that provides no portfolio value.

---

## Change Management

Changes to the IP plan can have wider consequences than the address itself.

Before changing an infrastructure address, verify dependencies such as:

- DNS
- DHCP reservations
- ACL object groups
- NFS/SMB configuration
- monitoring
- dashboard integrations
- backup jobs
- service configuration
- bookmarks/runbooks

Infrastructure-address changes should therefore be treated as controlled changes.

---

## Current State

Current addressing status:

- `DEFAULT` configured and restricted
- `INFRA` operational
- `TRUSTED` operational
- `GAME-DMZ` operational
- `IOT` operational
- `GUEST` operational
- Dual Pi-hole DNS operational
- Core infrastructure uses predictable addresses
- DHCP available on active client/service VLANs
- DHCP disabled on `DEFAULT`

The address plan should be treated as implemented current state rather than a future migration plan.

---

## Scope of This Document

This file owns:

- VLAN subnets
- gateways
- DHCP pools
- DNS addresses
- fixed infrastructure addresses
- addressing conventions

It does not own:

- Why VLANs exist
- ACL rule definitions
- Switch port assignments
- MAC addresses
- Public/WAN information
- Incident history

Those belong in the corresponding network documents.

---

## Related Documentation

- [`README.md`](README.md) — network overview
- [`vlan-design.md`](vlan-design.md) — VLAN purpose and trust model
- [`acl-policy.md`](acl-policy.md) — inter-VLAN access policy
- [`port-mapping.md`](port-mapping.md) — switch and trunk configuration
- [`incidents/es216g-recovery.md`](incidents/es216g-recovery.md) — ES216G incident
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — services and logical relationships
