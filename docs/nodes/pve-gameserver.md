# Game Server Proxmox Node — `pve-game-server`

This document describes the dedicated Proxmox VE host used for game-server workloads.

The node exists to separate externally reachable, frequently updated, and restart-heavy game services from the core homelab infrastructure.

Detailed game-server configuration is documented in [`../services/game-servers.md`](../services/game-servers.md).

---

## Overview

| Item | Value |
|---|---|
| Hostname | `pve-game-server` |
| Platform | Proxmox VE |
| Primary role | Dedicated game-server virtualization |
| Management network | `INFRA` / VLAN 10 |
| Management address | `192.168.10.30` |
| Workload network | `GAME-DMZ` / VLAN 25 |
| Primary game VM | `192.168.25.10` |
| Core switch connection | SG3210X-M2 port 3 |

The physical Proxmox host and the game workloads intentionally use different network zones.

```text
Proxmox management → INFRA
Game workloads     → GAME-DMZ
```

This keeps the hypervisor management plane separate from the externally oriented game-server environment.

---

## Role

The node has one clear responsibility:

> Run game-server workloads independently from the core infrastructure.

It was introduced so that game services no longer need to share the main Proxmox host with services such as:

- TrueNAS
- Immich
- Home Assistant
- Pi-hole
- Omada Controller

This allows the game environment to be:

- restarted independently
- updated independently
- rebuilt independently
- isolated from private infrastructure
- exposed externally without exposing the hypervisor management plane

---

## Hardware

| Component | Hardware |
|---|---|
| Chassis | Lian Li A3 mATX |
| CPU | Intel Core i5-14500 |
| Motherboard | ASUS Prime B760M-A WIFI |
| Memory | 32 GB G.Skill Trident Z5 DDR5-6400 |
| CPU cooler | Noctua NH-U12A |
| Storage | 1 TB Samsung SSD |
| Network | Onboard Ethernet |

The system was built specifically as a compact, dedicated service host rather than repurposing the primary storage/infrastructure server.

---

## Why This Hardware

### Intel Core i5-14500

The i5-14500 provides a mix of performance and efficiency cores suitable for hosting multiple game-server processes while retaining enough capacity for the Proxmox host itself.

Game servers can have different CPU characteristics:

- some depend heavily on strong single-thread performance
- others benefit from additional cores when multiple worlds/services run simultaneously

A modern multi-core CPU gives the node enough flexibility to host several independent services without requiring workstation-class hardware.

---

### 32 GB DDR5

32 GB provides sufficient memory for the current game-server role while leaving headroom for:

- multiple game instances
- Linux guest overhead
- caching
- temporary maintenance workloads

The node is not intended as a general-purpose VM host, so capacity is deliberately sized around the game-server workload rather than maximum VM density.

---

### Local SSD Storage

The node uses local SSD storage for:

- Proxmox
- VM disks
- game-server files
- save data
- temporary backups prior to transfer to the backup system

Game workloads do not depend on TrueNAS for normal runtime operation.

This avoids making storage on the main Proxmox host a required dependency for the game-server node.

---

# Networking

The physical host connects directly to the SG3210X-M2 core switch.

```text
pve-game-server
      │
      ▼
SG3210X-M2
Port 3
```

Port role:

```text
Trunk
```

Native VLAN:

```text
INFRA (10)
```

The trunk allows the same physical connection to carry both:

- Proxmox management traffic on `INFRA`
- VM traffic on `GAME-DMZ`

Detailed port configuration is documented in:

[`../network/port-mapping.md`](../network/port-mapping.md)

---

## Management Plane

The Proxmox host management interface resides on:

```text
INFRA / VLAN 10
```

Management address:

```text
192.168.10.30
```

Administrative access originates from the `TRUSTED` network.

The host itself is not placed in `GAME-DMZ`.

This is an important security boundary:

```text
TRUSTED
   │
   ▼
INFRA
   │
   ▼
Proxmox Management

GAME-DMZ
   │
   ▼
Game Workloads
```

A compromised or misconfigured game VM should not automatically share the same management network as the physical hypervisor.

---

# Virtualization Layout

The node runs game workloads inside dedicated virtual machines rather than directly on the Proxmox host.

Current primary workload:

| Workload | Type | Network | Role |
|---|---|---|---|
| Game Server VM | VM | `GAME-DMZ` | Linux game-server environment |

Current primary game-server address:

```text
192.168.25.10
```

VM IDs are local to the individual Proxmox node and should not be interpreted as globally unique across the homelab.

---

## Why Use a VM?

Running game services inside a VM instead of directly on the Proxmox host provides:

- separation from the hypervisor
- easier backup and restore
- easier rebuilds
- easier network isolation
- cleaner service management
- reduced risk from application-level changes

The Proxmox host should remain focused on virtualization rather than becoming the application server itself.

---

# Game Services

The node is intended to host dedicated game servers such as:

- Palworld
- Valheim
- Satisfactory

