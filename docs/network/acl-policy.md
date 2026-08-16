# ACL Policy

This document defines the implemented access-control policy between the homelab network zones.

The policy is enforced primarily through **Gateway ACLs** on the TP-Link ER605 V2 and is managed through the Omada Controller.

This document owns:

- Active Gateway ACL rules
- Rule ordering
- Permit exceptions
- Deny rules
- ACL object groups
- Inter-VLAN traffic policy
- Validation status
- Deferred hardening

It intentionally does not duplicate the complete IP plan or switch-port configuration.

Related documentation:

- [`vlan-design.md`](vlan-design.md) — why the security zones exist
- [`ip-plan.md`](ip-plan.md) — subnets, infrastructure addresses, DHCP, and DNS
- [`port-mapping.md`](port-mapping.md) — switch ports and VLAN profiles

---

## Policy Goals

The ACL policy implements the trust model defined in [`vlan-design.md`](vlan-design.md).

The main goals are:

1. Keep `DEFAULT` out of normal production use.
2. Allow `TRUSTED` to act as the administration network.
3. Prevent `INFRA` from initiating unnecessary traffic toward lower-trust or client networks.
4. Isolate `GAME-DMZ` from private infrastructure.
5. Isolate `IOT` from internal systems.
6. Isolate `GUEST` from internal systems.
7. Preserve Internet access where required.
8. Preserve DNS through Pi-hole using narrow exceptions.
9. Permit Home Assistant to communicate with IoT devices without opening all of `INFRA`.

---

## Enforcement Point

Current inter-VLAN segmentation is enforced through:

| ACL Type | Current Use |
|---|---|
| Gateway ACL | **Active** |
| Switch ACL | Not used |
| EAP ACL | Not used |

Using the gateway as the primary policy enforcement point keeps routed inter-VLAN policy in one location.

This reduces:

- overlapping rule sets
- conflicting enforcement
- troubleshooting complexity
- configuration drift

---

## Network Zones

| VLAN | Name | Security Role |
|---:|---|---|
| 1 | `DEFAULT` | Restricted legacy/management zone |
| 10 | `INFRA` | Servers and infrastructure |
| 20 | `TRUSTED` | Administration and trusted clients |
| 25 | `GAME-DMZ` | Game-server workloads |
| 30 | `IOT` | Smart-home and IoT devices |
| 40 | `GUEST` | Guest clients |

Subnets and addresses are maintained in [`ip-plan.md`](ip-plan.md).

---

## Policy Model

The ACL policy is intentionally asymmetric.

For example:

```text
TRUSTED → INFRA   PERMIT
INFRA   → TRUSTED DENY
```

The administration workstation needs to initiate connections toward infrastructure.

Infrastructure systems do not automatically require the ability to initiate connections back toward trusted clients.

The same principle is used for game-server administration and IoT access.

---

## Traffic Matrix

`ALLOW` means the source zone may initiate general traffic toward the destination.

`BLOCK` means new connections initiated from the source are denied by policy.

`DNS ONLY` means only the explicit Pi-hole DNS exception is permitted.

| Source ↓ / Destination → | INFRA | TRUSTED | GAME-DMZ | IOT | GUEST | Internet |
|---|---|---|---|---|---|---|
| `DEFAULT` | BLOCK | BLOCK | BLOCK | BLOCK | BLOCK | BLOCK |
| `INFRA` | — | BLOCK | BLOCK | BLOCK* | BLOCK | ALLOW |
| `TRUSTED` | ALLOW | — | ALLOW | ALLOW | Not explicitly allowed | ALLOW |
| `GAME-DMZ` | DNS ONLY | BLOCK | — | BLOCK | BLOCK | ALLOW |
| `IOT` | DNS ONLY | BLOCK | BLOCK | — | BLOCK | ALLOW |
| `GUEST` | DNS ONLY | BLOCK | BLOCK | BLOCK | — | ALLOW |

`*` Home Assistant has an explicit `INFRA → IOT` exception.

---

# Active Gateway ACL Rules

The order below reflects the implemented policy.

