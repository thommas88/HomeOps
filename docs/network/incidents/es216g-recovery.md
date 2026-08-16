# ES216G Recovery Incident

**Device:** TP-Link Omada ES216G  
**Incident type:** Management connectivity / controller adoption failure  
**Status:** Resolved  
**Resolved:** 2026-08-08  

---

## Summary

During VLAN and network-hardening work, the management path between the TP-Link ES216G and the Omada Controller was disrupted.

The switch stopped communicating normally with the controller and eventually entered an adoption-failure state.

The incident was caused by a change to the uplink/VLAN configuration between the core SG3210X-M2 and the ES216G that interrupted the traffic required for switch management.

Recovery required local access to the ES216G, restoration of the correct management configuration, verification of the uplink trunk, and re-adoption into Omada.

The switch was ultimately restored to a normal `CONNECTED` state.

---

## Impact

The incident primarily affected **management of the ES216G**, rather than the entire homelab network.

Observed impact included:

- Loss of normal Omada Controller communication with the ES216G
- `Adopt Failed` state in Omada
- Temporary inability to manage the switch centrally
- Repeated recovery attempts and local configuration changes
- Risk of losing connectivity to devices behind the ES216G while troubleshooting

The ER605 gateway, core switch, and other functioning parts of the network were not the root cause of the failure.

---

## Relevant Topology

The affected management path was:

```text
Omada Controller
      │
      │ INFRA / VLAN 10
      ▼
SG3210X-M2
Port 6
      │
      │ trunk
      ▼
ES216G
Port 1
      │
      ▼
ES216G management interface
```

Normal uplink configuration:

```text
SG3210X-M2 Port 6
        │
        ▼
ES216G Port 1
```

Native VLAN:

```text
DEFAULT (1)
```

Tagged VLANs:

```text
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

The ES216G management interface itself resides on:

```text
INFRA / VLAN 10
```

---

## Context

The incident occurred while the network was being hardened after migration from a flat LAN to VLAN-based segmentation.

The work included:

- Restricting `DEFAULT`
- Disabling DHCP on `DEFAULT`
- Implementing Gateway ACLs
- Moving infrastructure management to `INFRA`
- Verifying trunk and management behavior

The intended goal was to improve segmentation and remove unnecessary reliance on the legacy/default network.

The ES216G uplink was part of the management path and therefore represented a high-risk configuration point.

---

## Triggering Change

The incident was triggered by a change to the uplink/VLAN configuration associated with:

```text
SG3210X-M2 Port 6
        ↕
ES216G Port 1
```

The change disrupted the path used by the ES216G to reach the Omada Controller.

The key issue was not that the physical link was absent, but that the management traffic no longer traversed the uplink correctly.

---

## Symptoms

The most visible symptoms were:

- ES216G lost normal controller connectivity
- Omada displayed `Adopting...`
- Adoption later failed
- Omada reported `Adopt Failed`
- Local management became necessary
- The switch reported that it was already managed by the existing Omada Controller

One observed message indicated:

```text
This device is being managed by Controller 192.168.10.100
```

This confirmed that the device retained knowledge of the controller while normal controller communication was still broken.

---

## Initial Problem State

At the point where the failure was being investigated:

- Omada Controller was operational on `INFRA`
- The SG3210X-M2 was operational
- The physical uplink between the switches existed
- The ES216G was no longer usable through normal centralized management
- `DEFAULT` DHCP had already been disabled
- ES216G therefore could not simply fall back to receiving an address from the old default network

This made local management access necessary.

---

## Recovery Approach

Recovery was performed by restoring the management path rather than redesigning unrelated parts of the network.

The final successful recovery consisted of the following steps.

---

### 1. Regain Local Management Access

Local access to the ES216G was established so the switch could be configured independently of Omada.

This was necessary because the normal controller path was unavailable.

---

### 2. Configure the Management VLAN

The ES216G management network was set to:

```text
INFRA / VLAN 10
```

This aligned the switch management interface with the rest of the network infrastructure.

---

### 3. Restore the Management Address

The ES216G management address was configured as:

```text
192.168.10.103/24
```

Gateway:

```text
192.168.10.1
```

This restored predictable Layer-3 management reachability on `INFRA`.

---

### 4. Preserve VLAN 10 Across the Uplink

The uplink was configured so that `INFRA / VLAN 10` continued to traverse the trunk as tagged traffic.

Final relationship:

```text
SG3210X-M2 Port 6
        │
        │ VLAN 10 tagged
        ▼
