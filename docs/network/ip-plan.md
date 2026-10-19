# IP Addressing Plan

> **Public architecture documentation** — This document describes the homelab's logical IPv4 addressing strategy, VLAN subnets, and DHCP design. It is not an operational address inventory or a substitute for the live network configuration.

## Purpose and Scope

The IP plan provides a consistent addressing model for the Proxmox-based homelab. Each security zone has its own subnet, while infrastructure services use predictable addressing to support management, name resolution, monitoring, and inter-service dependencies.

This document covers:

- VLAN and subnet allocation
- Default gateway conventions
- DHCP scope design
- Static addressing and reservation principles
- DNS placement and redundancy
- Address-management and change-control rules

**Exact host-to-IP mappings are deliberately excluded.** Operational address records, DHCP reservations, and device-specific management endpoints are maintained separately in private infrastructure documentation.

For zone definitions and traffic policy, see [`vlan-design.md`](vlan-design.md) and [`acl-policy.md`](acl-policy.md).

---

## 1. Addressing Strategy

The homelab uses private RFC1918 IPv4 addressing. Production VLANs use a `/24` subnet, with the third octet corresponding to the VLAN ID. This provides a simple relationship between an address and its logical security zone.

The legacy `DEFAULT` network is the exception to this naming convention.

### VLAN and Subnet Plan

| VLAN | Zone | Subnet | Default Gateway | DHCP |
|---:|---|---|---|---|
| 1 | `DEFAULT` | `192.168.0.0/24` | `192.168.0.1` | Disabled |
| 10 | `INFRA` | `192.168.10.0/24` | `192.168.10.1` | Enabled |
| 20 | `TRUSTED` | `192.168.20.0/24` | `192.168.20.1` | Enabled |
| 25 | `GAME-DMZ` | `192.168.25.0/24` | `192.168.25.1` | Enabled |
| 30 | `IOT` | `192.168.30.0/24` | `192.168.30.1` | Enabled |
| 40 | `GUEST` | `192.168.40.0/24` | `192.168.40.1` | Enabled |

The TP-Link ER605 V2 provides the VLAN gateways and DHCP service. Inter-VLAN communication is governed by gateway ACLs; sharing a router does **not** imply unrestricted connectivity between zones.

---

## 2. DHCP Design

The documented DHCP configuration uses a consistent dynamic range in each active VLAN.

| VLAN | Zone | DHCP Pool |
|---:|---|---|
| 10 | `INFRA` | `192.168.10.100–192.168.10.199` |
| 20 | `TRUSTED` | `192.168.20.100–192.168.20.199` |
| 25 | `GAME-DMZ` | `192.168.25.100–192.168.25.199` |
| 30 | `IOT` | `192.168.30.100–192.168.30.199` |
| 40 | `GUEST` | `192.168.40.100–192.168.40.199` |

**Documented lease duration:** 120 minutes.  
**`DEFAULT`:** DHCP disabled.

### Allocation Principles

- **Infrastructure and critical services:** Use static configuration or DHCP reservations where stable addresses are needed.
- **Personal and ordinary client devices:** Prefer DHCP unless a fixed address is operationally justified.
- **Guest devices:** Use dynamic DHCP without persistent public device records.
- **IoT devices:** Use DHCP or reservations according to application dependencies.
- **Address collisions:** Ensure that fixed addresses do not conflict with dynamically assigned leases. Where reservations are used within a DHCP pool, they must be explicitly managed by the DHCP service.

The exact static assignments, reservations, exclusions, and MAC-address bindings are private operational records, not part of this public plan.

---

## 3. Zone Allocation

### `INFRA` — VLAN 10

**Purpose:** Core systems and infrastructure services.

Typical roles include hypervisor management, storage, backup, DNS, network control, home automation, and internal applications. Systems relied upon by multiple workloads receive predictable addresses.

The public documentation identifies these **roles**, but does not map individual hosts or management interfaces to their live IP addresses.

### `TRUSTED` — VLAN 20

**Purpose:** Trusted personal endpoints and administrative clients.

Routine clients primarily use DHCP. This repository does not publish individual endpoint assignments, device identifiers, or a personal-device inventory.

### `GAME-DMZ` — VLAN 25

**Purpose:** Game-server workloads, including services that may be reachable through an external tunneling solution.

