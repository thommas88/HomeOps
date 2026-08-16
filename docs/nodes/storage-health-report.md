# Homelab Storage Health & Drive Lifetime Report

**Environment:** Server 2.0 Homelab  
**Assessment date:** 2026-08-16  
**Scope:** Physical storage layout, SMART health indicators, endurance, and operational risk across the homelab nodes.

---

## Purpose

This document provides a hardware-level overview of storage devices used across the homelab. The goal is to maintain a repeatable baseline for:

- physical disk inventory
- operating-system and VM/LXC storage placement
- SSD/NVMe endurance tracking
- HDD SMART health monitoring
- identification of aging or higher-risk devices
- informed replacement and backup planning

The report is intended both as internal operational documentation and as portfolio material demonstrating practical infrastructure monitoring, lifecycle management, and risk assessment.

> **Note:** SMART data is useful for identifying degradation and wear, but it cannot predict every drive failure. Risk levels in this document combine SMART indicators, accumulated power-on time, endurance information, and the operational role of each device.

---

# Risk Classification

| Risk level | Meaning |
|---|---|
| **Very Low** | New or lightly used device with no reported wear or SMART concerns. |
| **Low** | Healthy device with normal operating history and no relevant SMART warnings. |
| **Moderate** | Significant runtime, write volume, or age warrants closer monitoring, but no active failure indicators are present. |
| **Elevated** | SMART shows evidence of media degradation or the device has both high runtime and relevant error indicators. |
| **High** | Active SMART warnings, pending/uncorrectable sectors, critical NVMe warnings, or other evidence of ongoing failure. |

---

# 1. PVE-main

## Role

Primary Proxmox VE hypervisor and the main compute/storage node in the homelab.

## Physical Drive Inventory

| Device | Model | Capacity | Power-On Hours | Approx. Powered-On Time | SMART / Endurance | Risk |
|---|---|---:|---:|---:|---|---|
| `/dev/sda` | Seagate IronWolf ST8000VN004-3CP101 | 8 TB | 6,271 h | 0.72 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Very Low** |
| `/dev/sdb` | Seagate IronWolf ST8000VN004-3CP101 | 8 TB | 3,183 h | 0.36 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Very Low** |
| `/dev/sdc` | Seagate IronWolf ST8000VN004-3CP101 | 8 TB | 3,183 h | 0.36 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Very Low** |
| `/dev/sdd` | Seagate IronWolf ST8000VN004-3CP101 | 8 TB | 6,271 h | 0.72 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Very Low** |
| `/dev/sde` | WD Caviar Black WD2002FAEX-007BA0 | 2 TB | 75,030 h | 8.56 years | 2 reallocated sectors, 1 reallocation event, no pending sectors | **Elevated** |
| `/dev/sdf` | WD Green WD20EARS-00MVWB0 | 2 TB | 52,574 h | 6.00 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Moderate** |
| `/dev/sdg` | Samsung SSD 850 EVO 250GB | 250 GB | 36,061 h | 4.12 years | SMART passed, Wear Leveling normalized value 86, no uncorrectable errors | **Low–Moderate** |
| `/dev/sdh` | WD Green WD30EZRX-00D8PB0 | 3 TB | 61,195 h | 6.99 years | SMART passed, no reallocated/pending/uncorrectable sectors | **Moderate** |
| `/dev/nvme0n1` | Samsung SSD 970 EVO Plus 1TB | 1 TB | 25,875 h | 2.95 years | 19% used, 367 TB written, Critical Warning `0x00` | **Low** |

## System and VM Storage Layout

The Samsung 970 EVO Plus is used for both the Proxmox operating system and local VM/LXC storage.

```text
Samsung SSD 970 EVO Plus 1 TB
│
├── EFI / boot
│
└── Proxmox LVM
    ├── pve-root       ~96 GB
    │   └── Proxmox VE operating system
    ├── pve-swap         8 GB
    └── pve-data       ~794 GB
        └── local-lvm VM/LXC storage
```

Observed virtual disks on `local-lvm` include workloads for VM/CT IDs 100, 101, 106, and the 550-series Kubernetes lab nodes.

