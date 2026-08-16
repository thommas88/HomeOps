# Hardening

This document describes the general hardening principles used across the homelab.

Hardening means reducing unnecessary attack surface, limiting privileges, protecting administrative paths, and configuring systems so that a compromise of one component does not automatically provide broad access to the rest of the environment.

This document is intentionally high level. Detailed implementation belongs in the relevant node, network, service, or security documentation.

---

## Overview

The hardening model is based on several recurring principles:

- minimize public exposure
- separate systems by trust level
- use least privilege
- keep management paths private
- reduce unnecessary services and access
- maintain current software
- isolate externally reachable workloads
- protect credentials and secrets
- preserve recovery options
- validate changes after implementation

The goal is not to make the homelab impossible to compromise.

The goal is to reduce the number of easy paths an attacker or misconfiguration could use to move from one system to another.

---

# Hardening Principles

A simplified model is:

```text
Reduce exposure
      │
      ▼
Limit privileges
      │
      ▼
Separate trust zones
      │
      ▼
Protect credentials
      │
      ▼
Keep systems updated
      │
      ▼
Maintain recovery paths
```

These controls work together.

No single control is expected to provide complete protection.

---

# Management Plane Protection

Administrative interfaces are treated as sensitive.

Examples include:

- Proxmox VE
- TrueNAS
- Proxmox Backup Server
- Omada Controller
- Pi-hole
- Home Assistant
- SSH
- network-device management

These interfaces are not intentionally exposed directly to the public Internet.

Remote administration uses private access paths instead.

Detailed remote-access documentation:

[`remote-access.md`](remote-access.md)

---

# Network Segmentation

Systems are separated by trust level rather than placed on one unrestricted LAN.

Current zones include:

```text
DEFAULT
INFRA
TRUSTED
GAME-DMZ
IOT
GUEST
```

The main security relationships are:

```text
TRUSTED → administration

INFRA   → internal infrastructure

GAME-DMZ → externally reachable game workloads

IOT     → lower-trust smart-home devices

GUEST   → untrusted guest clients
```

This reduces the blast radius of a compromised workload or endpoint.

Detailed segmentation documentation:

[`segmentation.md`](segmentation.md)

---

# Least Privilege

Services and accounts should receive only the permissions they actually require.

Preferred model:

```text
Required permission only
        │
        ▼
Service performs its function
```

instead of:

```text
Administrator access everywhere
```

Examples include:

- read-only API access for monitoring integrations
- narrow ACL exceptions instead of broad VLAN access
- dedicated service accounts for game servers
- avoiding unnecessary root-level execution
- limiting administrative access to trusted paths

---

# API Accounts

Monitoring and dashboard integrations should not require full administrator privileges unless technically necessary.

For example:

```text
Homepage
    │
    ▼
Read-only Proxmox API account
```

is preferable to:

```text
Homepage
    │
    ▼
Full Proxmox administrator account
```

A compromised dashboard should not automatically become a path to full infrastructure control.

---

# Service Accounts

Applications should run using dedicated service accounts where practical.

Current examples include game-server workloads managed through Linux service accounts and systemd.

Benefits include:

- clearer process ownership
- reduced privilege
- easier auditing
- easier troubleshooting
- smaller impact if one service is compromised

Running unrelated services under the same privileged account should be avoided where possible.

---

# Public Exposure

Public exposure is kept separate from infrastructure administration.

Current model:

```text
Administrators
      │
      ▼
Tailscale
      │
      ▼
Private management interfaces
```

while game connectivity uses:

```text
Players
   │
   ▼
Playit.gg
   │
   ▼
GAME-DMZ workloads
```

This avoids using the same public path for both game traffic and infrastructure administration.

---

# SSH Hardening

SSH is used for Linux and Proxmox administration.

General hardening principles include:

- expose SSH only where required
- prefer private administrative paths
- avoid unnecessary Internet-facing SSH
- use strong authentication
- remove obsolete accounts
- keep the SSH server updated
- limit privileged access where practical

Private keys, credentials, and authentication material must not be committed to the public repository.

---

# Web Interface Hardening

Many homelab services use browser-based administration.

Examples include:

```text
Proxmox
TrueNAS
PBS
Omada
Pi-hole
Home Assistant
Homepage
Uptime Kuma
```

Hardening principles include:

- keep interfaces private
- use HTTPS where supported
- keep applications updated
- avoid default credentials
- avoid unnecessary administrative users
- use read-only accounts for integrations where possible
- do not expose internal dashboards directly to the Internet

---

# Network Device Hardening

Network infrastructure is part of the security boundary.

Relevant devices include:

- ER605 V2
- SG3210X-M2
- ES216G
- EAP670

General controls include:

