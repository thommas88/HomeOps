# Switch Port Design

> **Public architecture documentation** — This file describes switch roles, link types, VLAN transport principles, and operational safeguards in the HomeOps network. It deliberately omits the live port-to-device inventory and configuration needed to administer or reconstruct the physical network.
>
> **Filename note:** The file remains `port-mapping.md` to preserve existing links. The authoritative, device-specific port map is maintained privately.

## Purpose and Scope

The homelab uses managed Ethernet switching to connect a Proxmox-based infrastructure, a routed gateway, wireless clients, and local endpoints. Different device classes require different port profiles, and VLAN membership is used to keep management traffic and application workloads logically separated.

This public document covers:

- Physical switching roles and high-level connectivity
- Access and trunk port design
- General VLAN transport and management-plane separation
- Unused-port handling and configuration hygiene
- Safe change-control and verification practices

It does **not** publish physical port numbers, individual port-to-device assignments, exact native/tagged VLAN combinations on production links, live port status, MAC addresses, or device management endpoints.

## Switching Components

| Component | Network role | Purpose |
|---|---|---|
| TP-Link SG3210X-M2 | Core switch | Aggregates infrastructure hosts, the gateway connection, wireless networking, and downstream switching |
| TP-Link ES216G | Access switch | Extends wired connectivity for local endpoints and selected management devices |
| TP-Link ER605 V2 | Gateway/router | Provides VLAN gateways, routing, DHCP, NAT, and gateway ACL enforcement |
| TP-Link EAP670 | Wireless access point | Connects wireless clients to their assigned network zones |

The switches are managed through the Omada environment. The switch model names identify the technologies used, not specific active interfaces or administrative addresses.

## High-Level Topology

```text
                     ER605 V2
                    Gateway
                        |
                  Core switching
                  SG3210X-M2
                        |
             +----------+-----------+
             |          |           |
         Proxmox      EAP670     Access switch
          hosts        Wi-Fi        ES216G
             |          |           |
         Hosted       Wireless     Wired
         services     clients      endpoints
```

This is a **logical connectivity sketch**, not a wiring diagram. It intentionally excludes cable identifiers, specific switch ports, host-to-port assignments, and exact VLAN memberships on individual links.

For the broader infrastructure, see [`../architecture/physical-topology.md`](../architecture/physical-topology.md).

---

## Port Profiles

### Access Ports

An **access port** connects an endpoint that does not need to perform VLAN tagging itself. Its traffic is associated with one designated security zone at the switch.

Typical endpoint categories include:

- Trusted workstations and personal devices
- Media and smart-home devices
- Dedicated management appliances

Assign each endpoint according to its function and trust level. An endpoint being physically connected to the same switch as an infrastructure host does **not** give it the same network permissions.

### Trunk Ports

A **trunk port** carries traffic for more than one VLAN where required by an upstream or downstream device.

Trunking is used conceptually for:

| Link category | Reason for trunking |
|---|---|
| Gateway uplink | Carry traffic for routed network zones |
| Inter-switch uplink | Extend required VLANs to the access-switch layer |
| Proxmox host uplink | Separate hypervisor management from hosted workload networks |
| Wireless AP uplink | Support management and client networks on the same physical connection |

Trunks should carry **only the VLANs required by the connected device and its approved workloads**. Avoid using an unrestricted, all-VLAN trunk as a convenience.

The live native-VLAN choice, tagged-VLAN allowlist, and physical interface assignment for each trunk are intentionally excluded from this public document.

---

## Management and Workload Separation

Network design separates infrastructure administration from less-trusted client and service traffic.

- **Hypervisors:** Management interfaces use the infrastructure security zone; virtual machines and containers may use separate workload VLANs.
- **Game servers:** Application workloads are placed in the `GAME-DMZ` security zone rather than sharing management access by default.
- **Wireless:** Client networks are associated with the appropriate security zones independently of access-point administration.
- **Management devices:** Devices used to administer other systems are treated as sensitive infrastructure endpoints.

These statements describe the **documented architecture and design intent**. They are not proof that every interface or flow is configured correctly. Inter-zone reachability depends on routing and ACL enforcement in addition to VLAN tagging.