ES216G Port 1
```

The trunk retained:

```text
Native:
DEFAULT (1)

Tagged:
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

---

### 5. Verify Controller Reachability

Communication between:

```text
Omada Controller
192.168.10.100
```

and:

```text
ES216G
192.168.10.103
```

was verified before continuing with adoption.

This step established that the management path itself was functional again.

---

### 6. Re-Adopt the Switch

With network connectivity restored, the ES216G was adopted back into the Omada Controller using the correct local device credentials.

The adoption completed successfully.

---

## Final State

After recovery:

| Component | State |
|---|---|
| ES216G | `CONNECTED` |
| Management VLAN | `INFRA / VLAN 10` |
| Management address | `192.168.10.103` |
| Uplink | ES216G Port 1 ↔ SG3210X-M2 Port 6 |
| Controller | Reachable |
| Omada management | Operational |

The incident was considered resolved.

---

## Validation

Recovery was not considered complete when the switch merely appeared in Omada.

The surrounding network behavior was also verified.

Confirmed after recovery:

- ES216G visible as `CONNECTED` in Omada: ✅
- ES216G management reachable on `INFRA`: ✅
- Controller communication restored: ✅
- Inter-switch trunk operational: ✅
- PlayStation 5 connectivity retained: ✅
- Samsung TV connectivity retained: ✅
- JetKVM connectivity retained: ✅
- VLAN segmentation remained operational: ✅
- Pi-hole DNS remained available where required: ✅
- Internet connectivity remained available for the intended zones: ✅
- TRUSTED administration path remained functional: ✅

---

## Root Cause

The root cause was a **management-path configuration error on the inter-switch uplink**.

The ES216G management interface depended on `INFRA / VLAN 10` traversing the trunk between the SG3210X-M2 and the ES216G.

A change to the uplink/VLAN configuration interrupted that traffic.

As a result:

```text
Physical link:        available
Controller:           available
Switch:               available
Management path:      broken
```

The failure therefore presented as an Omada adoption/controller problem even though the underlying issue was VLAN transport on the management path.

---

## Contributing Factors

Several conditions made recovery more difficult.

### Management Traffic Shared the Uplink Being Modified

The same uplink being changed was also carrying the traffic required to manage the downstream switch.

This meant a configuration mistake could immediately remove normal administrative access.

---

### DEFAULT DHCP Was Already Disabled

The network-hardening work had correctly removed DHCP from `DEFAULT`.

However, during recovery this meant the switch could not rely on the old network for automatic addressing.

Local management and explicit configuration were required.

---

### Controller State Could Be Misleading

The switch could still retain controller association information while being unable to communicate correctly with the controller.

Messages such as:

```text
This device is being managed by Controller ...
```

did not prove that the active controller communication path was working.

---

## What Worked

The successful troubleshooting approach was to reduce the problem to the management path:

```text
Controller
    ↓
Core switch
    ↓
Inter-switch trunk
    ↓
ES216G management VLAN
    ↓
ES216G management IP
```

Each component of that path could then be verified individually.

The decisive actions were:

- obtaining local switch access
- restoring management VLAN 10
- restoring the known management address
- ensuring VLAN 10 remained tagged over the trunk
- verifying controller-to-switch reachability before re-adoption

---

## What Should Be Avoided

The incident demonstrated why unrelated working infrastructure should not be changed while troubleshooting a known management-path failure.

Changing additional components such as:

- the gateway
- unrelated switch ports
- unrelated VLANs
- working services

would increase the number of variables without addressing the actual failure.

Once the fault domain is understood, troubleshooting should remain inside that boundary until evidence points elsewhere.