- management through trusted paths
- VLAN-based separation
- Gateway ACL enforcement
- unused ports disabled where appropriate
- controlled trunk configuration
- restricted default network usage
- configuration backups before major changes

Detailed implementation belongs in:

[`../network/`](../network/)

---

# Unused Switch Ports

Unused switch ports should not automatically remain active.

Disabling unused ports reduces opportunities for accidental or unauthorized network access.

The current switch-port state is documented in:

[`../network/port-mapping.md`](../network/port-mapping.md)

This should be reviewed after hardware changes or new device deployments.

---

# DEFAULT Network

`DEFAULT / VLAN 1` is deliberately restricted.

It is not used as the normal trusted-client network.

Normal administrative clients reside on:

```text
TRUSTED
```

while infrastructure resides on:

```text
INFRA
```

Reducing reliance on the default network makes trust relationships more explicit.

---

# GAME-DMZ Hardening

Game servers have a different risk profile from internal infrastructure.

They may:

- accept connections from external players
- run frequently updated third-party software
- expose game ports
- use public connectivity through Playit.gg

They are therefore isolated in:

```text
GAME-DMZ / VLAN 25
```

The physical Proxmox management interface remains on:

```text
INFRA / VLAN 10
```

This keeps public game workloads separate from the hypervisor management plane.

---

# IoT Hardening

IoT devices are treated as lower-trust systems.

They reside on:

```text
IOT / VLAN 30
```

and do not receive unrestricted access to internal infrastructure.

Home Assistant receives the specific access required to control IoT devices.

Preferred relationship:

```text
Home Assistant → IOT
```

rather than:

```text
IOT → unrestricted INFRA access
```

This preserves functionality without flattening the trust boundary.

---

# Guest Hardening

Guest clients are isolated from internal networks.

The intended model is:

```text
GUEST
  │
  ├── Internet
  └── required DNS
```

without normal access to:

- `INFRA`
- `TRUSTED`
- `IOT`
- `GAME-DMZ`

Guest access should remain separate from administrative and personal trusted devices.

---

# DNS Hardening

Pi-hole provides internal DNS through two separate instances.

Current placement:

```text
Pi-hole 1 → prodesk
Pi-hole 2 → elitedesk
```

This provides DNS service redundancy across physical hosts.

Lower-trust VLANs receive DNS access through narrow network-policy exceptions rather than broad infrastructure access.

Detailed documentation:

[`../services/pihole.md`](../services/pihole.md)

---

# Backup Infrastructure Hardening

Backups are high-value infrastructure.

A compromised backup system can expose:

- operating systems
- application data
- internal configuration
- credentials stored inside guest filesystems

Current principles include:

- PBS hosted on `INFRA`
- no intentional public exposure
- trusted administrative access
- credentials kept outside GitHub
- dedicated local backup storage
- restore testing as part of the roadmap

Detailed backup documentation:

[`backup-strategy.md`](backup-strategy.md)

---

# Software Updates

Keeping software current reduces exposure to known vulnerabilities.

Update responsibilities include:

- Proxmox VE
- Debian and Ubuntu hosts
- TrueNAS
- PBS
- Home Assistant
- Omada Controller
- Pi-hole
- Docker-based services
- game servers
- Tailscale
- network-device firmware

Updates should be performed in a controlled manner rather than across all critical systems simultaneously.

---

# Update Safety

Hardening is not only about installing updates quickly.

A safe update process also includes:

```text
Verify system health
       │
       ▼
Confirm backup
       │
       ▼
Apply update
       │
       ▼
Validate service
```

For high-risk services, rollback or restore capability should be known before major changes are introduced.

---

# Secrets Management

Secrets must not be stored in the public repository.

This includes:

- passwords
- API tokens
- SSH private keys
- Tailscale authentication keys
- Playit.gg credentials
- session cookies
- webhooks
- private certificates or keys
- backup credentials
- encryption keys

Public documentation may describe how authentication works without publishing the authentication material itself.

---

# GitHub Hygiene

The public repository is treated as documentation, not as a secrets store.

Before committing screenshots, logs, or configuration snippets, review them for:

- passwords
- tokens
- usernames where unnecessary
- private URLs
- private keys
- authentication headers
- MAC addresses where not useful
- device serial numbers
- personal data
- query histories
- webhook URLs

Configuration examples should use placeholders where secret values would normally appear.

---

# Logging and Visibility

Logs are useful for troubleshooting and security review, but they may also contain sensitive information.

Examples include:

- DNS query history
- authentication events
- device names
- addresses
- API errors
- internal service paths

Logs should therefore be reviewed before publication.

Raw production logs generally do not belong in the public repository unless sanitized.

---

# Monitoring

Monitoring supports hardening by making unexpected failures or outages easier to detect.

Current monitoring includes:

```text
Uptime Kuma
```

running on the offsite ThinkCentre M900.

