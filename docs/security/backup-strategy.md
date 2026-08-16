# Backup Strategy

This document describes the overall backup, recovery, and data-protection strategy used across the homelab.

The backup design combines Proxmox Backup Server, application-level backups, storage redundancy, and service-specific recovery methods.

The goal is not to treat every copy or redundancy mechanism as equivalent. VM backup, application backup, ZFS redundancy, save backup, snapshots, and offsite protection solve different failure scenarios.

---

## Overview

The current protection model includes:

| Layer | Purpose | Current State |
|---|---|---|
| Proxmox Backup Server | VM/LXC backup and restore | Active |
| Application-level backups | Service-specific recovery | Used where applicable |
| Game-save backups | Protect game state before updates | Active |
| ZFS redundancy | Protect against selected disk failures | Active |
| ZFS scrubs | Detect storage errors/corruption | Active |
| Automated ZFS snapshots | Local rollback/recovery | Not yet implemented |
| Full NAS backup | Second copy of TrueNAS datasets | Not yet implemented |
| Offsite backup | Site-level protection | Not yet implemented |

The current strategy therefore provides good VM/LXC recovery coverage but does not yet provide complete protection for all bulk-storage data.

---

# Design Principles

The backup strategy is based on several principles:

```text
Redundancy ≠ Backup
Backup ≠ Restore Test
Local Backup ≠ Offsite Backup
VM Backup ≠ Application Data Backup
```

A system is considered meaningfully protected only when the relevant failure scenario has an appropriate recovery path.

---

# Proxmox Backup Server

The main VM/LXC backup platform is:

```text
Proxmox Backup Server
```

running as:

```text
CT 299
```

on:

```text
prodesk
```

Address:

```text
192.168.10.50
```

The active datastore is:

```text
PBS-local-new
```

and is stored on a dedicated:

```text
Samsung 990 PRO 2 TB NVMe
```

Detailed service documentation:

[`../services/proxmox-backup-server.md`](../services/proxmox-backup-server.md)

---

# PBS Storage Layout

The PBS datastore is separated from the Proxmox system storage.

Current layout:

```text
Samsung 990 PRO 2 TB
        │
        ▼
ext4
        │
        ▼
/mnt/pbs-datastore
        │
        │ bind mount
        ▼
PBS CT 299
        │
        ▼
/mnt/backup-local
        │
        ▼
PBS-local-new
```

This gives the backup service a dedicated local storage device.

---

# Backup Schedule

PBS backups run weekly on Sunday.

The current jobs are staggered across:

```text
01:00
02:00
03:00
04:00
```

One Proxmox node is scheduled at a time.

The purpose is to avoid having all source nodes start backup activity against PBS simultaneously.

The exact source-node-to-hour mapping is maintained in the active Proxmox configuration and is intentionally not duplicated here unless explicitly documented.

---

# Retention

The current PBS retention policy is:

```text
keep-last    = 3
keep-weekly  = 8
keep-monthly = 12
```

This provides:

- recent restore points
- medium-term weekly history
- longer-term monthly history

Retention must remain balanced against the available 2 TB datastore capacity.

---

# Proxmox Workload Coverage

PBS protects selected VMs and containers across the Proxmox environment.

The current Proxmox nodes are:

```text
pve-main
prodesk
elitedesk
pve-game-server
```

Backup selection should be reviewed whenever workloads are:

- created
- deleted
- migrated
- renumbered
- replaced

The backup configuration should follow the current workload placement rather than historical VM IDs.

---

# Failure-Domain Separation

PBS is physically hosted on:

```text
prodesk
```

This provides host-level separation for workloads running on:

```text
pve-main
elitedesk
pve-game-server
```

Conceptually:

```text
Source workload host
        │
        ▼
Network
        │
        ▼
PBS on ProDesk
```

If one of those source hosts fails, PBS can remain available for restore.

---

## ProDesk Limitation

Workloads hosted on `prodesk` share the same physical host as PBS.

For those workloads:

```text
Source workload
      +
PBS service
      +
PBS datastore
```

can be affected by the same physical ProDesk failure.

This is a known limitation.

PBS therefore provides strong VM/LXC backup functionality, but it does not provide complete host-level separation for workloads that are themselves hosted on ProDesk.

---

# VM and LXC Backup

PBS is appropriate for protecting:

- VM boot disks
- LXC filesystems
- operating-system configuration
- service configuration stored inside guests
- application environments
- infrastructure VMs
- selected lab systems
- game-server VMs

