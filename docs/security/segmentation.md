# Security Segmentation

This document describes the security segmentation model used across the homelab.

The purpose of the segmentation design is to separate systems by trust level, limit unnecessary lateral movement, and keep administrative access distinct from lower-trust or externally reachable workloads.

Detailed VLAN configuration and Gateway ACL rules are documented separately under the network documentation.

---

## Overview

The homelab is divided into several security zones:

| VLAN | Name | Trust Role |
|---:|---|---|
| 1 | `DEFAULT` | Restricted legacy/default network |
| 10 | `INFRA` | Infrastructure and management services |
| 20 | `TRUSTED` | Trusted personal and administrative clients |
| 25 | `GAME-DMZ` | Externally reachable game workloads |
| 30 | `IOT` | Lower-trust smart-home and IoT devices |
| 40 | `GUEST` | Untrusted guest clients |

The main principle is:

```text
Different trust level
        │
        ▼
Different network zone
        │
        ▼
Explicitly controlled communication
```

The design avoids placing infrastructure, personal clients, public-facing workloads, IoT devices, and guests on one unrestricted LAN.

---

# Security Goals

The segmentation model is intended to provide:

- separation between administrative systems and lower-trust devices
- reduced lateral movement between network zones
- limited exposure of internal infrastructure
- isolation of externally reachable game workloads
- restricted IoT and guest access
- explicit exceptions for required cross-VLAN communication
- a clear management path from trusted systems
- simpler troubleshooting and security reasoning

The design favors narrow exceptions over broad inter-VLAN access.

---

# Trust Model

The zones are not treated as equally trusted.

A simplified trust hierarchy is:

```text
TRUSTED
   │
   │ administrative access
   ▼
INFRA

GAME-DMZ
IOT
GUEST
DEFAULT
```

`TRUSTED` and `INFRA` form the administrative side of the environment.

`GAME-DMZ`, `IOT`, and `GUEST` are more restricted because they contain systems that are either externally reachable, less trusted, or not intended to administer internal infrastructure.

---

# TRUSTED

`TRUSTED` contains personal devices that are allowed to administer the homelab.

Typical examples include:

- primary Fedora administration workstation
- trusted personal devices where appropriate

The primary administration workstation resides on:

```text
TRUSTED / VLAN 20
```

The security role of `TRUSTED` is:

```text
Trusted client
      │
      │ approved management traffic
      ▼
Infrastructure
```

This allows administration of systems such as:

- Proxmox VE
- TrueNAS
- Proxmox Backup Server
- Omada Controller
- Pi-hole
- Home Assistant
- Homepage
- other infrastructure services

Lower-trust zones are not given equivalent access back toward `TRUSTED`.

---

# INFRA

`INFRA` contains the core infrastructure and management plane.

Examples include:

- Proxmox management interfaces
- TrueNAS
- Proxmox Backup Server
- Pi-hole
- Home Assistant
- Omada Controller
- Homepage
- network infrastructure management
- JetKVM

The security role of `INFRA` is:

```text
Internal infrastructure
        │
        ├── management services
        ├── DNS
        ├── storage
        ├── backup
        └── automation
```

Systems on `INFRA` are more trusted than IoT, guest, or public-facing game workloads.

However, membership in `INFRA` does not automatically imply unrestricted access to every other VLAN.

Cross-zone communication is still limited where appropriate.

---

# GAME-DMZ

`GAME-DMZ` contains game workloads that may be reachable from outside the local network.

Current examples include:

- Palworld
- Valheim
- game-server Linux workloads

The dedicated game-server VM is located on:

```text
GAME-DMZ / VLAN 25
```

while the physical Proxmox management interface remains on:

```text
INFRA / VLAN 10
```

This creates an intentional separation:

```text
Proxmox management
      │
      └── INFRA

Game workload
      │
      └── GAME-DMZ
```

The game workload is therefore not placed directly inside the infrastructure management zone.

---

## Why Isolate Game Servers?

Game servers have a different risk profile from internal management services.

They may:

- accept connections from external players
- run third-party game-server software
- receive frequent application updates
- expose network services through Playit.gg
- contain software not required for infrastructure administration

The design therefore assumes that a compromise of a game workload should not automatically provide unrestricted access to internal infrastructure.

---

# IOT

`IOT` contains smart-home and embedded devices.

Examples include:

- television
- robot vacuum
- smart speakers
- other network-connected smart-home devices

These devices are placed on:

```text
IOT / VLAN 30
```

rather than sharing a network with trusted administration systems.

The intended security model is:

```text
IoT device
    │
    ├── Internet where required
    ├── DNS where required
    │
    └── blocked from general infrastructure access
```

IoT devices are treated as lower-trust endpoints.

---

# Home Assistant Exception

Home Assistant is hosted on `INFRA`, while many of the devices it manages are on `IOT`.

This requires a deliberate exception.

```text
Home Assistant
192.168.10.70
      │
      │ permitted
      ▼
IOT
```

The important direction is:

```text
Home Assistant → IOT     allowed
IOT → Infrastructure     restricted
```