## Proxmox Storage Summary

| Storage | Type | Role | Approx. Size |
|---|---|---|---:|
| `local` | Directory | Root/system storage | ~94 GB |
| `local-lvm` | LVM thin | VM/LXC disks | ~794 GB |
| `PBS-local-new` | PBS | Backup target | Remote/external to the local NVMe |
| `immich_library` | CIFS | Immich-related network storage | Disabled at assessment time |

## Risk Assessment

### WD Caviar Black 2 TB — `/dev/sde`

This is currently the highest-priority drive in the homelab storage watchlist.

```text
Power-On Hours:           75,030
Reallocated Sectors:      2
Reallocation Events:      1
Current Pending Sectors:  0
Offline Uncorrectable:    0
UDMA CRC Errors:          0
SMART Overall:            PASSED
```

The drive has successfully remapped two sectors and currently reports no pending or uncorrectable sectors. It is not in an active SMART failure state, but the combination of more than 75,000 operating hours and media reallocation warrants closer monitoring.

**Assessment: Elevated risk — monitor closely and avoid treating the device as the only copy of important data.**

### WD Green 2 TB and 3 TB

Both devices have substantial accumulated runtime but currently show clean error-related SMART indicators.

**Assessment: Moderate risk due to age/runtime, not due to active SMART degradation.**

### Samsung 970 EVO Plus 1 TB

The NVMe device reports:

```text
Power-On Hours:   25,875
Data Written:     367 TB
Percentage Used:  19%
Critical Warning: 0x00
SMART Overall:    PASSED
```

The endurance indicator suggests approximately 19% of the drive's reported endurance has been consumed.

The larger operational concern is the **failure domain**: the same physical NVMe hosts both the Proxmox operating system and local VM/LXC storage. A single-device failure therefore affects both the hypervisor installation and its locally stored workloads.

**Assessment: Low media-wear risk, but important operational dependency.**

---

# 2. ProDesk

## Role

Proxmox VE service node and Proxmox Backup Server host.

## Physical Drive Inventory

| Device | Model | Capacity | Power-On Hours | Approx. Powered-On Time | Writes | Endurance | Risk |
|---|---|---:|---:|---:|---:|---:|---|
| `/dev/nvme0n1` | WD_BLACK SN7100 1TB | 1 TB | 19 h | <1 day | 482 GB | 0% used | **Very Low** |
| `/dev/nvme1n1` | Samsung SSD 990 PRO 2TB | 2 TB | 4,527 h | 0.52 years | 2.12 TB | 0% used | **Very Low** |

Both NVMe devices reported:

```text
SMART overall-health: PASSED
Critical Warning:     0x00
```

## Storage Layout

```text
ProDesk
│
├── WD_BLACK SN7100 1 TB
│   ├── EFI / boot
│   ├── pve-root       ~96 GB
│   ├── pve-swap         8 GB
│   └── pve-data       ~794 GB
│       └── VM/LXC storage
│
└── Samsung 990 PRO 2 TB
    └── /mnt/pbs-datastore
        └── Proxmox Backup Server datastore
```

Observed local workloads include VM/CT IDs 200, 201, 202, 203, and 299.

## Risk Assessment

The ProDesk storage architecture separates the hypervisor/VM workload disk from the backup datastore. This reduces contention and separates two important storage roles across independent physical devices.

Both drives currently report negligible endurance consumption and no NVMe critical warnings.

**Assessment: Very Low risk.**

---

# 3. EliteDesk

## Role

Secondary Proxmox VE node used for lightweight services and redundancy.

## Physical Drive Inventory

| Device | Model | Capacity | Power-On Hours | Approx. Powered-On Time | Writes | Endurance | Risk |
|---|---|---:|---:|---:|---:|---:|---|
| `/dev/nvme0n1` | Kioxia KBG40ZNV256G | 256 GB | 179 h | 7.5 days | 1.20 TB | 0% used | **Very Low** |

SMART status:

```text
SMART overall-health: PASSED
Critical Warning:     0x00
Percentage Used:      0%
```

## Storage Layout