Rule order matters because specific permit exceptions must be evaluated before the broader deny rule they override.

---

## 1. `BLOCK-DEFAULT-GATEWAY-MGMT`

**Purpose:** Prevent devices on `DEFAULT` from accessing the ER605 management interface.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `DEFAULT` |
| Destination | `Gateway Management Page` |

### Rationale

`DEFAULT` is not intended as an administrative client network.

The gateway remains managed on this network at the addressing level, but devices placed on `DEFAULT` should not be able to use that fact as an administrative path.

---

## 2. `BLOCK-DEFAULT-INTER-VLAN`

**Purpose:** Prevent `DEFAULT` from initiating traffic toward the operational VLANs.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `DEFAULT` |
| Destination | `INFRA`, `TRUSTED`, `GAME-DMZ`, `IOT`, `GUEST` |

### Rationale

`DEFAULT` should not function as an unrestricted fallback network.

---

## 3. `BLOCK-DEFAULT-INTERNET`

**Purpose:** Remove Internet access from `DEFAULT`.

| Field | Value |
|---|---|
| Direction | `LAN → WAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `DEFAULT` |
| Destination | `IPGroup_Any` |

### Result

`DEFAULT` is effectively restricted from:

- production VLANs
- Internet access
- gateway management

---

## 4. `ALLOW-IOT-PIHOLE-DNS`

**Purpose:** Allow IoT devices to use the internal Pi-hole DNS service.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `IOT` |
| Destination | `PIHOLE-DNS-53` |

`PIHOLE-DNS-53` contains the two Pi-hole endpoints on DNS port 53.

The corresponding infrastructure addresses are documented in [`ip-plan.md`](ip-plan.md).

### Ordering Requirement

This rule must remain above:

```text
BLOCK-IOT-INTER-VLAN
```

---

## 5. `BLOCK-IOT-INTER-VLAN`

**Purpose:** Prevent IoT devices from initiating general connections toward the other internal zones.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `IOT` |
| Destination | `INFRA`, `TRUSTED`, `GAME-DMZ`, `GUEST` |

### Intended Result

`IOT` retains:

- Internet access
- Pi-hole DNS

but does not receive unrestricted access to private networks.

### Verified

- Internet access: ✅
- Pi-hole DNS: ✅
- `IOT → INFRA`: blocked ✅
- `IOT → TRUSTED`: blocked ✅

---

## 6. `ALLOW-GAME-DMZ-PIHOLE-DNS`

**Purpose:** Allow game-server workloads to use Pi-hole DNS.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `GAME-DMZ` |
| Destination | `PIHOLE-DNS-53` |

### Ordering Requirement

This rule must remain above:

```text
BLOCK-GAME-DMZ-INTER-VLAN
```

---

## 7. `BLOCK-GAME-DMZ-INTER-VLAN`

**Purpose:** Isolate game-server workloads from the private internal networks.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `GAME-DMZ` |
| Destination | `INFRA`, `TRUSTED`, `IOT`, `GUEST` |

### Intended Result

`GAME-DMZ` retains:

- Internet access
- Pi-hole DNS
- Playit.gg connectivity
- game/software update access

but cannot initiate unrestricted traffic toward private internal systems.

### Verified

- Internet access: ✅
- Pi-hole DNS: ✅
- `GAME-DMZ → INFRA`: blocked ✅
- `GAME-DMZ → TRUSTED`: blocked ✅

---

## 8. `ALLOW-GUEST-INTERNET`

**Purpose:** Explicitly allow guest clients to access the Internet.

| Field | Value |
|---|---|
| Direction | `LAN → WAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `GUEST` |
| Destination | `IPGroup_Any` |

This rule applies only toward WAN traffic and does not grant access to internal VLANs.

---

## 9. `ALLOW-GUEST-PIHOLE-DNS`

**Purpose:** Allow guest devices to use Pi-hole DNS on `INFRA`.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `GUEST` |
| Destination | `PIHOLE-DNS-53` |

### Ordering Requirement

This rule must remain above:

```text
BLOCK-GUEST-INTER-VLAN
```

---