The security-zone definitions are maintained in [`vlan-design.md`](vlan-design.md), and the routed access model is documented in [`acl-policy.md`](acl-policy.md).

## Unused Port Policy

The source port inventory records unused ports as **disabled**. This is a documented configuration state from the source material, not a fresh verification against the switches.

Operational practice is to:

1. Keep unused ports disabled where supported and practical.
2. Remove obsolete access or trunk profiles before reusing an interface.
3. Enable a port only after identifying its intended endpoint and required zone.
4. Apply the least-permissive profile that supports the device.
5. Verify connectivity after activation and update the private port inventory.

This reduces accidental misconfiguration and unnecessary physical access paths.

## Omada Client Identification Caveat

A managed switch's client view does not necessarily identify the device **physically connected** to a port.

When a port connects to a Proxmox hypervisor, Omada may display a hosted VM or container as the observed client because the virtual workload generates traffic through that uplink.

For physical troubleshooting, distinguish between:

- The **physical link** and attached host
- The **observed MAC/client** in the management interface
- The **logical VLAN or VM interface** producing traffic

The private, verified cabling inventory is the source of truth for physical connections. Controller client discovery alone should not be treated as definitive port-mapping evidence.

---

## Change-Control Procedure

Changes to switch profiles can affect production networking and remote management. A controlled procedure is particularly important for gateway uplinks, inter-switch connections, hypervisor uplinks, and access-point management.

### Before a Change

1. Identify the intended device, physical interface, and existing port profile in **private operational records**.
2. Confirm whether access or trunk mode is required.
3. Record existing native and tagged VLAN settings privately.
4. Check which management paths and hosted services may depend on the link.
5. Establish a tested rollback or local recovery method.
6. Make one bounded change at a time.

### After a Change

1. Confirm link state and expected VLAN membership.
2. Verify the management path remains available.
3. Test representative permitted and prohibited traffic paths, where appropriate.
4. Verify affected services, including virtual workloads and DNS dependencies.
5. Record results and update the **private** port map.

Do not interpret successful connectivity to one service as proof that unrelated isolation rules are enforced. Tests should correspond to the intended policy.

## Verification Boundaries

The published topology describes the network at an architectural level. It does not replace:

- A live Omada configuration review
- A validated physical port and cable inventory
- VLAN-tagging checks on actual interfaces
- End-to-end connectivity and isolation testing
- Configuration backups and a documented recovery procedure

Changes made in Omada after a documentation update may introduce drift. The private inventory should be reconciled against the deployed configuration when maintenance occurs.

---

## Public / Private Documentation Boundary

| Public HomeOps repository | Private operational documentation |
|---|---|
| Switch models and functional roles | Actual port-to-device and cable map |
| Access/trunk design principles | Port numbers and active per-port profiles |
| VLAN architecture and zone intent | Exact native/tagged VLAN configuration for each link |
| General unused-port and change-control practices | Live enabled/disabled status and configuration exports |
| Troubleshooting concepts | Management endpoints, detailed rollback instructions, and sensitive incident data |

Credentials, API tokens, private keys, and recovery secrets should be stored in an appropriate secrets manager—not in either version of this Markdown file.

**Repository-history note:** Replacing the current file does not remove earlier port details from existing Git commits. If historical information needs to be removed, assess Git-history cleanup separately and account for public forks, clones, and caches.

## Related Documentation

- [`README.md`](README.md) — network documentation overview
- [`vlan-design.md`](vlan-design.md) — VLAN purpose and security-zone model
- [`ip-plan.md`](ip-plan.md) — IPv4 address plan and DHCP design
- [`acl-policy.md`](acl-policy.md) — high-level inter-VLAN access-control policy
- [`../architecture/physical-topology.md`](../architecture/physical-topology.md) — broader physical infrastructure topology
- [`../architecture/logical-topology.md`](../architecture/logical-topology.md) — virtual infrastructure and service relationships

---

**Documentation status:** Public, intentionally abstracted architecture overview. Not an authoritative live port inventory.