Playit.gg is used where external connectivity is required.

Game-specific installation, update procedures, ports, save locations, and maintenance commands belong in:

[`../services/game-servers.md`](../services/game-servers.md)

---

## External Connectivity

The homelab is behind an upstream network where normal inbound port forwarding is not available.

Game traffic therefore uses Playit.gg.

Conceptually:

```text
Internet Player
      │
      ▼
Playit.gg
      │
      ▼
GAME-DMZ
      │
      ▼
Game Server VM
```

This keeps public game connectivity separate from remote administration.

### Remote Administration

Administrative access uses trusted internal paths or Tailscale.

### Public Game Access

Player traffic uses Playit.gg.

The two purposes are deliberately separated:

```text
Tailscale  → administration
Playit.gg  → game traffic
```

---

# GAME-DMZ

Game workloads reside on the dedicated `GAME-DMZ` VLAN.

The zone is designed for services with a higher external exposure profile than the private homelab infrastructure.

The game-server VM receives:

- Internet access
- Pi-hole DNS access
- Playit.gg connectivity

It is restricted from initiating general connections toward internal VLANs.

Administration is instead initiated from `TRUSTED`.

Detailed policy:

[`../network/acl-policy.md`](../network/acl-policy.md)

---

## DNS

Game workloads use the homelab Pi-hole DNS services.

The ACL policy explicitly permits DNS from `GAME-DMZ` toward the Pi-hole service without granting general access to `INFRA`.

This allows the game servers to retain internal DNS while maintaining network separation.

---

# Migration from Main Proxmox

Historically, game services ran on VM 100 — `Linux Server` — on the main `pve` host.

The dedicated game-server node was introduced to remove that responsibility from the core infrastructure server.

Migration model:

```text
Before

pve
├── TrueNAS
├── Immich
├── Home Assistant
├── Omada
├── Pi-hole
└── Game Servers
```

```text
Target / current direction

pve
├── Core infrastructure
├── Storage
└── Private services

pve-game-server
└── Game Servers
```

The migration is being completed incrementally so existing game services do not need to be moved simultaneously.

Any remaining legacy game workloads on `pve` should be treated as transitional rather than part of the long-term architecture.

---

# Backup

Game-server data should be protected independently of the runtime VM.

Important data includes:

- world saves
- configuration
- server settings
- application-specific save directories

The intended backup model has two layers:

```text
Game save / configuration backup
            +
Proxmox VM backup
```

A VM backup provides broad recovery, while game-specific save backups provide faster recovery of the most important application data.

---

## Proxmox Backup Server

The game-server node can use the PBS instance hosted on `elitedesk`.

```text
pve-game-server
      │
      ▼
PBS
      │
      ▼
elitedesk
```

This places the backup target on different physical hardware from the game-server host.

Detailed backup policy belongs in:

[`../security/backup-strategy.md`](../security/backup-strategy.md)

---

## Save-File Backups

Game-specific scripts are intended to protect save data independently from full VM backups.

These can be used before:

- server updates
- game version upgrades
- configuration changes
- mod changes
- maintenance

Reusable scripts should be stored under:

[`../../configs/scripts/`](../../configs/scripts/)

Sensitive values must remain outside the repository.

---

# Maintenance

Game-server workloads usually require more frequent application maintenance than the core homelab infrastructure.

Typical maintenance includes:

- Proxmox updates
- Linux guest updates
- SteamCMD/game updates
- server restarts
- save backups
- configuration changes
- Playit.gg validation
- storage-health checks

The dedicated node allows this work to occur without restarting unrelated core services.

---

## Pre-Update Workflow

Before a significant game-server update:

1. Check current player activity.
2. Stop the affected game server cleanly.
3. Create or verify a current save backup.
4. Apply the update.
5. Start the service.
6. Check logs.
7. Verify local service availability.
8. Verify external connectivity through Playit.gg.
9. Confirm that saved worlds load correctly.

---

## Post-Reboot Validation

After rebooting the physical node, verify:

- Proxmox management access
- local storage
- Game Server VM startup
- `GAME-DMZ` addressing
- Pi-hole DNS
- Internet access
- Playit.gg connection
- individual game services
- save-data availability
- backup connectivity to PBS

A running VM does not necessarily mean the game service itself is healthy.

---

# Failure Impact

A complete failure of `pve-game-server` primarily affects game workloads.

Affected:

- Game Server VM
- Palworld
- Valheim
- Satisfactory
- Playit.gg processes hosted on the game environment

Not automatically affected:

- TrueNAS
- Immich
- Home Assistant
- Pi-hole 1
- Pi-hole 2
- Omada Controller
- Proxmox Backup Server
- Homepage dashboard
- local network routing/switching
- offsite Uptime Kuma

This limited failure domain is one of the main reasons the node exists.

---

## Why the Failure Domain Matters

On the original architecture, game-server maintenance could share the same physical host as storage and private services.

With the dedicated node:

```text
Game Server Failure
        │
        └── Game services affected

Core Infrastructure
        └── remains independent
```

This reduces the operational blast radius of:

- game updates
- VM crashes
- experimental server configuration
- heavy game-server load
- reboots

---

# Monitoring

The node should be monitored at both host and application level.

### Proxmox Layer

Monitor:

- CPU usage
- memory usage
- storage
- VM state
- host temperature

### Game VM Layer

Monitor:

- guest availability
- CPU/RAM usage
- disk usage
- service state
- logs

### External Layer

Offsite Uptime Kuma can provide independent availability monitoring for selected services.

### Game-Level Validation

Where practical, game availability should be checked at the actual service level rather than relying only on ICMP or VM state.

---

# Security Considerations

Game servers have a different risk profile from private infrastructure because they may accept external traffic and run frequently updated third-party software.

Current architectural controls include:

- dedicated physical host
- dedicated game VM
- `GAME-DMZ`
- hypervisor management on `INFRA`
- Gateway ACL isolation
- administration from `TRUSTED`
- no direct public exposure of Proxmox management
- Playit.gg for player connectivity
- Tailscale for private remote administration

This does not make the workload risk-free, but it significantly reduces unnecessary trust relationships.

---

# Design Decisions

## Dedicated Physical Host

The strongest separation between the game environment and core services is achieved by putting them on different physical hardware.

The node can be rebooted without intentionally affecting:

- storage
- DNS redundancy
- backup infrastructure
- home automation
- network management

---

## Proxmox Instead of Bare Metal Game Hosting

Proxmox provides flexibility for:

- VM snapshots/backups
- workload separation
- rebuilding guests
- future multiple game VMs
- isolated virtual networking

The physical machine remains an infrastructure layer rather than becoming a single-purpose bare-metal game installation.

---

## Management on INFRA

The hypervisor itself is infrastructure.

It therefore belongs on `INFRA`, not in the game workload zone.

This keeps:

```text
management plane
```

separate from:

```text
application exposure plane
```

---

## GAME-DMZ for Workloads

Game-server VMs are intentionally placed in `GAME-DMZ`.

This follows the principle that external exposure should occur from a restricted zone rather than from the private server network.

---

## Local Runtime Storage

Game workloads use storage local to the game-server node for normal operation.

This reduces coupling to TrueNAS and prevents the game platform from depending on the main Proxmox host simply to run.

Backups can still be transferred to separate systems.

---

# Current State

The dedicated game-server architecture is operational.

Current confirmed state:

- `pve-game-server` Proxmox host: active
- Management on `INFRA`: active
- Dedicated Game Server VM on `GAME-DMZ`: active
- Network segmentation: implemented
- Internet access from `GAME-DMZ`: verified
- Pi-hole DNS from `GAME-DMZ`: verified
- `GAME-DMZ → INFRA`: blocked
- `GAME-DMZ → TRUSTED`: blocked
- Administration from `TRUSTED`: available
- Legacy game workload cleanup/migration: ongoing

The node should be treated as the long-term home for dedicated game-server workloads.

---

# Roadmap

Planned improvements include:

- Complete migration of remaining legacy game workloads
- Standardize game-server update procedures
- Standardize save-file backups
- Validate PBS recovery for the game VM
- Improve service-level monitoring
- Document each game server under `docs/services/game-servers.md`
- Remove obsolete game-server configuration from `pve`

---

# Public Repository Notes

This document intentionally excludes:

- passwords
- Playit.gg tunnel secrets
- API tokens
- private keys
- Tailscale addresses
- public endpoints
- MAC addresses
- server passwords
- private player/admin lists

Internal RFC1918 addresses are included where they explain the architecture.

---

# Scope of This Document

This file owns documentation for the **physical `pve-game-server` node**:

- hardware
- node role
- host networking
- VM placement
- failure domain
- migration role
- backup relationship
- maintenance considerations
- security architecture

It does not own:

- individual game-server installation instructions
- game ports
- player configuration
- Playit.gg secrets
- ACL definitions
- VLAN design
- detailed backup scripts

Those belong in the relevant service, network, security, and configuration documentation.

---

## Related Documentation

- [`../architecture/overview.md`](../architecture/overview.md) — overall architecture
- [`../architecture/physical-topology.md`](../architecture/physical-topology.md) — physical topology
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — workload relationships
- [`../network/vlan-design.md`](../network/vlan-design.md) — VLAN design
- [`../network/acl-policy.md`](../network/acl-policy.md) — GAME-DMZ policy
- [`../network/port-mapping.md`](../network/port-mapping.md) — host switch connection
- [`../services/game-servers.md`](../services/game-servers.md) — game-service documentation
- [`../security/backup-strategy.md`](../security/backup-strategy.md) — backup architecture
- [`pve-main.md`](pve-main.md) — main infrastructure node
- [`pve-elitedesk.md`](pve-elitedesk.md) — backup/support node