This allows Home Assistant to control smart-home devices without placing Home Assistant itself inside the lower-trust IoT zone.

The detailed Gateway ACL implementation belongs in:

[`../network/acl-policy.md`](../network/acl-policy.md)

---

# GUEST

`GUEST` is intended for devices that should receive Internet access without access to the internal homelab.

The intended behavior is:

```text
GUEST
  │
  ├── Internet
  └── no normal access to internal networks
```

Guest clients should not be able to administer or directly access:

- `INFRA`
- `TRUSTED`
- `IOT`
- `GAME-DMZ`

Required DNS access is provided separately through the network policy.

---

# DEFAULT

`DEFAULT / VLAN 1` is deliberately not used as the normal trusted client network.

It is retained as a restricted legacy/default network and for specific infrastructure requirements.

The ER605 management interface currently remains on `DEFAULT`.

Normal trusted client activity is instead placed on `TRUSTED`, while infrastructure services are placed on `INFRA`.

This reduces reliance on the implicit default network as a general-purpose trusted zone.

---

# DNS Exceptions

The homelab uses two Pi-hole instances on `INFRA`.

Lower-trust VLANs still require DNS, so DNS is treated as a narrow infrastructure exception rather than as a reason to allow broad access to `INFRA`.

Conceptually:

```text
GAME-DMZ ──┐
IOT      ──┼── DNS only ──► Pi-hole pair
GUEST    ──┘
```

The two DNS servers are:

```text
192.168.10.90
192.168.10.91
```

The complete ACL implementation and object definitions are maintained in:

[`../network/acl-policy.md`](../network/acl-policy.md)

---

# High-Level Traffic Model

The segmentation model can be summarized as:

```text
TRUSTED ───────► INFRA
    │
    └──────────► required services

INFRA ──X─────► IOT
    │
    └── exception → Home Assistant → IOT

GAME-DMZ ──X──► private infrastructure
    │
    └─────────► DNS / Internet as required

IOT ─────X────► trusted/internal administration
    │
    └─────────► DNS / Internet as required

GUEST ───X────► internal networks
    │
    └─────────► DNS / Internet
```

This is an architectural summary only.

The authoritative rule order, address groups, permit exceptions, and deny rules belong in the network ACL documentation.

---

# Management Plane Separation

Administrative interfaces are kept separate from public or lower-trust workloads where possible.

Examples include:

```text
Proxmox host management → INFRA
Game-server VM          → GAME-DMZ
```

and:

```text
Omada Controller        → INFRA
Trusted admin PC        → TRUSTED
```

The intent is that normal administration occurs from trusted paths rather than from guest, IoT, or game-server networks.

---

# Public Exposure

The homelab does not intentionally expose its management interfaces directly to the public Internet.

This includes services such as:

- Proxmox VE
- TrueNAS
- PBS
- Omada Controller
- Pi-hole
- Home Assistant
- Homepage

Remote administration uses a private remote-access path.

Public game connectivity is handled separately through Playit.gg.

This keeps:

```text
Administrative access
        ≠
Public game access
```

Detailed remote-access architecture belongs in:

[`remote-access.md`](remote-access.md)

---

# Tailscale

Tailscale is used for private remote administration.

It is not treated as a replacement for the internal VLAN security model.

The two layers serve different purposes:

```text
VLAN segmentation
      │
      └── controls local trust boundaries

Tailscale
      │
      └── provides private remote access
```

Internal security policy should therefore remain meaningful even when remote access is available.

---

# Playit.gg

Playit.gg is used for external game-server connectivity.

It is not part of the management plane.

Conceptually:

```text
External player
      │
      ▼
Playit.gg
      │
      ▼
GAME-DMZ workload
```

This is intentionally separated from:

```text
Tailscale / administrative access
```

Game exposure should not require publishing Proxmox, SSH, or other infrastructure management interfaces.

---

# Physical and Virtual Separation

Segmentation is implemented through both network and workload placement.

Examples include:

- game workloads isolated on `GAME-DMZ`
- infrastructure services placed on `INFRA`
- trusted administration on `TRUSTED`
- IoT devices on `IOT`
- guest clients on `GUEST`
- Pi-hole redundancy across ProDesk and EliteDesk
- game workloads moved to a dedicated Proxmox host where possible

Network segmentation does not eliminate every shared failure domain, but it reduces unnecessary trust relationships between workloads.

---

# Least Privilege

The design follows a least-privilege approach.

The preferred pattern is:

```text
Block broad access
      │
      ▼
Add narrow exception
```

rather than:

```text
Allow entire VLAN
```

Examples include:

- Home Assistant specifically allowed toward `IOT`
- lower-trust VLANs allowed to reach Pi-hole DNS
- administrative access originating from `TRUSTED`
- read-only API permissions for monitoring integrations where possible

This keeps required functionality without treating convenience as a reason for unrestricted communication.

---

# Service Isolation

Not every service needs its own VLAN.

The current design groups systems primarily by trust level and role rather than creating a VLAN for every application.

For example:

```text
INFRA
├── TrueNAS
├── PBS
├── Pi-hole
├── Home Assistant
├── Omada
└── Homepage
```

