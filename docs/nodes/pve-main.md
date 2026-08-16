# Main Proxmox Node — `pve-main`

This document describes the primary Proxmox VE host in the homelab.

The node is the main local storage and virtualization platform. Following the Server 2.0 workload cleanup, several supporting infrastructure services were moved to the ProDesk node so that `pve-main` can remain focused on storage-heavy workloads and selected application services.

Detailed configuration for individual applications is documented under [`../services/`](../services/).

---

## Overview

| Item | Value |
|---|---|
| Hostname | `pve-main` |
| Platform | Proxmox VE |
| Primary role | Core virtualization and storage host |
| Management network | `INFRA` / VLAN 10 |
| Management address | `192.168.10.10` |
| Network link | 2.5 GbE |
| Backup target | Proxmox Backup Server on `prodesk` |

`pve-main` is the largest local compute and storage node in the homelab.

Its most important responsibilities are:

- hosting TrueNAS
- providing the physical disk platform for ZFS storage
- hosting Immich
- hosting the remaining Satisfactory game server workload
- providing capacity for selected lab and test workloads

Core support services such as DNS, network management, Home Assistant, the dashboard, and PBS have been moved to other physical nodes.

---

# Role

The node has three primary responsibilities.

## 1. Storage Platform

The physical HDDs used by TrueNAS are installed in this system and connected through an LSI HBA.

TrueNAS receives direct access to those disks and is responsible for the ZFS storage layout.

## 2. Core Application Hosting

The node hosts services that benefit from its storage capacity or have a direct relationship with TrueNAS.

The main examples are:

- TrueNAS
- Immich
- Linux/Debian server hosting Satisfactory

## 3. Lab Capacity

The node can also provide compute for temporary or experimental workloads when spare capacity is available.

Experimental workloads are intentionally treated as lower priority than the core storage and application services.

---

# Hardware

## Core System

| Component | Hardware |
|---|---|
| CPU | Intel Core i9-9900K |
| Memory | 64 GB DDR4 |
| Motherboard | ASUS ROG Strix Z390-F Gaming |
| CPU cooler | Noctua NH-D15 |
| GPU | NVIDIA GTX 1070 Ti |
| Primary NIC | Intel I226-T1 2.5 GbE |
| HBA | LSI 9207-8i, IT mode |
| Chassis | Fractal Design Define 7 |

The system combines virtualization capacity with a comparatively large number of directly attached disks.

The HBA is important because it allows TrueNAS to manage the storage disks directly rather than using virtual disks created by Proxmox.

---

# Storage

## Physical Storage Layout

| Storage | Role |
|---|---|
| 1 TB Samsung 970 EVO Plus NVMe | Proxmox VE and VM storage |
| 4 × 8 TB HDD | Primary TrueNAS ZFS pool |
| 2 × 2 TB HDD | Mirrored storage used for Immich |
| 1 × 3 TB HDD | Workdrive / general storage |
| Additional local SSD storage | Auxiliary Proxmox storage |

The main storage layout is represented conceptually as:

```text
pve-main
├── Proxmox VE
│   └── 1 TB NVMe
│       ├── Proxmox OS
│       └── VM storage
│
└── TrueNAS VM
    ├── 4 × 8 TB HDD
    │   └── RAIDZ1
    │       └── 1-disk parity
    │
    ├── 2 × 2 TB HDD
    │   └── Mirror
    │       └── Immich storage
    │
    └── 3 TB HDD
        └── Workdrive
```

The exact ZFS pools, datasets, shares, and permissions belong in:

[`../services/truenas.md`](../services/truenas.md)

---

# Storage Architecture

The HDDs are passed through to TrueNAS rather than presented as ordinary Proxmox virtual disks.

```text
Physical HDDs
     │
     ▼
LSI 9207-8i HBA
     │
     ▼
PCIe passthrough
     │
     ▼
TrueNAS VM
     │
     ▼
ZFS pools / datasets
```

This gives TrueNAS direct control of the storage devices and allows ZFS to manage the physical disks as intended.

---

## Important Backup Consequence

A backup of the TrueNAS VM does not automatically include the data stored on the passed-through HDDs.

These are two separate recovery problems:

```text
TrueNAS VM
    ≠
TrueNAS ZFS data
```

Proxmox Backup Server can protect the VM configuration and virtual boot disk.

Bulk NAS data requires a separate backup strategy.

This distinction is documented in:

[`../security/backup-strategy.md`](../security/backup-strategy.md)

---

# Virtualization

The node-based VMID convention reserves the `100–199` range for workloads assigned to `pve-main`.

## Core Workloads

