# Current State

This document describes the currently documented state of the homelab.

It is intended to provide a quick, portfolio-friendly snapshot of the active architecture. Detailed hardware specifications, IP addresses, VM configuration, ACL rules, and service procedures remain in the specialized documents linked below.

**Last reviewed:** 2026-09-22

**Status:** Active architecture documentation

---

## Purpose

The homelab is a multi-node Proxmox and Linux environment used for practical work with:

- virtualization and Linux administration
- storage and ZFS
- VLANs, routing, DNS, and ACLs
- backup and recovery
- monitoring and remote administration
- service isolation and troubleshooting

The design separates workloads by role and failure domain instead of placing every service on one host.

---

## Architecture at a Glance

- Four local Proxmox VE hosts provide compute and service hosting.
- `pve-main` provides the primary storage platform and storage-dependent workloads.
- `prodesk` provides infrastructure services, management services, Proxmox Backup Server, and the NUT UPS server.
- `elitedesk` provides secondary DNS on separate physical hardware.
- `pve-game-server` hosts the dedicated game-server environment.
- An offsite ThinkCentre M900 provides independent monitoring and remote-access services.
- TP-Link Omada provides routing, switching, wireless access, and gateway ACL enforcement.
- Workloads are separated using VLANs based on purpose and trust level.
- A CyberPower CP1600EPFCLCD UPS provides power-loss protection for selected infrastructure systems.

---

## Physical Hosts

| Host | Current role | Main workloads or responsibilities |
|---|---|---|
| `pve-main` | Primary Proxmox and storage host | TrueNAS, Immich, Linux/Debian workloads, Satisfactory, experimental Kubernetes capacity |
| `prodesk` | Infrastructure and backup host | Omada Controller, Pi-hole DNS 1, Home Assistant, dashboard, PBS, NUT server |
| `elitedesk` | Secondary DNS host | Pi-hole DNS 2 |
| `pve-game-server` | Dedicated game-server host | Game Server VM, Palworld, Valheim, Playit.gg |
| ThinkCentre M900 | Offsite monitoring and utility node | Uptime Kuma, Tailscale, SSH administration, selected utility tasks |

The administration workstation is a trusted client used to manage the infrastructure. It is not a server dependency and the homelab is not designed to require the workstation to remain online.

Detailed physical connections are documented in [`physical-topology.md`](physical-topology.md).

---

## Workload Placement

### `pve-main`

`pve-main` is focused on storage and workloads that depend on the storage platform.

Current responsibilities include:

- TrueNAS
- Immich
- Linux/Debian server workloads
- Satisfactory
- experimental Kubernetes workloads

The older Linux server workload remains on this host while the game-server environment is being consolidated on `pve-game-server`.

### `prodesk`

`prodesk` is the primary local infrastructure and backup node.

It hosts services that support the rest of the environment:

- Omada Controller
- Pi-hole DNS 1
- Home Assistant
- operations dashboard
- Proxmox Backup Server
- NUT UPS server

Moving these services away from `pve-main` reduces the number of infrastructure services affected by maintenance or failure of the primary storage host.

### `elitedesk`

`elitedesk` has a deliberately narrow role: running the secondary Pi-hole instance on separate physical hardware.

This provides DNS redundancy without turning the node into another general-purpose infrastructure host.

### `pve-game-server`

The dedicated game-server host runs game workloads in a separate virtualized environment.

The primary game VM is currently VM 400, with additional game-server workloads such as VM 420 where applicable. Game workloads are placed in `GAME-DMZ`, while the Proxmox management interface remains in `INFRA`.

This allows game services to be updated, restarted, rebuilt, or exposed through Playit.gg without placing the hypervisor management plane in the game-server zone.

### Offsite ThinkCentre M900

The M900 is physically separate from the main homelab. It provides monitoring and remote-access functions that remain available when the local environment is unavailable.

It is an offsite monitoring and utility node, not the primary backup destination for all homelab data.

---

## Network State

The network is managed through TP-Link Omada.

| Component | Current role |
|---|---|
| TP-Link ER605 V2 | Gateway, routing, DHCP, NAT, and gateway ACL enforcement |
| TP-Link SG3210X-M2 | Main managed 2.5 GbE switch |
| TP-Link ES216G | Secondary managed access switch |
| TP-Link EAP670 | Wireless access point |
| Omada Controller | Central network-management platform on `prodesk` |

### VLANs and Trust Zones