Typical examples include:

- Home Assistant
- Immich
- Omada Controller
- Pi-hole
- Homepage
- game-server workloads
- TrueNAS VM configuration
- Linux infrastructure VMs

---

# TrueNAS Backup Model

TrueNAS requires separate treatment because the VM and the stored data are not the same backup object.

```text
TrueNAS VM
      │
      ├── boot disk / VM config
      │       └── PBS
      │
      └── physical ZFS data disks
              └── not included in VM backup
```

TrueNAS runs as:

```text
VM 101
```

on:

```text
pve-main
```

Detailed service documentation:

[`../services/truenas.md`](../services/truenas.md)

---

## TrueNAS VM Backup

PBS can protect:

- TrueNAS boot disk
- VM configuration
- virtual hardware configuration
- operating-system state

This is useful for recovering the TrueNAS virtual appliance itself.

---

## TrueNAS Data Disks

The physical ZFS data disks are mapped separately to the TrueNAS VM.

They are configured with:

```text
backup=0
```

in the Proxmox VM configuration.

They are therefore not included in the normal VM backup.

This distinction is critical:

```text
TrueNAS VM backup successful
        ≠
NAS data fully backed up
```

---

# ZFS Storage Protection

Current TrueNAS storage protection includes:

| Pool | Layout | Disk-Failure Protection |
|---|---|---|
| `tank` | RAIDZ1, 4 × 8 TB | One disk |
| `immichpool` | Mirror, 2 × 2 TB | One disk |
| `pool_3tb` | Single disk | None |

ZFS redundancy protects against selected disk failures.

It does not replace backup.

---

# ZFS Scrubs

Scheduled ZFS scrubs are configured.

Scrubs help detect:

- checksum errors
- unreadable sectors
- latent corruption
- disk problems

This is part of storage-health protection rather than backup.

A scrub does not create another recoverable copy of data.

---

# ZFS Snapshots

Automated ZFS snapshots are currently:

```text
Not implemented
```

This is one of the main gaps in the current storage-protection model.

Snapshots could provide local recovery from scenarios such as:

- accidental file deletion
- unwanted changes
- selected logical corruption
- application mistakes

Snapshots would still remain on the same storage system and therefore would not replace a second backup copy.

---

# Full NAS Backup

The TrueNAS datasets do not currently have a complete second storage copy.

Current state:

```text
Full NAS backup: not implemented
```

The main limiting factor is capacity.

The 2 TB PBS datastore is not sized to maintain a complete second copy of the TrueNAS storage pools.

---

# Data-Priority Model

Because full duplication of all NAS data is not currently practical, future backup expansion should prioritize data based on recovery value.

Higher-priority examples include:

- personal photos
- important documents
- project files
- configuration
- irreplaceable data

Lower-priority examples may include:

- reproducible files
- downloaded software
- replaceable media

This allows limited backup capacity to be focused on the data with the highest recovery value.

---

# Immich

Immich has two separate protection requirements:

```text
Immich application VM
        +
Immich media data
```

The application runs as VM 106 and can be backed up through PBS.

The media library resides on:

```text
TrueNAS
└── immichpool
```

through NFS.

Therefore:

```text
Immich VM backup
        ≠
Immich media backup
```

A complete Immich recovery requires both the application environment and the underlying media storage to be available.

Detailed documentation:

[`../services/immich.md`](../services/immich.md)

---

# Home Assistant

Home Assistant uses two possible recovery layers:

```text
Home Assistant application backup
              +
Proxmox VM backup
```

The Proxmox backup protects the complete VM.

Home Assistant's own backup system can protect application-level configuration and state.

These mechanisms are complementary rather than redundant.

The exact application-level backup schedule is not currently documented as a fixed policy.

Detailed documentation:

[`../services/home-assistant.md`](../services/home-assistant.md)

---

# Omada Controller

Omada uses two recovery layers:

```text
Omada configuration backup
          +
PBS VM backup
```

Manual Omada configuration backups are created after significant network changes.

Examples include:

- VLAN changes
- Gateway ACL changes
- switch-port changes
- device adoption or replacement
- wireless configuration changes
- management-path changes

The PBS VM backup provides full-system recovery of the controller VM.

Detailed documentation:

[`../services/omada-controller.md`](../services/omada-controller.md)

---

# Pi-hole

The two Pi-hole containers provide service redundancy:

```text
Pi-hole 1 → prodesk
Pi-hole 2 → elitedesk
```

