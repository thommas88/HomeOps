# Immich

This document describes the Immich deployment used for self-hosted photo and video management in the homelab.

Immich runs as a dedicated virtual machine on the main Proxmox host and stores its media library on TrueNAS-backed NFS storage.

The storage dependency is important: **TrueNAS must be available before Immich can access its media path correctly.**

---

## Overview

| Item | Value |
|---|---|
| Service | Immich |
| Host | `pve-main` |
| VM ID | `106` |
| Network | `INFRA` / VLAN 10 |
| Address | `192.168.10.71` |
| CPU | 4 vCPU |
| Memory | 8 GiB |
| Boot disk | 80 GB |
| Machine type | `q35` |
| Network adapter | VirtIO |
| Media storage | TrueNAS NFS |
| Current documented Immich version | `v2.5.6` |

Immich provides the homelab's self-hosted photo and video library.

---

# Architecture

Immich is split across two systems:

```text
Immich application
        │
        │ NFS
        ▼
TrueNAS
        │
        ▼
immichpool
        │
        ▼
2 × 2 TB mirror
```

The application runs on:

```text
pve-main
└── VM 106 — Immich
```

The media storage is provided by:

```text
pve-main
└── VM 101 — TrueNAS
```

This makes TrueNAS a direct dependency of the Immich service.

---

## Logical Diagram

```mermaid
flowchart LR
    USER["Immich Client"]
    IMMICH["Immich VM 106"]
    NFS["NFS"]
    TRUENAS["TrueNAS VM 101"]
    POOL["immichpool"]
    DISKS["2 × 2 TB Mirror"]

    USER --> IMMICH
    IMMICH --> NFS
    NFS --> TRUENAS
    TRUENAS --> POOL
    POOL --> DISKS
```

---

# Proxmox Deployment

Immich runs as:

```text
VM 106
```

on:

```text
pve-main
```

Current VM resources:

| Resource | Configuration |
|---|---|
| CPU | 4 cores |
| Memory | 8 GiB |
| Boot disk | 80 GB |
| Machine type | `q35` |
| SCSI controller | VirtIO SCSI |
| Network adapter | VirtIO |

The VM boot disk contains the application environment.

The photo/video library is not stored primarily on the 80 GB virtual boot disk.

---

# Storage

Immich uses storage provided by TrueNAS.

The current TrueNAS NFS export is:

```text
/mnt/immichpool/immich-data
```

The underlying pool is:

```text
immichpool
```

Topology:

```text
2 × 2 TB mirror
```

Usable capacity:

```text
~1.76 TiB
```

At the time of documentation, Immich reported approximately:

```text
46.8 GiB used
```

The usage figure is only a point-in-time reference and should not be treated as static architecture.

---

# Storage Dependency

The most important operational dependency is:

```text
TrueNAS must be available before Immich expects its NFS-backed media path.
```

If Immich starts while the storage path is unavailable, the application can be running while its media storage is missing or mounted incorrectly.

Preferred startup order:

```text
1. pve-main
2. TrueNAS VM 101
3. ZFS pools online
4. NFS service available
5. Immich VM 106
```

This dependency should be considered during:

- host reboots
- Proxmox maintenance
- TrueNAS updates
- Immich updates
- restore operations
- troubleshooting

---

# Why Startup Order Matters

A VM being in the `running` state does not guarantee that its dependent storage is ready.

For Immich, the actual service path is:

```text
pve-main
 │
 ▼
TrueNAS
 │
 ▼
immichpool
 │
 ▼
NFS export
 │
 ▼
Immich
```

If TrueNAS is unavailable:

- Immich may still boot
- the application may still respond
- the media path may be unavailable
- uploads or media access may fail

This makes storage validation part of Immich health validation.

---

# TrueNAS Relationship

TrueNAS provides the persistent media storage.

The NFS path used by Immich belongs to:

```text
immichpool
```

which is configured as:

```text
2-disk mirror
```

This gives the Immich storage pool local disk redundancy against one drive failure.