The design distinguishes between **hypervisor management** and **game workloads**. Management remains within the infrastructure zone, while game services are placed in `GAME-DMZ`. The precise VM addresses and routing details are maintained privately.

### `IOT` — VLAN 30

**Purpose:** Smart-home devices, televisions, and other lower-trust connected appliances.

IoT endpoints obtain addresses through DHCP or reservations. Home Assistant is hosted in `INFRA`; required communication with IoT endpoints is handled through controlled inter-zone policy rather than by moving Home Assistant into the IoT network.

### `GUEST` — VLAN 40

**Purpose:** Temporary guest clients.

Guest devices use DHCP. Individual assignments are intentionally omitted from public documentation. Cross-zone restrictions are covered in [`acl-policy.md`](acl-policy.md).

### `DEFAULT` — VLAN 1

**Purpose:** Restricted legacy/default network.

This network retains the original subnet, does not offer DHCP, and is not intended for normal client placement. Its operational management details are kept private.

---

## 4. DNS Architecture

Two Pi-hole instances provide DNS resolution for the active VLANs. They are hosted on **separate physical Proxmox nodes** to reduce dependence on any single compute host.

| Component | Function | Placement |
|---|---|---|
| Pi-hole 1 | DNS resolver | ProDesk |
| Pi-hole 2 | DNS resolver | EliteDesk |

The VLAN DHCP configurations advertise both resolvers. Restricted zones are intended to receive only the DNS access required by the inter-VLAN policy, rather than general access to `INFRA`.

Separating the resolvers across hosts improves service resilience, but does **not** establish full DNS high availability: common network, gateway, power, and upstream dependencies can still affect both instances.

The resolvers' exact addresses and client configuration values are excluded from this public document.

---

## 5. Address Management Standards

### Stable Infrastructure Addresses

Use predictable addressing for systems referenced by DNS, ACL objects, backup jobs, monitoring, storage mounts, or integrations. A changed address can break multiple dependent services even when the underlying system is healthy.

### Role-Based Zone Placement

Assign a device to the VLAN appropriate to its role and trust level. For example:

| Role | Intended Zone |
|---|---|
| Hypervisor and infrastructure management | `INFRA` |
| Trusted workstation | `TRUSTED` |
| Game-server workload | `GAME-DMZ` |
| Consumer IoT endpoint | `IOT` |
| Guest client | `GUEST` |

### Controlled Reassignment

Do not reuse an infrastructure address without checking references in DNS, ACLs, DHCP, monitoring, backup configurations, and service dependencies. Keep the private inventory synchronized with the active configuration.

---

## 6. Change Control and Verification

Before introducing a new subnet, moving a workload, or changing a fixed address:

1. Confirm that the proposed zone matches the workload's purpose and trust requirements.
2. Check DHCP pools, reservations, and address collisions.
3. Review dependencies in DNS, firewall/ACL policy, storage, backups, and monitoring.
4. Apply the change in the relevant network or host configuration.
5. Test address assignment, gateway reachability, DNS resolution, and the required cross-zone service paths.
6. Verify that prohibited cross-zone paths remain restricted.
7. Update the **private** host-address inventory and the appropriate **public** architecture documentation.

The source document describes all listed production VLANs and the dual-DNS design as operational. **Those status claims have not been independently revalidated during this public-documentation rewrite.** The live gateway and DHCP configuration remain the authoritative source for current operational values.

---

## 7. Public Documentation Boundary

The repository intentionally distinguishes between architectural information and sensitive operational detail.

| Public in this file | Private operational documentation |
|---|---|
| VLAN IDs and zone names | Exact host-to-IP mappings |
| Subnets and gateway convention | Management interface inventory |
| DHCP scope design | DHCP reservation/MAC mappings |
| Service categories and DNS design | Device-specific DNS settings |
| General addressing rules | Tailscale and remote-access addresses |
| Change-control approach | Detailed troubleshooting and recovery endpoints |

Passwords, API tokens, private keys, and recovery secrets must be stored in an appropriate secrets manager, **not in either repository**.

This separation preserves the educational and portfolio value of the network design without publishing a ready-made inventory of infrastructure targets.

---

## Related Documentation

- [`README.md`](README.md) — network overview
- [`vlan-design.md`](vlan-design.md) — network zones and trust model
- [`acl-policy.md`](acl-policy.md) — inter-VLAN access policy
- [`port-mapping.md`](port-mapping.md) — physical network documentation
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — logical service relationships
