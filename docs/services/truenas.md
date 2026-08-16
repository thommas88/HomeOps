# TrueNAS

This document describes the TrueNAS storage service used by the homelab.

TrueNAS provides the primary bulk-storage platform for project files, documents, media, software, general storage, and Immich data.

It runs as **VM 101** on the main Proxmox host (`pve-main`). The physical HDDs are connected to the Proxmox host through an LSI 9207-8i HBA in IT mode and are presented individually to the TrueNAS VM as raw disks using stable `/dev/disk/by-id` mappings.

---

## Overview

| Item | Value |
|---|---|
| Platform | TrueNAS SCALE |
| Version | `25.04.2.3` |
| Host | `pve-main` |
| VM ID | `101` |
| Network | `INFRA` / VLAN 10 |
| Address | `192.168.10.40` |
| Storage model | ZFS |
| Primary protocols | SMB and NFS |
| Main storage pools | `tank`, `immichpool`, `pool_3tb` |

TrueNAS is one of the most important services in the homelab because several other services depend on the storage it provides.

---

# VM Configuration

TrueNAS runs as a QEMU/KVM virtual machine on Proxmox.

Current VM resources:

| Resource | Configuration |
|---|---|
| vCPU | 4 cores |
| Memory | 20 GiB |
| Boot disk | 40 GB virtual disk |
| Machine type | `q35` |
| Network adapter | VirtIO |
| Host | `pve-main` |

The virtual boot disk contains the TrueNAS operating system.

The ZFS data disks are separate physical HDDs presented individually to the VM.

---

# Storage Architecture

The physical disks are connected to the main Proxmox host through an:

```text
LSI 9207-8i HBA
```

running in:

```text
IT mode
```

The important distinction is that the **HBA itself is not passed through as a PCI device to TrueNAS**.

Instead, Proxmox presents the individual physical disks to VM 101 using persistent `/dev/disk/by-id` references.

Conceptually:

```text
Physical HDDs
     │
     ▼
LSI 9207-8i HBA
     │
     ▼
Proxmox host — pve-main
     │
     │ raw-disk mapping by /dev/disk/by-id
     ▼
TrueNAS VM 101
     │
     ▼
ZFS pools
```

This gives TrueNAS direct use of the individual disks while Proxmox remains responsible for attaching those block devices to the VM.

---

## Why Use `/dev/disk/by-id`

Persistent disk identifiers are used instead of volatile Linux device names such as:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

Device letters can change between boots or after hardware changes.

`/dev/disk/by-id` provides a stable mapping between a physical disk and the TrueNAS VM configuration.

Serial numbers and complete device IDs are intentionally excluded from this public repository.

---

# Physical Disk Layout

The TrueNAS VM currently receives:

- 4 × 8 TB HDD
- 2 × 2 TB HDD
- 1 × 3 TB HDD

These disks form three separate ZFS pools.

| Pool | Layout | Disks | Approx. Usable Capacity |
|---|---|---|---:|
| `tank` | RAIDZ1 | 4 × 8 TB | 21.01 TiB |
| `immichpool` | Mirror | 2 × 2 TB | 1.76 TiB |
| `pool_3tb` | Single disk | 1 × 3 TB | 2.63 TiB |

All three pools are currently online.

---

# `tank`

`tank` is the primary bulk-storage pool.

Topology:

```text
RAIDZ1
4 × 8 TB HDD
```

Current usable capacity:

```text
~21.01 TiB
```

The pool contains the majority of general-purpose storage.

Current high-level datasets/shares include:

- `Projects`
- `backups`
- `documents`
- `media`
- `software`

---

## RAIDZ1 Characteristics

The four-disk RAIDZ1 layout provides:

- single-disk fault tolerance
- ZFS checksumming
- scrub support
- better usable capacity than a mirrored four-disk layout

It should not be treated as a backup.

RAIDZ1 does not protect against:

- accidental deletion
- malicious deletion
- ransomware
- complete host loss
- site loss
- multiple disk failures beyond parity tolerance

---

# `immichpool`

`immichpool` is a dedicated storage pool for Immich.

Topology:

```text
Mirror
2 × 2 TB HDD
```

Current usable capacity:

```text
~1.76 TiB
```

The pool hosts the NFS-backed Immich data path.