These services share a similar infrastructure trust role.

By contrast, game servers and IoT devices are separated because their security characteristics differ substantially from infrastructure services.

---

# Multicast and Service Discovery

Cross-VLAN mDNS/multicast forwarding is not currently enabled as part of the normal design.

This reduces automatic discovery between network zones.

If cross-VLAN discovery becomes necessary later, it should be introduced deliberately and only for the required services rather than broadly forwarding discovery traffic between all VLANs.

---

# Failure and Compromise Containment

Segmentation is intended to reduce blast radius.

Examples:

## Compromised IoT Device

The device should not automatically gain broad access to:

- trusted clients
- Proxmox management
- storage management
- network administration

---

## Compromised Game Server

A compromised game workload should remain separated from:

- Proxmox management
- TrueNAS management
- PBS management
- Omada management
- trusted personal devices

The security value of `GAME-DMZ` is therefore containment, not merely organizational separation.

---

## Guest Device

A guest client should be able to use the Internet without becoming a trusted participant in the internal network.

---

# Security Dependencies

Segmentation depends on several infrastructure components working correctly:

```text
ER605
  │
  ├── VLAN routing
  └── Gateway ACL enforcement

SG3210X-M2 / ES216G
  │
  └── correct access and trunk configuration

EAP670
  │
  └── correct wireless VLAN placement
```

Incorrect port or VLAN configuration can bypass the intended logical design even if the ACL policy itself is correct.

For that reason, network segmentation should be validated after significant switch, trunk, gateway, or WLAN changes.

---

# Validation

Security segmentation should be function-tested rather than assumed from configuration alone.

Representative tests include:

- `TRUSTED` can reach required infrastructure
- `GUEST` cannot access internal networks
- `IOT` cannot initiate unrestricted access toward infrastructure
- Home Assistant can reach required IoT devices
- `GAME-DMZ` cannot access private infrastructure broadly
- isolated VLANs can still reach Pi-hole DNS
- normal Internet connectivity remains available where intended

The implemented network has already undergone functional inter-VLAN validation.

---

# Operational Principles

Security-relevant network changes should follow several rules:

1. Make one major VLAN, ACL, or management-path change at a time.
2. Preserve a rollback path before changing trunks or management VLANs.
3. Place specific permit exceptions before corresponding deny rules.
4. Verify the intended traffic immediately after a change.
5. Confirm that management from `TRUSTED` remains available.
6. Avoid broad temporary allow rules unless strictly necessary for troubleshooting.
7. Remove temporary troubleshooting exceptions when validation is complete.

These principles reduce the risk of both accidental lockout and unintended access.

---

# Current State

Current confirmed state:

- `DEFAULT`: restricted legacy/default network
- `INFRA`: infrastructure and management services
- `TRUSTED`: trusted administration
- `GAME-DMZ`: isolated game workloads
- `IOT`: lower-trust smart-home devices
- `GUEST`: isolated guest access
- Gateway ACL segmentation: active
- DNS exceptions for isolated VLANs: active
- Home Assistant → IoT exception: active
- general IoT → infrastructure access: restricted
- game workloads separated from infrastructure management
- direct public management exposure: not used
- Tailscale: used for private remote administration
- Playit.gg: used for public game connectivity
- cross-VLAN mDNS/multicast forwarding: not currently enabled
- inter-VLAN behavior: function-tested

---

# Roadmap

Potential improvements include:

- continue reviewing ACL scope as services change
- periodically validate cross-VLAN behavior
- document remote-access hardening in greater detail
- review management-plane exposure after major migrations
- introduce mDNS/multicast only if a defined use case requires it
- continue disabling or restricting unused network paths
- periodically review whether service integrations still require their current permissions

---

# Scope of This Document

This file owns the high-level security segmentation model:

- trust zones
- security rationale
- management separation
- public-service separation
- least-privilege relationships
- DNS exceptions at a conceptual level
- Home Assistant / IoT trust relationship
- remote/public-access separation
- compromise containment
- validation principles

It does not own:

- exact Gateway ACL rules
- ACL ordering
- switch-port configuration
- VLAN subnet definitions
- DHCP configuration
- service-specific security settings
- Tailscale implementation details
- Playit.gg credentials

Those belong in the dedicated network, service, remote-access, or private configuration documentation.

---

## Related Documentation

- [`../network/README.md`](../network/README.md) — network overview
- [`../network/vlan-design.md`](../network/vlan-design.md) — VLAN design and rationale
- [`../network/acl-policy.md`](../network/acl-policy.md) — implemented Gateway ACL rules
- [`../network/port-mapping.md`](../network/port-mapping.md) — switch-port and trunk configuration
- [`../network/ip-plan.md`](../network/ip-plan.md) — addressing
- [`remote-access.md`](remote-access.md) — private remote administration
- [`backup-strategy.md`](backup-strategy.md) — backup and resilience model
- [`../services/pihole.md`](../services/pihole.md) — DNS infrastructure
- [`../services/home-assistant.md`](../services/home-assistant.md) — IoT control exception
- [`../services/game-servers.md`](../services/game-servers.md) — GAME-DMZ workloads