This allows availability monitoring to remain separate from the main homelab failure domain.

Detailed monitoring documentation:

[`../services/uptime-kuma.md`](../services/uptime-kuma.md)

---

# Recovery as a Security Control

Security also includes the ability to recover after failure, compromise, or bad configuration.

Relevant recovery layers include:

- PBS VM/LXC backups
- application-level backups
- game-save backups
- ZFS redundancy
- ZFS scrubs
- future ZFS snapshots
- future offsite backup

A secure system without a practical recovery path can still have a large operational impact after an incident.

---

# Change Control

High-risk changes should be introduced incrementally.

Examples include:

- VLAN changes
- Gateway ACL changes
- management VLAN changes
- switch trunk changes
- remote-access changes
- backup-storage changes
- major platform upgrades

Preferred workflow:

```text
Document current state
        │
        ▼
Confirm backup / rollback
        │
        ▼
Make one change
        │
        ▼
Validate
        │
        ▼
Continue
```

This reduces both security risk and accidental lockout.

---

# Privileged Systems

Some systems have broad control over the environment and should receive additional attention.

Examples include:

- Proxmox VE
- Omada Controller
- TrueNAS
- PBS
- Tailscale-enabled administration systems
- JetKVM

Compromise of these systems can have a larger impact than compromise of a normal client.

They should therefore remain on trusted management paths and avoid unnecessary exposure.

---

# Failure and Compromise Containment

Hardening should reduce blast radius.

Examples:

## Game Server Compromise

Expected containment:

```text
GAME-DMZ
   │
   X
   │
INFRA / TRUSTED
```

The game workload should not automatically gain broad administrative access.

---

## IoT Device Compromise

Expected containment:

```text
IOT
 │
 X
 │
Trusted infrastructure
```

The device may retain the network access required for its normal function without becoming a trusted administrator.

---

## Dashboard Compromise

A Homepage compromise should not automatically provide full Proxmox administrator access if the integration uses read-only credentials.

---

# Physical Security Considerations

The homelab is physically local and therefore still depends on physical access to the hardware.

Physical access can bypass many software controls.

The public repository should therefore avoid publishing unnecessary information that would make physical targeting or recovery abuse easier.

Exact physical security controls are outside the scope of the current repository.

---

# Current State

Current confirmed hardening practices include:

- VLAN-based trust segmentation: active
- Gateway ACL segmentation: active
- `TRUSTED` used for administration
- `INFRA` used for infrastructure
- `GAME-DMZ` used for externally reachable game workloads
- `IOT` isolated from trusted infrastructure
- `GUEST` isolated from internal networks
- direct public management exposure: not intentionally used
- Tailscale used for private remote administration
- Playit.gg separated from administrative access
- read-only API access used where appropriate
- unused switch ports disabled where appropriate
- credentials excluded from GitHub
- backups used before or alongside high-risk changes
- offsite monitoring through Uptime Kuma
- software updates performed as normal maintenance
- service-specific recovery procedures documented

---

# Roadmap

Potential hardening improvements include:

- review SSH authentication and disable unnecessary password access where practical
- review stale user and API accounts
- periodically review Tailscale devices and access
- periodically review unused switch ports
- review service privileges and API permissions
- add automated ZFS snapshots
- improve offsite backup coverage
- continue restore testing
- consider UPS protection for critical infrastructure
- periodically review public repository content for sensitive information
- continue removing obsolete services and configuration

---

# Scope of This Document

This file owns the high-level hardening principles used across the homelab:

- attack-surface reduction
- least privilege
- management-plane protection
- service-account principles
- network-device hardening
- software updates
- credential handling
- repository hygiene
- monitoring
- recovery as a security control
- change-control principles
- compromise containment

It does not own:

- exact Gateway ACL rules
- detailed VLAN configuration
- host-specific firewall rules
- complete SSH configuration
- full user/account inventories
- private credentials
- service-specific configuration
- detailed Tailscale policy

Those belong in the relevant node, network, service, remote-access, or private documentation.

---

## Related Documentation

- [`segmentation.md`](segmentation.md) — trust zones and network separation
- [`remote-access.md`](remote-access.md) — private administration
- [`backup-strategy.md`](backup-strategy.md) — backup and recovery
- [`../network/acl-policy.md`](../network/acl-policy.md) — Gateway ACL implementation
- [`../network/port-mapping.md`](../network/port-mapping.md) — switch-port configuration
- [`../network/vlan-design.md`](../network/vlan-design.md) — VLAN architecture
- [`../services/proxmox-backup-server.md`](../services/proxmox-backup-server.md) — backup platform
- [`../services/uptime-kuma.md`](../services/uptime-kuma.md) — offsite monitoring
- [`../services/game-servers.md`](../services/game-servers.md) — GAME-DMZ workloads