Current NFS export:

```text
/mnt/immichpool/immich-data
```

The NFS service is currently running.

---

## Why a Dedicated Immich Pool?

The separate pool provides a clear storage boundary for Immich.

Advantages include:

- dedicated capacity tracking
- simple NFS dependency
- separation from general bulk storage
- mirror redundancy
- easier future migration or replacement

The two-disk mirror can tolerate failure of one member disk.

---

# `pool_3tb`

`pool_3tb` is a separate single-disk pool.

Topology:

```text
1 × 3 TB HDD
```

Current usable capacity:

```text
~2.63 TiB
```

Primary use:

```text
WorkDrive
```

Current SMB path:

```text
/mnt/pool_3tb/WorkDrive
```

Because this pool consists of one physical disk, it has no local disk-level redundancy.

Data stored only on this pool should therefore be treated accordingly.

---

# SMB

SMB is enabled and currently running.

Current SMB shares:

| Share | Path | Purpose |
|---|---|---|
| `Projects` | `/mnt/tank/Projects` | Project storage |
| `WorkDrive` | `/mnt/pool_3tb/WorkDrive` | General work storage |
| `backups` | `/mnt/tank/backups` | General backup storage |
| `documents` | `/mnt/tank/documents` | Documents |
| `media` | `/mnt/tank/media` | Media |
| `software` | `/mnt/tank/software` | Software and installation files |

SMB is used for normal file access from trusted systems.

User accounts, permissions, passwords, and share credentials are intentionally excluded from the public repository.

---

# NFS

NFS is enabled and currently running.

The current documented NFS share is:

```text
/mnt/immichpool/immich-data
```

Purpose:

```text
Immich storage
```

Logical dependency:

```text
Immich VM
    │
    │ NFS
    ▼
TrueNAS
    │
    ▼
immichpool
```

This is one of the most important service dependencies in the homelab.

---

# Immich Dependency

Immich relies on TrueNAS for its media storage.

```mermaid
flowchart LR
    IMMICH["Immich VM"]
    NFS["NFS"]
    TRUENAS["TrueNAS VM 101"]
    POOL["immichpool"]
    DISKS["2 × 2 TB Mirror"]

    IMMICH --> NFS
    NFS --> TRUENAS
    TRUENAS --> POOL
    POOL --> DISKS
```

This means:

- the Immich VM may be running while storage is unavailable
- VM status alone does not represent application health
- TrueNAS and NFS must be validated when troubleshooting Immich
- network changes affecting the NFS path can also affect Immich

Detailed Immich documentation belongs in:

[`immich.md`](immich.md)

---

# iSCSI

iSCSI is not currently used.

Current state:

```text
iSCSI: stopped
Targets: none
```

There is currently no requirement for block-storage sharing through iSCSI.

---

# ZFS Scrubs

Scheduled scrub tasks are configured.

The storage dashboard reports scheduled scrub tasks for the active pools.

Scrubs allow ZFS to periodically read stored data and verify it against checksums.

This helps detect:

- unreadable sectors
- checksum errors
- latent data corruption

Scrub completion and ZFS error counts should be checked as part of normal maintenance.

---

# Snapshots

Automated ZFS snapshots are **not currently configured**.

Current state:

```text
Automated snapshots: not implemented
```

This is a known gap in the current data-protection model.

Snapshots would provide a useful local recovery layer for:

- accidental deletion
- unwanted file modification
- application mistakes
- selected logical corruption scenarios

Snapshots are not a replacement for backup, but they significantly improve local recoverability.

---

# Backup Strategy

TrueNAS has two very different backup concerns:

```text
TrueNAS VM
      and
Stored ZFS data
```

They must not be treated as the same thing.

---

## TrueNAS VM Backup

VM 101 can be backed up through Proxmox Backup Server.

The VM backup can protect:

- the 40 GB TrueNAS boot disk
- VM configuration
- operating-system state
- virtual hardware configuration

The physical ZFS data disks are configured with:

```text
backup=0
```

in the Proxmox VM configuration.

They are therefore not included in the normal Proxmox VM backup.

---

## ZFS Data Backup

The data stored in:

- `tank`
- `immichpool`
- `pool_3tb`

does not currently have a complete second NAS copy.

