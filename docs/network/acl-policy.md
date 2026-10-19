# Network Access Control Policy

> **Public documentation** — This document describes the homelab's network access-control design, trust boundaries, and recorded validation outcomes. It is not an export of the live firewall configuration and must not be used as a recovery runbook.

## Purpose and Scope

The access-control policy governs **routed traffic between network security zones** in the Proxmox-based homelab. Its purpose is to limit unnecessary communication, maintain administrative access, and support services that require carefully scoped cross-zone connectivity.

Inter-VLAN policy is implemented primarily through **Gateway ACLs** on a TP-Link ER605 V2, managed with Omada. The gateway acts as the central enforcement point for routed traffic between VLANs.

This document covers:

- Security objectives and trust boundaries
- High-level inter-zone communication policy
- Business and operational justification for required exceptions
- Recorded functional validation and its limitations
- Change-control principles

It intentionally excludes live ACL names and sequence numbers, precise object definitions, management endpoints, device identifiers, and security-remediation details. Operational configuration and rollback information belong in **private infrastructure documentation**.

---

## Security Objectives

The policy is designed to:

1. Restrict the `DEFAULT` network from ordinary production use.
2. Use `TRUSTED` as the primary source of administrative connections.
3. Separate infrastructure services from less-trusted workloads and clients.
4. Limit game-server workloads to the connectivity necessary for their operation.
5. Prevent IoT and guest devices from initiating general access to private networks.
6. Support Internet access where it is required.
7. Provide controlled access to shared infrastructure services, including DNS.
8. Support Home Assistant's interactions with IoT devices without generally opening the infrastructure zone.

These are policy objectives, not a claim that every possible network path has been exhaustively tested.

---

## Network Security Zones

| VLAN | Zone | Role |
|---:|---|---|
| 1 | `DEFAULT` | Restricted legacy/default zone |
| 10 | `INFRA` | Infrastructure and hosted services |
| 20 | `TRUSTED` | Trusted clients and administrative access |
| 25 | `GAME-DMZ` | Game-server workloads |
| 30 | `IOT` | Smart-home and IoT devices |
| 40 | `GUEST` | Guest clients |

Zone assignments reflect the different trust levels and communication requirements of the systems they contain. IP addressing is documented separately in [`ip-plan.md`](ip-plan.md).

### Trust Model

The design is **asymmetric**: an administrative client can initiate a connection to a managed system without giving that managed system equivalent permission to initiate unrelated connections back to the client network.

Segmentation also separates externally oriented game-server workloads and lower-trust IoT/guest devices from core services. A VLAN provides a logical boundary; **ACL policy is required to control traffic routed across that boundary**.

---

## Logical Communication Policy

The following matrix summarizes the **policy represented by the documented configuration**. It is deliberately a logical overview, not a line-by-line representation of gateway rules.

| Source / Destination | `INFRA` | `TRUSTED` | `GAME-DMZ` | `IOT` | `GUEST` | Internet |
|---|---|---|---|---|---|---|
| `DEFAULT` | Restrict | Restrict | Restrict | Restrict | Restrict | Restrict |
| `INFRA` | — | Block | Block | Service exception | Block | Allow |
| `TRUSTED` | Allow | — | Allow | Allow | Not explicitly permitted | Allow |
| `GAME-DMZ` | DNS exception | Block | — | Block | Block | Allow |
| `IOT` | DNS exception | Block | Block | — | Block | Allow |
| `GUEST` | DNS exception | Block | Block | Block | — | Allow |

**How to interpret the matrix:**

- **Allow:** General connectivity initiated by the source zone is permitted by the documented policy.
- **Block / Restrict:** General connectivity is denied or intentionally restricted by the documented policy.
- **DNS exception:** A limited service-specific path exists; this does not permit general access to the destination zone.
- **Service exception:** A specific application requires connectivity across this boundary.
- **Not explicitly permitted:** The source documentation does not establish an explicit allow; this must **not** be interpreted as a verified block.
- **—:** Traffic remaining within a single zone is outside this inter-VLAN matrix.

This matrix concerns **new traffic initiated by the source zone**. Return traffic and behavior within a VLAN depend on the applicable network controls. The matrix does not establish access to the gateway's own management interface.

---

## Required Service Exceptions

### DNS for isolated zones

IoT devices, guest clients, and game-server workloads need DNS resolution without unrestricted access to infrastructure systems. The design provides scoped access to the Pi-hole DNS service rather than opening the entire `INFRA` zone.

**Security rationale:** Preserve a necessary shared service while limiting the destinations and services reachable from lower-trust networks.

### Home Assistant and IoT

Home Assistant is hosted in `INFRA` and needs to initiate communication with supported IoT devices. A targeted exception supports this application relationship without allowing general `INFRA`-to-`IOT` communication.