| ID | Workload | Type | Role | Criticality |
|---:|---|---|---|---|
| 100 | Linux / Debian Server | VM | Satisfactory / SteamCMD | Medium |
| 101 | TrueNAS SCALE | VM | NAS, ZFS, SMB/NFS | High |
| 106 | Immich | VM | Photo and video management | High |

The most important workloads on this node are now directly related to storage or services that benefit from being close to the storage platform.

---

## Experimental Workloads

A Kubernetes lab has also existed on the node.

Historically this included:

| ID | Workload | Role |
|---:|---|---|
| 550 | k8s-base | Template / lab base |
| 551 | k8s-cp1 | Kubernetes control plane |
| 552 | k8s-w1 | Kubernetes worker |
| 553 | k8s-w2 | Kubernetes worker |

These workloads are experimental and are not considered part of the production-like homelab architecture.

They may be rebuilt, renumbered, relocated, or removed as the Kubernetes project develops.

---

# Workload Layout

```text
Fractal Server
└── Proxmox VE
    ├── TrueNAS SCALE [VM]
    ├── Immich [VM]
    └── Linux / Debian [VM]
        └── Satisfactory Server / SteamCMD
```

The architecture is intentionally simpler than before the Server 2.0 cleanup.

Several infrastructure services previously hosted here were migrated to ProDesk.

---

# TrueNAS

TrueNAS is the central storage service in the homelab.

Its responsibilities include:

- ZFS storage
- SMB
- NFS
- bulk data storage
- storage used by Immich
- Workdrive storage

The disks are physically installed in `pve-main`, but logically managed by the TrueNAS VM.

Documentation:

[`../services/truenas.md`](../services/truenas.md)

---

# Immich

Immich provides self-hosted photo and video management.

Its media storage is provided by TrueNAS.

Logical dependency:

```text
Immich
   │
   ▼
TrueNAS
   │
   ▼
2 × 2 TB ZFS mirror
```

This means the Immich VM can technically remain powered on while its storage dependency is unavailable.

Service health therefore requires validating both:

```text
Immich VM state
        +
TrueNAS storage availability
```

Documentation:

[`../services/immich.md`](../services/immich.md)

---

# Linux / Debian Server

The Linux VM currently hosts the remaining Satisfactory dedicated server workload.

```text
Linux / Debian VM
└── Satisfactory Server
    └── SteamCMD
```

Palworld, Valheim, and Minecraft have been moved to the dedicated game-server node.

This reduces game-related workload concentration on the main infrastructure server.

Documentation:

[`../services/game-servers.md`](../services/game-servers.md)

---

# Networking

The node connects to the managed Omada network through the SG3210X-M2 core switch.

```text
pve-main
    │
    │ 2.5 GbE
    ▼
SG3210X-M2
    │
    ▼
INFRA — VLAN 10
```

The Proxmox management interface resides on `INFRA`.

The switch connection can carry additional VLANs to virtual workloads where required.

Detailed networking is documented in:

- [`../network/vlan-design.md`](../network/vlan-design.md)
- [`../network/ip-plan.md`](../network/ip-plan.md)
- [`../network/acl-policy.md`](../network/acl-policy.md)
- [`../network/port-mapping.md`](../network/port-mapping.md)

---

# Management

Primary management methods include:

- Proxmox web interface
- SSH
- Proxmox VM console
- JetKVM for physical remote-console access
- Tailscale for remote administration where required

The main administration workstation resides on `TRUSTED`.

Management interfaces are not intentionally exposed directly to the public Internet.

---

# Backup

The Proxmox Backup Server is no longer hosted on the EliteDesk.

PBS now runs on the ProDesk node with a dedicated 2 TB NVMe datastore.

```text
pve-main
    │
    │ VM / LXC backups
    ▼
ProDesk
    │
    └── Proxmox Backup Server
            │
            ▼
       2 TB NVMe datastore
```

This keeps the backup target physically separate from the host being protected.

Documentation:

[`../services/proxmox-backup-server.md`](../services/proxmox-backup-server.md)

---

## Backup Scope

PBS is used to protect selected virtual workloads and configuration.

Bulk TrueNAS data is not stored in PBS because the NAS dataset is significantly larger than the available PBS datastore.

The backup architecture therefore separates:

```text
Virtual workload backup
        and
Bulk data backup
```

The authoritative backup policy belongs in:

[`../security/backup-strategy.md`](../security/backup-strategy.md)

---

# Infrastructure Services Moved to ProDesk

As part of the Server 2.0 cleanup, several workloads were migrated away from `pve-main`.

Current placement:

```text
ProDesk
├── 200  Omada Controller
├── 201  Pi-hole DNS 1
├── 202  Home Assistant
├── 203  Homepage Dashboard
└── 299  Proxmox Backup Server
```