The main reason is storage capacity.

The ProDesk/PBS node does not have enough storage to maintain another full copy of the TrueNAS datasets.

Current state:

```text
Full NAS backup: not implemented
```

This is a known limitation and an important roadmap item.

---

# Current Protection Layers

| Layer | Current Protection |
|---|---|
| TrueNAS boot/VM | Proxmox/PBS backup possible |
| `tank` | RAIDZ1 |
| `immichpool` | 2-disk mirror |
| `pool_3tb` | Single disk |
| ZFS scrubs | Configured |
| Automated snapshots | Not configured |
| Full second NAS | Not available |
| Full offsite NAS backup | Not available |

---

## RAID Is Not Backup

The ZFS layouts provide resilience against some physical disk failures.

They do not provide complete data protection.

For example:

```text
RAIDZ1
   │
   └── protects against one disk failure
```

does not protect against:

```text
rm -rf
ransomware
accidental overwrite
complete host loss
site loss
multiple drive failures
```

Backup strategy therefore has to exist independently of storage redundancy.

---

# Data Protection Priorities

Because full duplication of all NAS data is not currently practical, future backup planning should prioritize data by recovery value.

### High Priority

Examples:

- personal photos
- important documents
- important project files
- irreplaceable configuration

### Lower Priority

Examples may include:

- downloaded software
- reproducible files
- replaceable media

This allows limited backup capacity to protect the most valuable data first.

---

# Networking

TrueNAS resides on:

```text
INFRA / VLAN 10
```

Address:

```text
192.168.10.40
```

Administration is performed from `TRUSTED`.

The TrueNAS management interface is not intentionally exposed directly to the public Internet.

Detailed addressing and segmentation are documented under:

[`../network/`](../network/)

---

# Availability Dependencies

TrueNAS depends on several infrastructure layers:

```text
Main physical server
        │
        ▼
Proxmox VE
        │
        ▼
VM 101
        │
        ▼
Raw physical disk mappings
        │
        ▼
ZFS pools
        │
        ├── SMB
        └── NFS
```

A failure at any of these layers can affect storage availability.

---

# Failure Impact

A TrueNAS outage can affect:

- SMB file access
- Projects
- Documents
- Media
- Software storage
- WorkDrive
- general backup shares
- Immich media storage
- NFS-dependent workloads

A failure of VM 101 does not necessarily imply that the physical ZFS disks have failed.

A complete failure of `pve-main`, however, also removes access to the TrueNAS VM and its disk mappings until the host is recovered.

---

## Pool Failure Characteristics

### `tank`

RAIDZ1 can tolerate one failed member disk.

A second disk failure before recovery can result in pool loss.

### `immichpool`

The mirror can tolerate one member disk failure.

### `pool_3tb`

The pool has no disk redundancy.

Loss of the physical disk can result in loss of the pool.

---

# Maintenance

Normal TrueNAS maintenance includes:

- TrueNAS updates
- ZFS pool-health checks
- scrub verification
- SMART checks
- disk-temperature checks
- capacity monitoring
- SMB/NFS validation
- TrueNAS configuration backup/export
- future snapshot validation

---

## Pre-Update Checklist

Before updating TrueNAS:

1. Verify that all pools are online.
2. Check for ZFS errors.
3. Confirm that no important storage operation is in progress.
4. Verify access to the Proxmox host.
5. Confirm that the VM configuration/boot disk has current backup coverage.
6. Record the currently installed TrueNAS version.
7. Apply the update.
8. Reboot if required.

---

## Post-Update Validation

After maintenance:

- TrueNAS VM boots normally
- `tank` imports and reports online
- `immichpool` imports and reports online
- `pool_3tb` imports and reports online
- SMB is running
- NFS is running
- shares are visible
- Immich can access its NFS storage
- no new ZFS errors are present
- management access works normally

A successful VM boot alone is not sufficient validation.

---

# Monitoring

TrueNAS should be monitored at several levels.

## VM Level

Monitor:

- VM state
- CPU usage
- memory usage
- network availability

## Storage Level

Monitor:

- pool status
- ZFS errors
- disk health
- disk temperatures
- available capacity
- scrub results

## Service Level

Monitor:

- SMB availability
- NFS availability
- Immich storage access