```text
Kioxia KBG40ZNV256G 256 GB
│
├── EFI / boot
├── pve-root       ~69 GB
├── pve-swap         8 GB
└── pve-data       ~141 GB
    └── VM/LXC storage
        └── VM/CT 301
```

## Risk Assessment

The device has very low accumulated operating time and reports no endurance consumption or critical warnings. The main limitation is capacity rather than device health.

**Assessment: Very Low risk.**

---

# 4. PVE-game-server

## Role

Dedicated Proxmox VE node for game-server workloads.

## Physical Drive Inventory

| Device | Model | Capacity | Power-On Hours | Approx. Powered-On Time | SMART | Risk |
|---|---|---:|---:|---:|---|---|
| `/dev/sda` | Samsung SSD 860 EVO 1TB | 1 TB | 47,044 h | 5.37 years | Passed | **Moderate** |

Relevant SMART attributes:

```text
SMART Overall:              PASSED
Reallocated Sector Count:   0
Uncorrectable Error Count:  0
Wear Leveling normalized:   97
```

## Storage Layout

```text
Samsung SSD 860 EVO 1 TB
│
├── EFI / boot
├── pve-root       100 GB
├── pve-swap         8 GB
└── pve-data       ~790 GB
    └── local-lvm
        └── VM 400
            └── 300 GB virtual disk
```

## Risk Assessment

The SSD has accumulated more than 47,000 power-on hours, making it one of the longest-running SSDs in the environment.

However, the current SMART data remains healthy:

- no reallocated sectors
- no uncorrectable errors
- healthy wear-leveling value
- SMART overall test passed

**Assessment: Moderate risk due to accumulated runtime, with no current evidence of active media failure. Continue monitoring rather than replacing solely based on age.**

---

# 5. ThinkCentre M900 — Offsite Node

## Role

Offsite Debian node used for external monitoring, remote access, and offsite storage.

## Physical Drive Inventory

| Device | Model | Capacity | Power-On Hours | Approx. Powered-On Time | SMART / Endurance | Risk |
|---|---|---:|---:|---:|---|---|
| `/dev/nvme0n1` | WD_BLACK SN850X 1000GB | 1 TB | 148 h | 6.2 days | 0% used, 322 GB written, Critical Warning `0x00` | **Very Low** |
| `/dev/sda` | Seagate Barracuda ST2000LM015-2E8174 | 2 TB | 3,258 h | 0.37 years | 0 reallocated/pending/uncorrectable sectors, 0 CRC errors | **Very Low** |

## Storage Layout

```text
ThinkCentre M900
│
├── WD_BLACK SN850X 1 TB
│   ├── EFI
│   ├── Debian root filesystem
│   └── Swap
│
└── Seagate Barracuda 2 TB
    └── /srv/offsite
        └── Offsite storage
```

## SMART / Endurance Details

### WD_BLACK SN850X 1 TB

```text
Power-On Hours:     148
Data Written:       322 GB
Percentage Used:    0%
Critical Warning:   0x00
```

The system NVMe is effectively new from an endurance perspective. No critical NVMe warning is reported and the endurance counter remains at 0%.

**Assessment: Very Low risk.**

### Seagate Barracuda 2 TB

```text
Power-On Hours:             3,258
Reallocated Sector Count:   0
Reported Uncorrectable:     0
Current Pending Sectors:    0
Offline Uncorrectable:      0
UDMA CRC Errors:            0
```

The offsite HDD currently shows clean error-related SMART counters and relatively low accumulated runtime.

The filtered output did not include the overall SMART self-assessment result, so this assessment is based on the captured attributes above.

**Assessment: Very Low risk based on the available SMART indicators.**

---


# Overall Storage Risk Summary

## Highest-Priority Watchlist

| Priority | Node | Device | Reason |
|---:|---|---|---|
| **1** | PVE-main | WD Caviar Black 2 TB | 75,030 h and 2 reallocated sectors |
| **2** | PVE-main | WD Green 3 TB | 61,195 h; very high runtime |
| **3** | PVE-main | WD Green 2 TB | 52,574 h; very high runtime |
| **4** | PVE-game-server | Samsung 860 EVO 1 TB | 47,044 h; healthy SMART but high runtime |
| **5** | PVE-main | Samsung 850 EVO 250 GB | 36,061 h; older SSD with measurable NAND wear |