This migration reduced the number of unrelated dependencies on the main storage server.

It also established a clearer physical-node responsibility model:

```text
pve-main
└── storage and storage-dependent workloads

ProDesk
└── infrastructure, management and backup

EliteDesk
└── secondary DNS

Game Server
└── dedicated game workloads
```

---

# DNS Relationship

Neither Pi-hole instance is hosted on `pve-main` anymore.

Current DNS placement:

```text
ProDesk
└── Pi-hole DNS 1

EliteDesk
└── Pi-hole DNS 2
```

This keeps DNS available independently of the main storage node.

Documentation:

[`../services/pihole.md`](../services/pihole.md)

---

# Failure Impact

A complete `pve-main` outage still has a significant impact because the primary storage system is hosted here.

## Affected

- TrueNAS
- bulk NAS storage
- Immich storage
- Immich
- Workdrive
- Satisfactory server
- workloads directly dependent on TrueNAS
- experimental workloads hosted on the node

## Not Automatically Affected

- Pi-hole DNS 1
- Pi-hole DNS 2
- Proxmox Backup Server
- Omada Controller
- Home Assistant
- Homepage Dashboard
- ProDesk
- EliteDesk
- dedicated game-server host
- Palworld
- Valheim
- Minecraft
- offsite Uptime Kuma monitoring

Moving infrastructure services away from `pve-main` reduces the blast radius of maintenance or host failure.

---

# Service Availability During Maintenance

| Service | Expected State |
|---|---|
| TrueNAS | Offline |
| Immich | Offline / storage unavailable |
| Workdrive | Offline |
| Satisfactory | Offline |
| Pi-hole DNS 1 | Available |
| Pi-hole DNS 2 | Available |
| PBS | Available |
| Omada Controller | Available |
| Home Assistant | Available |
| Homepage Dashboard | Available |
| Palworld | Available |
| Valheim | Available |
| Minecraft | Available |
| Offsite monitoring | Available |

This represents the intended architecture rather than a formal high-availability guarantee.

---

# Dependency Awareness

The most important local dependency chain is:

```text
pve-main
   │
   ▼
TrueNAS
   │
   ├── Immich storage
   └── Workdrive
```

For this reason, startup and maintenance procedures should account for storage availability before dependent applications are considered healthy.

A VM being in a `running` state does not necessarily mean the service is operational.

---

# Monitoring

The node is monitored through several layers.

## Proxmox

Used for:

- host health
- CPU and memory usage
- local storage
- VM state
- node availability

## Homepage Dashboard

The dashboard is hosted on ProDesk and provides a high-level view of the overall homelab.

Because it no longer runs on `pve-main`, it can remain available while the main node is under maintenance.

## Offsite Uptime Kuma

The offsite ThinkCentre provides independent availability monitoring from outside the main homelab.

This provides another failure domain for detecting:

- host outages
- network outages
- service outages

## Hardware Monitoring

Local hardware health is periodically checked using tools such as:

- `smartctl`
- `nvme-cli`
- Linux hardware sensors

Disk temperature monitoring is particularly relevant because the node contains multiple HDDs in addition to the primary NVMe and other local storage.

---

# Maintenance

Typical host maintenance includes:

- Proxmox VE updates
- kernel and security updates
- SMART/NVMe health checks
- disk temperature checks
- HBA visibility checks
- VM state validation
- TrueNAS storage validation
- backup validation
- airflow and dust inspection

---

## Pre-Maintenance Checklist

Before planned downtime:

1. Confirm recent backups for important virtual workloads.
2. Confirm PBS on ProDesk is reachable.
3. Confirm no important TrueNAS storage operation is active.
4. Confirm Immich is not performing important storage work.
5. Confirm external infrastructure services on ProDesk and EliteDesk are healthy.
6. Shut down dependent workloads cleanly.
7. Shut down TrueNAS before powering down the host.

---

## Post-Maintenance Validation

After reboot or hardware maintenance, verify:

1. Proxmox node health.
2. Local Proxmox storage.
3. HBA visibility.
4. TrueNAS startup.
5. ZFS pool health.
6. SMB/NFS availability.
7. Immich storage access.
8. Immich application health.
9. Satisfactory server.
10. PBS connectivity.
11. Offsite monitoring state.

A successful Proxmox boot alone does not prove that the storage stack and dependent applications are healthy.

---

# Security Considerations

The node contains high-value storage and virtualization infrastructure.

Important controls include:

- Proxmox management on `INFRA`
- administration from `TRUSTED`
- no intentional direct public exposure of Proxmox
- Tailscale for remote administration where appropriate
- VLAN and ACL segmentation
- game workloads moved away from the main infrastructure node
- backup storage on separate physical hardware
- credentials and secrets excluded from the public repository