| VLAN | Name | Current purpose |
|---:|---|---|
| 1 | `DEFAULT` | Restricted legacy/default network |
| 10 | `INFRA` | Proxmox management, servers, and infrastructure |
| 20 | `TRUSTED` | Trusted personal and administrative clients |
| 25 | `GAME-DMZ` | Game-server workloads |
| 30 | `IOT` | Smart-home and lower-trust IoT devices |
| 40 | `GUEST` | Guest clients with no general internal access |

A separate administrative segment is also used for the Codex VM. Its detailed addressing and implementation belong in the relevant network documentation rather than this high-level snapshot.

### Documented Communication Model

- Trusted clients may administer infrastructure through explicit trusted paths.
- Infrastructure management interfaces are not intended to be exposed directly to the Internet.
- Game workloads receive Internet access and required DNS access, but do not receive general access to private infrastructure.
- IoT devices use internal DNS and receive only the required exceptions, including Home Assistant integrations.
- Guest clients receive Internet access without general access to internal zones.
- The default network is restricted and is not intended to be the normal administration network.

The detailed policy and rule ordering are documented in [`../network/acl-policy.md`](../network/acl-policy.md).

---

## Storage and Backup State

TrueNAS runs as a VM on `pve-main`. The physical data disks are passed through using the LSI 9207-8i HBA in IT mode so that TrueNAS and ZFS manage the disks directly.

The main storage platform includes:

- a four-disk 8 TB ZFS pool with single-disk parity
- a mirrored pair of 2 TB disks used for Immich storage
- separate storage for work files and local VM workloads

Proxmox Backup Server runs on the separate `prodesk` host using a dedicated datastore. The current documented retention policy is:

- three latest backups
- eight weekly backups
- twelve monthly backups

The current backup design provides useful VM/LXC recovery coverage, but it is not a complete second copy of every TrueNAS dataset. Full NAS-data backup and broader offsite data protection remain limitations of the current design.

Detailed backup scope and recovery procedures are documented in [`../security/backup-strategy.md`](../security/backup-strategy.md).

---

## Monitoring, Remote Access, and Power

### Monitoring

Uptime Kuma runs on the offsite ThinkCentre M900. Because it is outside the main physical environment, it can detect when local services, hosts, or the local network become unavailable.

### Remote Access

Tailscale is used for secure remote administration and monitoring paths. It complements the VLAN design and does not replace local network segmentation.

### UPS and Power-Loss Protection

A CyberPower CP1600EPFCLCD 1600 VA / 1000 W UPS is part of the active architecture.

- NUT runs on `prodesk`.
- Home Assistant monitors UPS state and load.
- Selected infrastructure systems are configured for controlled shutdown.
- Network equipment connected to the UPS is protected during short outages, but does not necessarily participate in the same controlled shutdown sequence.
- The offsite monitor can detect an outage, but cannot prevent it.

The UPS therefore improves short-outage resilience and reduces the risk of uncontrolled shutdowns, but it does not provide unlimited runtime or replace backups.

---

## Current Exceptions and Limitations

The following points are important when interpreting the current architecture:

1. Satisfactory is still hosted on the Linux/Debian workload on `pve-main`; migration to the dedicated game-server host remains a future task.
2. TrueNAS bulk data is not fully duplicated to an independent offsite storage system.
3. Experimental Kubernetes workloads are not treated as production services and may not be backed up.
4. UPS shutdown behavior applies only to configured clients; power protection and controlled shutdown are not identical for every device.
5. Detailed IP addresses, switch ports, VMIDs, ACL rule ordering, and service-specific dependencies are maintained in their specialized documents and may change independently of this summary.

This document should describe the current state, not every historical state or planned feature.

---

## Related Documentation

- [`overview.md`](overview.md) — architectural goals and high-level design
- [`physical-topology.md`](physical-topology.md) — physical hosts, cabling, and hardware connections
- [`logical-topology.md`](logical-topology.md) — VM, container, service, and dependency relationships
- [`../network/vlan-design.md`](../network/vlan-design.md) — VLAN and trust-zone design
- [`../network/acl-policy.md`](../network/acl-policy.md) — inter-VLAN access policy
- [`../security/backup-strategy.md`](../security/backup-strategy.md) — backup and recovery strategy
- [`../nodes/offsite-m900.md`](../nodes/offsite-m900.md) — offsite monitoring node
- [`../nodes/pve-gameserver.md`](../nodes/pve-gameserver.md) — dedicated game-server host

---

## Updating This Document

Update this file when one of the following changes:

- a physical host changes role
- a major service moves between hosts
- a new trust zone or network boundary is introduced
- backup ownership or recovery scope changes
- UPS or monitoring behavior changes

Keep detailed configuration in the specialized documents and keep this file focused on the active architectural state.