---

## Lessons Learned

### 1. Management Paths Are Critical Dependencies

A switch uplink may carry both production traffic and the traffic required to manage the switch itself.

Management connectivity must therefore be treated as an explicit dependency before changing:

- native VLANs
- tagged VLANs
- trunks
- management VLANs
- uplink profiles

---

### 2. One Change at a Time

Management and VLAN changes should be made individually.

Preferred process:

```text
Record current state
        ↓
Make one change
        ↓
Verify management
        ↓
Verify required traffic
        ↓
Continue
```

Multiple simultaneous changes make rollback and root-cause identification much harder.

---

### 3. Record the Working Trunk Before Modification

Before changing an inter-switch trunk, record:

- both physical ports
- native VLAN
- tagged VLANs
- management VLAN
- management address
- expected controller path

This turns recovery from reconstruction into rollback.

---

### 4. Maintain a Local Recovery Path

A centrally managed switch should have a known local recovery procedure.

For the ES216G, this includes knowing how to:

- reach the local interface
- set a temporary workstation address if required
- configure management VLAN
- configure management IP
- restore controller reachability

---

### 5. Do Not Confuse Adoption Failure With Root Cause

`Adopt Failed` describes the visible failure in Omada.

It does not necessarily mean the adoption mechanism itself is the underlying problem.

In this incident, adoption failed because the management network path was incorrect.

---

### 6. Verify Services After Network Recovery

Recovery should include more than:

```text
ping works
```

The actual downstream environment must also be checked.

Examples:

- controller connectivity
- DNS
- client Internet access
- VLAN isolation
- management interfaces
- downstream devices

---

## Preventive Actions

The following operational rules were adopted after the incident.

### Before Management/Uplink Changes

- Document the current working state
- Identify the exact management path
- Confirm native and tagged VLANs
- Identify the physical uplink at both ends
- Prepare a rollback procedure
- Ensure local management access is possible

### During Changes

- Change one item at a time
- Do not modify unrelated working infrastructure
- Verify management connectivity immediately
- Stop if the expected result is not obtained

### After Changes

Verify:

- switch/controller communication
- management access
- DHCP where applicable
- DNS
- Internet access
- intended inter-VLAN permits
- intended inter-VLAN blocks
- downstream clients

---

## Current Safeguards

The current documented uplink state is:

```text
SG3210X-M2 Port 6
        ↕
ES216G Port 1
```

```text
Native VLAN:
DEFAULT (1)

Tagged VLANs:
INFRA (10)
TRUSTED (20)
GAME-DMZ (25)
IOT (30)
GUEST (40)
```

ES216G management:

```text
VLAN: INFRA (10)
IP:   192.168.10.103
```

This configuration is now documented in:

[`../port-mapping.md`](../port-mapping.md)

Any future modification should use that document as the starting state.

---

## Incident Outcome

The incident resulted in:

- Successful ES216G recovery
- Restored Omada management
- A verified inter-switch trunk
- A documented management path
- Better change-control rules
- Clearer separation between root-cause troubleshooting and unrelated infrastructure changes

The failure did not require abandoning the VLAN design.

Instead, it exposed an operational weakness in how a management-critical change was performed and documented.

The resulting process is now safer and more repeatable.

---

## Public Repository Notes

This incident report intentionally excludes:

- switch credentials
- controller credentials
- MAC addresses
- private authentication information
- Tailscale information
- Wi-Fi credentials
- tokens and keys

Private RFC1918 addresses are retained only where they are required to explain the internal management path.

---

## Related Documentation

- [`../README.md`](../README.md) — network overview
- [`../vlan-design.md`](../vlan-design.md) — VLAN architecture and trust model
- [`../ip-plan.md`](../ip-plan.md) — internal addressing
- [`../acl-policy.md`](../acl-policy.md) — Gateway ACL policy
- [`../port-mapping.md`](../port-mapping.md) — switch ports and trunk configuration
- [`../../architecture/physical-topology.md`](../../architecture/physical-topology.md) — physical homelab topology