**Security rationale:** Grant the application the required connectivity instead of extending that privilege to an entire infrastructure network.

### Administrative access

The `TRUSTED` zone serves as the primary origin for routine administration of infrastructure and supported workloads. The reverse direction does not inherit the same access automatically.

**Security rationale:** Keep administrative workflows functional while reducing unnecessary paths toward personal endpoints.

### Internet access

Selected operational and lower-trust zones need outbound Internet access for service operation, downloads, or updates. Internet access does not imply unrestricted inter-VLAN access.

---

## Implementation Principles

The implementation follows these principles:

- **Least privilege:** Prefer service-specific connectivity over broad inter-zone permissions.
- **Explicit exceptions:** Allow a required dependency without weakening an entire zone boundary.
- **Controlled evaluation:** Account for ACL evaluation order so narrowly scoped exceptions and general restrictions behave as intended.
- **Central enforcement:** Keep routed inter-VLAN policy primarily at the gateway to simplify operational responsibility.
- **Role-based objects:** Prefer maintainable groups and service roles over duplicating individual addresses throughout the policy.
- **Asymmetric access:** Do not assume that permission in one direction implies permission to initiate traffic in the other direction.

The exact rule sequence, object membership, protocol settings, and rollback procedures are maintained outside the public repository.

---

## Validation Summary

The source documentation records successful **functional checks of representative paths**. The results below summarize those recorded observations; they are **not fresh measurements performed for this rewrite**.

| Control area | Documented check | Recorded outcome |
|---|---|---|
| Isolated-zone connectivity | IoT, guest, and game-server zones can access needed Internet services | Passed |
| Shared DNS | Those zones can reach Pi-hole DNS | Passed |
| Internal segmentation | Attempts from IoT, guest, and game-server zones toward restricted internal zones | Blocked as intended |
| Administrative access | `TRUSTED` can initiate required connections to infrastructure and managed workloads | Passed |
| Reverse-path restriction | General `INFRA`-initiated access to selected client and workload zones | Blocked as intended |
| Application exception | Home Assistant can communicate with IoT devices | Passed |
| Exception isolation | General infrastructure-to-IoT access remains restricted | Passed |

### Validation Limits

The recorded checks support the behavior of the **tested paths**; they do not prove universal isolation or complete hardening. In particular, they do not, by themselves, establish:

- Protection of every management-plane endpoint
- Every protocol, port, device, and rule interaction
- Layer-2 isolation between devices on the same VLAN
- IPv6 filtering, host-based firewall behavior, or all alternative routing paths
- Continued correctness following future configuration changes

Test procedures, timestamps, evidence, and any unresolved findings should be tracked in private operational or assessment records. Publish sanitized test evidence separately when useful for demonstrating the methodology.

---

## Change Management

ACL changes can affect multiple workloads at once. Changes should follow a predictable process:

1. Define the service requirement and required source/destination roles.
2. Check whether the communication path is already permitted.
3. Select the narrowest practical exception.
4. Review rule interactions and ordering before applying the change.
5. Record the existing configuration and a rollback procedure **privately**.
6. Apply one logical change at a time.
7. Test both a path that **must work** and a path that **must remain restricted**.
8. Update the relevant public design notes without exposing operational secrets or unnecessary implementation details.

Changes to administrative access should be treated as particularly sensitive because an incorrect policy can interrupt remote management. Specific recovery paths and outstanding hardening work are recorded privately.

---

## Documentation Boundaries

### Appropriate for this public document

- Zone purpose and trust model
- High-level traffic policy and required service dependencies
- Design rationale and access-control principles
- Sanitized validation summaries and methodology
- General change-management practice

### Keep in private operational documentation

- Exact active ACL inventory, object definitions, and rule order
- Management IPs, device identities, and access paths
- Switchport-level enforcement details and configuration exports
- Unresolved vulnerabilities, exceptions, and remediation plans
- Detailed test logs or captures that identify infrastructure endpoints
- Configuration backup and recovery procedures

**Credentials, private keys, tokens, and recovery codes must be stored in an appropriate secrets manager—not in a public or ordinary private Git repository.**

---

## Related Documentation

- [`README.md`](README.md) — Network architecture overview
- [`vlan-design.md`](vlan-design.md) — Security-zone design and rationale
- [`ip-plan.md`](ip-plan.md) — Addressing and DNS design
- [`port-mapping.md`](port-mapping.md) — Physical and logical connectivity
- [`../security/segmentation.md`](../security/segmentation.md) — Segmentation security perspective

> **Repository consistency:** Other network documents should follow the same public/private information boundary. Links above are retained for compatibility; they do not imply that every linked document has already been sanitized.