---

# Design Decisions

## Virtualized TrueNAS

TrueNAS remains virtualized on `pve-main`.

The LSI HBA is passed through to the VM so that TrueNAS retains direct control of the physical disks used by ZFS.

This provides efficient use of the available hardware while still allowing TrueNAS to manage its own storage stack.

The tradeoff is that a complete `pve-main` outage also removes access to the NAS.

---

## Storage-Focused Main Node

The Server 2.0 migration reduced the number of unrelated infrastructure workloads hosted on `pve-main`.

Services such as:

- DNS
- PBS
- Omada Controller
- Home Assistant
- Homepage

were moved to ProDesk or EliteDesk.

The main node can therefore remain focused on workloads where its storage capacity provides a clear benefit.

---

## Separate Infrastructure Node

ProDesk now hosts most supporting infrastructure.

This improves service separation and allows several management services to remain available during `pve-main` maintenance.

---

## Separate Secondary DNS Node

EliteDesk hosts Pi-hole DNS 2.

This prevents both DNS instances from sharing the same physical failure domain.

---

## Dedicated Game-Server Node

Palworld, Valheim, and Minecraft are hosted on the dedicated game-server Proxmox node.

Only Satisfactory currently remains on the Linux VM on `pve-main`.

This keeps externally oriented game workloads increasingly separate from storage and private infrastructure.

---

## Node-Based VMID Convention

The current workload convention uses physical-node ranges:

```text
pve-main      100–199
ProDesk       200–299
EliteDesk     300–399
Game Server   400–499
```

This makes workload placement easier to identify and keeps the Proxmox environment more consistent.

Experimental Kubernetes workloads still use historical IDs and are intentionally treated as outside the finalized workload layout until that project is revisited.

---

# Current Status

`pve-main` is operational as the primary storage and core virtualization node.

Current architecture:

- TrueNAS: active
- Immich: active
- Satisfactory server: active
- primary ZFS storage: active
- Immich mirror storage: active
- Workdrive: active
- PBS: hosted on ProDesk
- Pi-hole DNS 1: hosted on ProDesk
- Pi-hole DNS 2: hosted on EliteDesk
- Omada Controller: hosted on ProDesk
- Home Assistant: hosted on ProDesk
- Homepage Dashboard: hosted on ProDesk
- Palworld / Valheim / Minecraft: hosted on dedicated game-server node
- Kubernetes: experimental / outside current production-like architecture

The current direction is to keep `pve-main` focused on storage and workloads that directly benefit from that storage.

---

# Public Repository Notes

This document intentionally excludes:

- passwords
- API tokens
- private keys
- Tailscale addresses
- MAC addresses
- serial numbers
- credentials
- public/WAN information
- unredacted configuration exports

Private RFC1918 management addressing is included where it provides architectural value.

---

# Scope of This Document

This file owns documentation for the physical `pve-main` node:

- hardware
- role
- local storage
- HBA relationship
- virtualization
- workload inventory
- networking
- backup relationship
- dependencies
- failure impact
- maintenance
- architectural decisions

It does not own detailed configuration for:

- TrueNAS
- Immich
- game servers
- PBS
- Pi-hole
- VLANs
- ACLs
- switch port configuration

Those belong in their dedicated service, security, and network documents.

---

## Related Documentation

- [`../architecture/overview.md`](../architecture/overview.md) — overall architecture
- [`../architecture/physical-topology.md`](../architecture/physical-topology.md) — physical topology
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — workload topology
- [`../network/README.md`](../network/README.md) — network overview
- [`../network/vlan-design.md`](../network/vlan-design.md) — VLAN design
- [`../network/acl-policy.md`](../network/acl-policy.md) — ACL policy
- [`../network/port-mapping.md`](../network/port-mapping.md) — physical switch connectivity
- [`../services/truenas.md`](../services/truenas.md) — storage service
- [`../services/immich.md`](../services/immich.md) — Immich
- [`../services/game-servers.md`](../services/game-servers.md) — dedicated game services
- [`../services/proxmox-backup-server.md`](../services/proxmox-backup-server.md) — PBS
- [`../security/backup-strategy.md`](../security/backup-strategy.md) — backup architecture
- [`pve-prodesk.md`](pve-prodesk.md) — infrastructure and backup node
- [`pve-elitedesk.md`](pve-elitedesk.md) — secondary DNS node
- [`pve-gameserver.md`](pve-gameserver.md) — dedicated game-server node
- [`offsite-m900.md`](offsite-m900.md) — offsite monitoring node