The historical CPU/RAM/network graphs shown in Proxmox are useful operationally but are not included in this static GitHub documentation because they represent point-in-time utilization rather than architecture.

---

# Security

TrueNAS is treated as internal infrastructure.

Current design principles include:

- management on `INFRA`
- administration from `TRUSTED`
- no intentional direct Internet exposure
- SMB/NFS available only where required
- credentials excluded from GitHub
- no public storage-user inventory

The public documentation intentionally avoids:

- usernames
- passwords
- SMB credentials
- API tokens
- serial numbers
- raw disk IDs containing serial information

---

# Design Decisions

## Virtualized TrueNAS

TrueNAS runs as a VM rather than on a separate physical NAS server.

Advantages:

- efficient hardware utilization
- centralized Proxmox management
- no separate NAS chassis required
- VM-level backup of the operating environment
- flexible resource allocation

Trade-off:

- TrueNAS availability depends on `pve-main`

---

## Raw Physical Disk Mapping

The storage disks are attached individually to VM 101 using stable disk-by-ID paths.

This was chosen instead of virtual Proxmox disks for the ZFS data pools.

It preserves a direct relationship between TrueNAS and the physical storage devices while still keeping the HBA under the Proxmox host.

---

## LSI HBA in IT Mode

The disks are connected through an LSI 9207-8i HBA running in IT mode.

IT mode avoids hardware RAID abstraction and exposes the disks individually to the host.

ZFS therefore remains responsible for:

- redundancy
- checksumming
- pool health
- disk layout

---

## Separate Immich Pool

Immich has a dedicated mirrored pool.

This gives the service:

- clear storage isolation
- predictable capacity
- local disk redundancy
- a simple NFS dependency

---

## Separate WorkDrive Pool

The 3 TB WorkDrive is managed by TrueNAS as its own pool.

This allows it to use the same SMB/storage management platform while remaining separate from the main `tank` pool.

Because the pool has no redundancy, important data placed there should exist elsewhere if loss would be unacceptable.

---

# Current State

Current confirmed state:

- TrueNAS SCALE `25.04.2.3`: active
- VM 101 on `pve-main`: active
- 4 vCPU: configured
- 20 GiB RAM: configured
- 40 GB boot disk: configured
- Physical HDDs mapped individually to the VM: active
- `tank`: online
- `immichpool`: online
- `pool_3tb`: online
- SMB: running
- NFS: running
- iSCSI: unused/stopped
- Scheduled ZFS scrubs: configured
- Automated snapshots: not configured
- Full NAS backup: not available
- Immich storage dependency: active

---

# Roadmap

Planned improvements include:

- Configure automated ZFS snapshots
- Define snapshot retention by dataset
- Improve backup coverage for irreplaceable data
- Develop an offsite backup strategy
- Test file and dataset restores
- Continue monitoring pool capacity
- Review protection requirements for `pool_3tb`
- Document recovery of the raw-disk mappings if the Proxmox host is rebuilt

---

# Public Repository Notes

This document intentionally excludes:

- TrueNAS usernames
- passwords
- SMB credentials
- API keys
- private keys
- MAC addresses
- drive serial numbers
- full `/dev/disk/by-id` identifiers
- public/WAN addresses
- authentication configuration

Internal RFC1918 addressing and generic dataset paths are retained because they provide architectural and operational value.

---

# Scope of This Document

This file owns the TrueNAS service documentation:

- VM configuration
- physical disk presentation
- storage architecture
- ZFS pools
- SMB/NFS shares
- Immich storage dependency
- backup limitations
- snapshots
- scrubs
- maintenance
- failure characteristics

It does not own:

- physical `pve-main` hardware documentation
- VLAN design
- ACL policy
- Immich application configuration
- credentials
- user/share permissions

Those belong in the corresponding node, network, security, and service documentation.

---

## Related Documentation

- [`../nodes/pve-main.md`](../nodes/pve-main.md) — physical Proxmox host
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — service dependencies
- [`../network/ip-plan.md`](../network/ip-plan.md) — addressing
- [`../security/backup-strategy.md`](../security/backup-strategy.md) — overall backup architecture
- [`immich.md`](immich.md) — Immich service
- [`proxmox-backup-server.md`](proxmox-backup-server.md) — VM/LXC backup platform