## Healthy / Low-Risk Devices

The following currently show no meaningful health concerns based on the collected data:

- 4 × Seagate IronWolf 8 TB on PVE-main
- Samsung 970 EVO Plus 1 TB on PVE-main
- WD_BLACK SN7100 1 TB on ProDesk
- Samsung 990 PRO 2 TB on ProDesk
- Kioxia KBG40ZNV256G 256 GB on EliteDesk
- WD_BLACK SN850X 1 TB on ThinkCentre M900
- Seagate Barracuda 2 TB on ThinkCentre M900

---

# Operational Findings

## 1. PVE-main has a shared OS and VM failure domain

The Samsung 970 EVO Plus hosts both Proxmox VE itself and local VM/LXC storage. This is operationally simple, but a failure of the NVMe affects both layers simultaneously.

Backups are therefore especially important for workloads stored on `local-lvm`.

## 2. ProDesk separates compute storage from backup storage

ProDesk uses the WD_BLACK SN7100 for Proxmox and VM/LXC workloads, while the Samsung 990 PRO is dedicated to the PBS datastore.

This provides cleaner storage-role separation and prevents the backup repository from sharing the same physical device as the hypervisor workload storage.

## 3. SMART age alone is not treated as a failure indicator

Several devices have high power-on hours but clean SMART data.

This report distinguishes between:

- age/runtime risk
- actual media degradation
- SSD/NVMe endurance
- architectural impact of a device failure

The WD Caviar Black is the clearest example of a device that deserves increased attention because it combines very high runtime with actual sector reallocation.

---

# Recommended Maintenance

Run SMART health checks periodically and compare results against this baseline.

Important HDD indicators:

- Reallocated Sector Count
- Current Pending Sector Count
- Offline Uncorrectable
- Reported Uncorrectable Errors
- UDMA CRC Error Count
- SMART self-test results
- drive temperature

Important SSD/NVMe indicators:

- Wear Leveling Count
- Percentage Used
- Data Units Written
- Critical Warning
- Available Spare
- Media/Data Integrity Errors
- temperature

Replacement decisions should consider:

1. active SMART degradation
2. increasing error counts over time
3. endurance consumption
4. workload importance
5. backup coverage
6. replacement cost and service impact

Drives with high runtime but stable SMART counters do not require immediate replacement solely because of age.

---

# Assessment Snapshot

```text
PVE-main
├── 4 × IronWolf 8 TB ............... Very Low
├── WD Black 2 TB ................... Elevated
├── WD Green 2 TB ................... Moderate
├── WD Green 3 TB ................... Moderate
├── Samsung 850 EVO 250 GB .......... Low–Moderate
└── Samsung 970 EVO Plus 1 TB ....... Low

ProDesk
├── WD_BLACK SN7100 1 TB ............ Very Low
└── Samsung 990 PRO 2 TB ............ Very Low

EliteDesk
└── Kioxia 256 GB NVMe .............. Very Low

PVE-game-server
└── Samsung 860 EVO 1 TB ............ Moderate

ThinkCentre M900
├── WD_BLACK SN850X 1 TB ............ Very Low
└── Seagate Barracuda 2 TB .......... Very Low
```

---

# Conclusion

The homelab storage environment is generally healthy based on the currently collected SMART data.

The primary device requiring closer observation is the **WD Caviar Black 2 TB on PVE-main**, due to its combination of approximately 75,000 power-on hours and two reallocated sectors.

The older WD Green HDDs and Samsung 860 EVO have high accumulated runtime but currently show no active SMART failure indicators.

The newer NVMe devices on ProDesk, EliteDesk, and the ThinkCentre M900 show negligible endurance consumption and no critical warnings. The ThinkCentre's 2 TB Seagate Barracuda also currently reports clean sector-related SMART counters.

This document should be treated as a baseline and updated periodically so that changes in SMART attributes, endurance, and device roles can be tracked over time.