Detailed storage architecture:

[`truenas.md`](truenas.md)

---

# Network

Immich resides on:

```text
INFRA / VLAN 10
```

Address:

```text
192.168.10.71
```

TrueNAS also resides on `INFRA`.

The storage path therefore remains inside the infrastructure network.

Administrative access originates from trusted management paths.

Immich is not documented as being directly exposed to the public Internet.

---

# DNS

Immich uses the homelab's redundant Pi-hole DNS infrastructure.

```text
Immich
   │
   ▼
Pi-hole 1 / Pi-hole 2
```

The two Pi-hole instances run on different physical Proxmox hosts.

Documentation:

[`pihole.md`](pihole.md)

---

# Service Dependencies

Immich depends on:

- `pve-main`
- VM 106
- `INFRA` networking
- TrueNAS VM 101
- `immichpool`
- TrueNAS NFS service
- DNS
- the Immich application stack inside the VM

A simplified dependency tree:

```text
pve-main
├── TrueNAS
│   └── immichpool
│       └── NFS
│
└── Immich
    └── consumes NFS storage
```

---

# Failure Scenarios

## Immich VM Failure

Affected:

- Immich application
- photo/video access through Immich

Not automatically affected:

- TrueNAS
- stored media
- other SMB/NFS shares

---

## TrueNAS Failure

Affected:

- Immich media storage
- NFS path
- other TrueNAS services

Immich may still appear online, but it should not be considered healthy until the storage path is restored.

---

## `immichpool` Failure

Affected:

- Immich media library storage

The pool is mirrored and can tolerate one member disk failure.

Complete pool loss would require recovery from available backups or other copies.

---

## `pve-main` Failure

Because both TrueNAS and Immich run on the same physical Proxmox host:

```text
pve-main failure
    │
    ├── TrueNAS offline
    └── Immich offline
```

This is a shared physical failure domain.

The separate PBS node can still remain available on `prodesk`.

---

## DNS Failure

One Pi-hole instance can fail without removing normal DNS service.

If both DNS services are unavailable, Immich functionality that depends on DNS resolution may be affected.

---

# Backup

Immich has two separate backup concerns:

```text
Immich VM
     and
Immich media data
```

These are not the same thing.

---

## Immich VM Backup

VM 106 is included in the Proxmox backup architecture.

PBS can protect:

- VM boot disk
- operating system
- application configuration stored inside the VM
- application environment

The backup target resides on:

```text
prodesk
```

Documentation:

[`proxmox-backup-server.md`](proxmox-backup-server.md)

---

## Immich Media Backup

The actual media library resides on:

```text
TrueNAS → immichpool
```

That data is **not automatically included in the Immich VM backup**.

Current limitation:

```text
Full secondary backup of the TrueNAS media dataset is not yet implemented.
```

The mirrored pool provides disk redundancy, but redundancy is not equivalent to backup.

---

# Recovery

A complete Immich recovery may require both:

```text
Immich VM recovery
        +
Immich storage recovery
```

The application cannot be considered fully recovered until its storage dependency is available.

---

## Preferred Recovery Order

```text
1. Recover / start pve-main
2. Recover / start TrueNAS
3. Verify immichpool
4. Verify NFS export
5. Start / restore Immich
6. Verify media path
7. Verify application
```

---

## Recovery Validation

After restoring Immich:

- VM boots normally
- expected IP address is present
- DNS works
- TrueNAS is reachable
- NFS path is mounted/available
- application starts
- existing photos/videos load
- new uploads can be written
- thumbnails/previews function
- no unexpected storage errors appear

A successful login alone is not sufficient validation.

---

# Updates

Immich is an actively developed application and updates can be relatively frequent.

The current documented server version is:

```text
v2.5.6
```

The application interface also indicated that a newer release was available at the time of documentation.

Version numbers in this file should therefore be treated as a **documented point in time**, not a permanently fixed architecture value.

---

## Update Workflow

Before a significant Immich update:

1. Verify TrueNAS is healthy.
2. Verify the NFS storage path.
3. Confirm a recent VM backup exists.
4. Review the Immich release/update requirements.
5. Apply the update.
6. Restart services if required.
7. Validate the media library.
8. Test a new upload.

---

# Monitoring

Immich should be monitored at multiple levels.

## Proxmox Level

Monitor:

- VM status
- CPU
- memory
- boot-disk usage
- network connectivity

## Application Level

Monitor:

- Immich server state
- application availability
- storage-space reporting
- media access
- upload behavior

## Storage Level

Monitor:

- TrueNAS availability
- `immichpool` health
- NFS availability
- pool capacity

The application is only fully healthy when all three layers are working.

---

# Security

Immich contains personal photos and videos and should therefore be treated as sensitive infrastructure.

Current design principles include:

- service hosted on `INFRA`
- storage hosted on `INFRA`
- administration from trusted systems
- no credentials committed to GitHub
- no public documentation of user accounts
- no publication of personal media content

---

# Public Repository Privacy

Screenshots from Immich require particular care.

A public repository should avoid showing:

- personal photos
- faces
- EXIF/location information
- albums with personal names
- user accounts
- authentication details
- API keys
- private sharing links

Screenshots showing only:

- service status
- generic storage statistics
- logo/version information

are considerably safer for public documentation.

---

# Design Decisions

## Dedicated VM

Immich runs in its own VM rather than sharing a general-purpose application server.

Advantages include:

- isolation
- dedicated resources
- simpler VM backup
- clear network identity
- easier troubleshooting

---

## Storage on TrueNAS

Media is stored outside the Immich boot disk.

This avoids tying the main media library to the lifecycle of the application VM.

It also allows ZFS to manage the storage separately from the application.

---

## Dedicated `immichpool`

Immich uses a dedicated mirrored ZFS pool.

This provides:

- clear capacity boundaries
- one-disk fault tolerance
- independent pool health
- simpler NFS mapping

---

## TrueNAS Before Immich

The startup dependency is intentional and documented.

TrueNAS should be brought online first so that the expected storage path exists before Immich starts using it.

This is more reliable than treating both VMs as completely independent workloads.

---

# Current State

Current confirmed state:

- Immich: active
- Host: `pve-main`
- VM ID: `106`
- Network: `INFRA`
- Address: `192.168.10.71`
- CPU: 4 vCPU
- Memory: 8 GiB
- Boot disk: 80 GB
- Current documented server version: `v2.5.6`
- Media storage: TrueNAS NFS
- TrueNAS pool: `immichpool`
- Pool layout: 2 × 2 TB mirror
- Approx. pool capacity: 1.76 TiB
- NFS dependency: active
- Startup order dependency: TrueNAS before Immich
- VM backup through PBS: available
- Full secondary media backup: not yet available

---

# Roadmap

Potential improvements include:

- Formalize VM startup ordering in Proxmox
- Add explicit NFS health checking before Immich startup
- Improve backup coverage for the media dataset
- Add offsite protection for irreplaceable photos/videos
- Periodically test full Immich recovery
- Monitor pool-growth trends
- Keep application updates documented

---

# Scope of This Document

This file owns Immich service documentation:

- Proxmox deployment
- VM resources
- network placement
- TrueNAS/NFS dependency
- startup order
- backup separation
- recovery
- maintenance
- security/privacy considerations

It does not own:

- TrueNAS pool internals beyond the Immich dependency
- full PBS configuration
- Immich user accounts
- application secrets
- personal photo inventory
- external sharing configuration

Those belong in the relevant service, node, security, or private configuration documentation.

---

## Related Documentation

- [`../nodes/pve-main.md`](../nodes/pve-main.md) — physical host
- [`truenas.md`](truenas.md) — storage provider
- [`proxmox-backup-server.md`](proxmox-backup-server.md) — VM backup
- [`pihole.md`](pihole.md) — DNS
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — service dependencies
- [`../security/backup-strategy.md`](../security/backup-strategy.md) — backup architecture