## 10. `BLOCK-GUEST-INTER-VLAN`

**Purpose:** Prevent guest devices from initiating connections toward internal networks.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `GUEST` |
| Destination | `INFRA`, `TRUSTED`, `GAME-DMZ`, `IOT` |

### Intended Result

Guests receive:

- Internet access
- Pi-hole DNS

but no general access to the homelab.

### Verified

- DHCP/connectivity: ✅
- Internet access: ✅
- Pi-hole DNS: ✅
- `GUEST → INFRA`: blocked ✅
- `GUEST → TRUSTED`: blocked ✅

### Guest Isolation Design

Omada's SSID-level `Guest Network` feature is not used as the primary isolation mechanism.

Isolation is instead implemented through:

- VLAN 40
- Gateway ACLs

This allows the explicit Pi-hole DNS exception to remain under direct policy control.

---

## 11. `ALLOW-TRUSTED-INTER-VLAN`

**Purpose:** Establish `TRUSTED` as the administration network.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `TRUSTED` |
| Destination | `INFRA`, `GAME-DMZ`, `IOT` |

### Intended Result

Trusted clients can initiate administrative or required traffic toward:

- infrastructure
- game workloads
- IoT devices

The reverse directions are not automatically permitted.

### Verified

From the trusted administration network:

- `TRUSTED → INFRA`: works ✅
- `TRUSTED → GAME-DMZ`: works ✅
- `TRUSTED → IOT`: works ✅

---

## 12. `ALLOW-HA-IOT`

**Purpose:** Allow Home Assistant to communicate with IoT devices.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Permit` |
| Protocol | `All` |
| Source | `HOME-ASSISTANT` |
| Destination | `IOT` |

`HOME-ASSISTANT` is an IP/object group representing only the Home Assistant service.

Its address is documented in [`ip-plan.md`](ip-plan.md).

### Ordering Requirement

This rule must remain above:

```text
BLOCK-INFRA-TO-IOT
```

### Rationale

Only Home Assistant requires this broad service relationship.

Allowing the whole `INFRA` network to initiate traffic toward `IOT` would be wider than necessary.

### Verified

- Home Assistant → IoT control: works ✅
- Other `INFRA → IOT` traffic: blocked ✅

---

## 13. `BLOCK-INFRA-TO-TRUSTED-GUEST`

**Purpose:** Prevent infrastructure systems from initiating connections toward user and guest networks.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `INFRA` |
| Destination | `TRUSTED`, `GUEST` |

### Rationale

The relationship is intentionally asymmetric:

```text
TRUSTED → INFRA   permitted
INFRA   → TRUSTED denied
```

Trusted systems can administer infrastructure without automatically allowing infrastructure services to initiate connections toward personal endpoints.

### Verified

- `INFRA → TRUSTED`: blocked ✅
- `TRUSTED → INFRA`: works ✅

---

## 14. `BLOCK-INFRA-TO-IOT`

**Purpose:** Prevent general infrastructure systems from initiating connections toward IoT devices.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `INFRA` |
| Destination | `IOT` |

### Exception

Home Assistant is permitted by:

```text
ALLOW-HA-IOT
```

which must appear before this deny rule.

### Verified

- General `INFRA → IOT`: blocked ✅
- Home Assistant → IoT: works ✅

---

## 15. `BLOCK-INFRA-TO-GAME-DMZ`

**Purpose:** Prevent infrastructure systems from initiating general connections toward game-server workloads.

| Field | Value |
|---|---|
| Direction | `LAN → LAN` |
| Policy | `Deny` |
| Protocol | `All` |
| Source | `INFRA` |
| Destination | `GAME-DMZ` |

### Rationale

Administration of the game environment should originate from `TRUSTED`, not from arbitrary infrastructure systems.

This maintains separation between private infrastructure and externally oriented workloads.

### Verified

- `INFRA → GAME-DMZ`: blocked ✅
- `TRUSTED → GAME-DMZ`: works ✅

---

# Rule Order

The active Gateway ACL order is:

| Index | Rule | Policy |
|---:|---|---|
| 1 | `BLOCK-DEFAULT-GATEWAY-MGMT` | Deny |
| 2 | `BLOCK-DEFAULT-INTER-VLAN` | Deny |
| 3 | `BLOCK-DEFAULT-INTERNET` | Deny |
| 4 | `ALLOW-IOT-PIHOLE-DNS` | Permit |
| 5 | `BLOCK-IOT-INTER-VLAN` | Deny |
| 6 | `ALLOW-GAME-DMZ-PIHOLE-DNS` | Permit |
| 7 | `BLOCK-GAME-DMZ-INTER-VLAN` | Deny |
| 8 | `ALLOW-GUEST-INTERNET` | Permit |
| 9 | `ALLOW-GUEST-PIHOLE-DNS` | Permit |
| 10 | `BLOCK-GUEST-INTER-VLAN` | Deny |
| 11 | `ALLOW-TRUSTED-INTER-VLAN` | Permit |
| 12 | `ALLOW-HA-IOT` | Permit |
| 13 | `BLOCK-INFRA-TO-TRUSTED-GUEST` | Deny |
| 14 | `BLOCK-INFRA-TO-IOT` | Deny |
| 15 | `BLOCK-INFRA-TO-GAME-DMZ` | Deny |

---

## Ordering Dependencies

The most important ordering relationships are:

```text
ALLOW-IOT-PIHOLE-DNS
    before