This protects DNS availability against failure of one Pi-hole or one host.

However:

```text
Service redundancy
        ≠
Backup
```

The containers can also be protected through PBS.

Pi-hole configuration can additionally be exported independently if a service-level recovery method is desired.

Detailed documentation:

[`../services/pihole.md`](../services/pihole.md)

---

# Game Servers

Game servers use a layered protection model.

```text
Game save backup
       +
VM backup through PBS
```

The automated update workflow follows:

```text
Stop service
      │
      ▼
Back up save
      │
      ▼
SteamCMD update
      │
      ▼
Start service
```

The save backup protects the most valuable state immediately before a potentially disruptive game update.

PBS protects the wider VM environment.

Detailed documentation:

[`../services/game-servers.md`](../services/game-servers.md)

---

# Backup vs. Redundancy

Several parts of the homelab provide redundancy.

Examples include:

```text
Pi-hole pair
RAIDZ1
ZFS mirror
```

These improve availability or disk-failure tolerance.

They are not substitutes for backup.

---

## Example: Pi-hole

```text
Pi-hole 1 failure
      │
      ▼
Pi-hole 2 remains available
```

This is service redundancy.

It does not provide historical recovery from:

- bad configuration
- accidental deletion
- compromise
- simultaneous failure

---

## Example: RAIDZ1

```text
One disk fails
      │
      ▼
Pool remains available
```

This is storage redundancy.

It does not protect against:

- accidental deletion
- ransomware
- complete pool loss
- host destruction
- site loss

---

# Restore Testing

A backup is only useful if it can be restored successfully.

Restore testing should include:

```text
Select backup
      │
      ▼
Restore VM / LXC
      │
      ▼
Boot system
      │
      ▼
Validate networking
      │
      ▼
Validate service
      │
      ▼
Validate dependencies
```

A VM reaching the `running` state is not sufficient proof of recovery.

---

# Service-Specific Recovery Validation

Different workloads require different validation.

## Home Assistant

Verify:

- VM boots
- network works
- USB Zigbee coordinator is available
- Zigbee2MQTT works
- IoT devices respond
- automations work

---

## Immich

Verify:

- VM boots
- NFS path is available
- existing media loads
- uploads work
- thumbnails/previews work

---

## Omada

Verify:

- controller starts
- all managed devices reconnect
- VLANs remain present
- Gateway ACLs remain present
- representative network paths work

---

## Game Servers

Verify:

- VM boots
- game services start
- save data is intact
- Playit.gg connectivity works
- external connection succeeds

---

# Recovery Order

Some services have dependency ordering.

For example:

```text
pve-main
   │
   ▼
TrueNAS
   │
   ▼
ZFS pools
   │
   ▼
NFS
   │
   ▼
Immich
```

Restoring or starting dependent services before storage is ready can create misleading partial-success states.

Recovery documentation should therefore account for service dependencies rather than only VM startup.

---

# Local Backup Limitation

PBS remains inside the main physical homelab site.

A site-wide incident may affect both:

```text
source workloads
      +
local PBS
```

Examples include:

- complete local power loss
- fire
- theft
- major hardware damage
- severe local network failure
- other site-level events

This is the largest remaining architectural limitation in the backup model.

---

# Offsite M900

The offsite Lenovo ThinkCentre M900 currently serves primarily as:

- Uptime Kuma monitoring
- Tailscale endpoint
- SSH/utility host

It does not currently contain:

- a complete PBS replica
- a complete TrueNAS backup
- a full second copy of all important data

The node could potentially be used later for selected offsite backup tasks.

Detailed node documentation:

[`../nodes/offsite-m900.md`](../nodes/offsite-m900.md)

---

# Offsite Backup Roadmap

The long-term backup design should add a second physical location for the most important data.

A future model could follow:

```text
Primary data
     │
     ▼
Local backup / redundancy
     │
     ▼
Selected offsite copy
```

The offsite copy does not need to include every replaceable file.

Priority should be given to data that cannot easily be recreated.

---

# Power-Loss Considerations

The homelab does not currently have UPS protection documented as part of the active architecture.

A complete power outage can therefore interrupt:

- Proxmox hosts
- PBS
- TrueNAS
- network infrastructure
- active backup jobs

The offsite monitoring node can help detect such an outage, but it does not prevent it.

UPS protection remains a useful future resilience improvement.

---

# Security of Backups

Backup infrastructure contains sensitive data.

A PBS backup may contain:

- operating systems
- application configuration
- credentials stored inside guest filesystems
- internal network information
- application data

The backup environment should therefore be treated as high-value infrastructure.

Current security principles include:

- PBS on `INFRA`
- no intentional direct public exposure
- administration from trusted paths
- backup credentials excluded from GitHub
- encryption/authentication secrets kept private
- restricted remote administration

---

# Public Repository Security

The repository may document:

- backup architecture
- retention policy
- internal RFC1918 addressing
- storage layout
- recovery procedures
- backup scope
- known limitations

Do not publish:

- PBS credentials
- API tokens
- encryption keys
- datastore secrets
- backup authentication material
- private recovery keys
- complete backup archives
- application backup files containing secrets

---

# Current Protection Matrix

| Component | VM/LXC Backup | Application Backup | Redundancy | Offsite |
|---|---|---|---|---|
| Home Assistant | PBS | Supported / schedule not fixed here | — | No |
| Immich application | PBS | Application-specific | — | No |
| Immich media | No full PBS coverage | — | ZFS mirror | No |
| Omada Controller | PBS | Manual config backup | — | No |
| Pi-hole | PBS capable | Config export possible | Dual instances | No |
| Game servers | PBS | Save backups | — | No |
| TrueNAS VM | PBS | Config export possible | — | No |
| `tank` data | No full second copy | — | RAIDZ1 | No |
| `immichpool` data | No full second copy | — | Mirror | No |
| `pool_3tb` data | No full second copy | — | None | No |

This matrix describes the current protection model at a high level.

---

# Current State

Current confirmed state:

- PBS: active
- PBS host: `prodesk`
- PBS container: `299`
- PBS address: `192.168.10.50`
- PBS datastore: `PBS-local-new`
- Datastore device: Samsung 990 PRO 2 TB
- Datastore filesystem: ext4
- Backup schedule: Sunday at 01:00, 02:00, 03:00, and 04:00
- Jobs staggered one source node at a time
- Retention: keep-last 3 / keep-weekly 8 / keep-monthly 12
- TrueNAS VM backup: available
- TrueNAS physical ZFS data disks: excluded from VM backup
- ZFS scrubs: configured
- Automated ZFS snapshots: not configured
- Game-save backup before updates: active
- Omada manual configuration backup after major changes: in use
- Full NAS backup: not implemented
- Full offsite backup: not implemented
- Restore testing: recommended / ongoing roadmap item
- UPS protection: not currently part of the documented active architecture

---

# Roadmap

Priority improvements include:

1. Configure automated ZFS snapshots.
2. Define snapshot retention per important dataset.
3. Perform regular PBS restore tests.
4. Prioritize irreplaceable data for secondary backup.
5. Add selected offsite backup coverage.
6. Document disaster-recovery procedures for major host failures.
7. Periodically review PBS capacity and retention.
8. Verify backup selection after workload migrations.
9. Consider UPS protection for core infrastructure.
10. Periodically validate application-level recovery procedures.

---

# Scope of This Document

This file owns the high-level backup and recovery strategy:

- PBS architecture
- backup scheduling
- retention
- failure-domain separation
- VM/LXC protection
- TrueNAS backup limitations
- ZFS redundancy and snapshots
- application-level backups
- game-save backups
- restore testing
- offsite strategy
- backup security
- known limitations

It does not own:

- full PBS configuration
- exact backup-job definitions
- TrueNAS pool administration
- service-specific update scripts
- application credentials
- backup authentication secrets
- private recovery material

Those belong in the relevant service, node, configuration, or private documentation.

---

## Related Documentation

- [`../services/proxmox-backup-server.md`](../services/proxmox-backup-server.md) — PBS deployment
- [`../services/truenas.md`](../services/truenas.md) — ZFS storage
- [`../services/immich.md`](../services/immich.md) — media backup dependency
- [`../services/home-assistant.md`](../services/home-assistant.md) — Home Assistant recovery
- [`../services/omada-controller.md`](../services/omada-controller.md) — controller backups
- [`../services/pihole.md`](../services/pihole.md) — DNS redundancy
- [`../services/game-servers.md`](../services/game-servers.md) — game-save and VM backup
- [`../nodes/pve-prodesk.md`](../nodes/pve-prodesk.md) — PBS host
- [`../nodes/offsite-m900.md`](../nodes/offsite-m900.md) — offsite node
- [`segmentation.md`](segmentation.md) — security segmentation
- [`remote-access.md`](remote-access.md) — private remote administration