BLOCK-IOT-INTER-VLAN
```

```text
ALLOW-GAME-DMZ-PIHOLE-DNS
    before
BLOCK-GAME-DMZ-INTER-VLAN
```

```text
ALLOW-GUEST-PIHOLE-DNS
    before
BLOCK-GUEST-INTER-VLAN
```

```text
ALLOW-HA-IOT
    before
BLOCK-INFRA-TO-IOT
```

These permit rules are exceptions to broader deny rules.

Changing their order can break required services while leaving the overall network superficially reachable.

---

# ACL Object Groups

The ACL design uses named groups so policy refers to services and roles instead of scattering raw addresses throughout every rule.

## `PIHOLE-DNS-53`

Represents:

- Pi-hole 1
- Pi-hole 2
- DNS port 53

Used by:

- `ALLOW-IOT-PIHOLE-DNS`
- `ALLOW-GAME-DMZ-PIHOLE-DNS`
- `ALLOW-GUEST-PIHOLE-DNS`

The current Pi-hole addresses are maintained in [`ip-plan.md`](ip-plan.md).

---

## `HOME-ASSISTANT`

Represents only the Home Assistant service.

Used by:

```text
ALLOW-HA-IOT
```

The current address is maintained in [`ip-plan.md`](ip-plan.md).

---

## `IPGroup_Any`

Used where an ACL applies broadly toward WAN/Internet destinations.

Examples:

- `BLOCK-DEFAULT-INTERNET`
- `ALLOW-GUEST-INTERNET`

---

# Validation

The policy has been function-tested across the primary security boundaries.

| Test | Expected | Result |
|---|---|---|
| `IOT → Internet` | Allow | ✅ |
| `IOT → Pi-hole DNS` | Allow | ✅ |
| `IOT → INFRA` | Block | ✅ |
| `IOT → TRUSTED` | Block | ✅ |
| `GAME-DMZ → Internet` | Allow | ✅ |
| `GAME-DMZ → Pi-hole DNS` | Allow | ✅ |
| `GAME-DMZ → INFRA` | Block | ✅ |
| `GAME-DMZ → TRUSTED` | Block | ✅ |
| `GUEST → Internet` | Allow | ✅ |
| `GUEST → Pi-hole DNS` | Allow | ✅ |
| `GUEST → INFRA` | Block | ✅ |
| `GUEST → TRUSTED` | Block | ✅ |
| `TRUSTED → INFRA` | Allow | ✅ |
| `TRUSTED → GAME-DMZ` | Allow | ✅ |
| `TRUSTED → IOT` | Allow | ✅ |
| General `INFRA → TRUSTED` | Block | ✅ |
| General `INFRA → IOT` | Block | ✅ |
| Home Assistant → `IOT` | Allow | ✅ |
| General `INFRA → GAME-DMZ` | Block | ✅ |

The validation focuses on actual policy behavior rather than merely confirming that ACL objects exist in Omada.

---

# Deferred Hardening

The current policy is operational and stable.

Several possible hardening steps have intentionally been deferred.

## Gateway Management Isolation

Additional restrictions may later be added for:

```text
IOT      → Gateway Management Page
GAME-DMZ → Gateway Management Page
GUEST    → Gateway Management Page
```

These changes were deferred because management-path changes carry higher operational risk and should be introduced separately with rollback planning.

---

## TRUSTED → GUEST

`TRUSTED → GUEST` is not explicitly permitted as part of the current policy.

A stricter explicit isolation rule may be added later if complete guest separation from the administrative network is required.

---

## mDNS / Multicast

Cross-VLAN discovery is not currently part of the ACL design.

If services such as Chromecast, smart speakers, or TVs require discovery across `TRUSTED` and `IOT`, mDNS/multicast should be implemented as a controlled service requirement rather than by weakening the general segmentation policy.

---

# Change-Control Rules

ACL changes can affect multiple services at once.

The following process should be used:

1. Identify the exact required communication path.
2. Confirm the current source and destination zones.
3. Determine whether an existing rule already covers the requirement.
4. Prefer a narrow permit exception over a broad allow rule.
5. Place the exception above its corresponding deny rule.
6. Apply one policy change at a time.
7. Test both the intended allowed path and at least one path that should remain blocked.
8. Record the change in documentation.

---

## Example

If a new service on `INFRA` requires access to a specific IoT device, the preferred approach is:

```text
specific INFRA service
        │
        ▼
specific required IOT service
        │
        PERMIT
        ▼
general INFRA → IOT DENY
```

The preferred approach is **not**:

```text
INFRA → IOT ALLOW ALL
```

---

# Security Interpretation

The implemented ACL model follows several broader principles.

### Least Privilege

Only the communication paths required by a service are opened.

### Zone Separation

Devices are assigned to security zones based on role and trust level.

### Asymmetric Administration

Trusted clients can administer infrastructure without automatically giving infrastructure equivalent access back.

### Management-Plane Protection

Hypervisor and infrastructure management remain separate from game workloads and IoT networks.

### Defense in Depth

VLANs create logical separation while Gateway ACLs determine which routed communication is allowed between them.

---

# Current Status

The Gateway ACL policy is implemented and operational.

Current state:

- `DEFAULT` isolation: ✅
- `IOT` isolation: ✅
- `GAME-DMZ` isolation: ✅
- `GUEST` isolation: ✅
- `TRUSTED` administration path: ✅
- `INFRA` outbound restrictions: ✅
- Pi-hole DNS exceptions: ✅
- Home Assistant → IoT exception: ✅
- Core policy paths function-tested: ✅

Switch ACLs and EAP ACLs are currently empty.

---

# Public Repository Notes

This document describes policy logic but intentionally excludes:

- credentials
- device passwords
- API tokens
- private keys
- MAC addresses
- public/WAN addressing
- Tailscale addressing
- SSID credentials

Private RFC1918 addresses are maintained in [`ip-plan.md`](ip-plan.md) where they are required to understand the internal architecture.

---

# Scope of This Document

This file is the authoritative location for the **implemented ACL policy**.

It owns:

- rule definitions
- rule order
- object-group purpose
- traffic matrix
- validation
- deferred ACL hardening

It does not own:

- VLAN design rationale
- full IP addressing
- DHCP configuration
- physical switch ports
- incident recovery history

---

## Related Documentation

- [`README.md`](README.md) — network overview
- [`vlan-design.md`](vlan-design.md) — VLAN trust model and rationale
- [`ip-plan.md`](ip-plan.md) — addressing and DNS
- [`port-mapping.md`](port-mapping.md) — physical/logical switch-port configuration
- [`incidents/es216g-recovery.md`](incidents/es216g-recovery.md) — ES216G incident and recovery
- [`../security/segmentation.md`](../security/segmentation.md) — homelab security perspective
