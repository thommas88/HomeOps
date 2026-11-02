# Incident Report: High CPU Usage Caused by NUT USB Driver

| Field | Details |
|---|---|
| Date | 2026-11-02 (CET) |
| Host | `prodesk` (Proxmox VE) |
| Component | Network UPS Tools (NUT), `usbhid-ups` driver |
| UPS | CyberPower CP1600EPFCLCD |
| Incident type | Host performance degradation / UPS USB communication errors |
| Status | **Mitigated** — CPU usage normalized; underlying USB error cause not confirmed |
| Service downtime | No VM or container downtime observed during remediation |

## Summary

The Proxmox host `prodesk` showed sustained CPU utilization of approximately **8–10%**, despite low reported utilization across its virtual machines and LXC containers. Host-level process inspection identified the NUT `usbhid-ups` driver consuming **89.4% of one logical CPU thread**.

System logs showed recurring USB HID communication errors (`Input/Output Error` and `Pipe error`). Restarting the dedicated NUT driver service restored normal host CPU utilization to approximately **1.7%**. Subsequent checks showed a successful driver startup and confirmed that the UPS was reporting an **online (`OL`)** state.

The immediate performance issue was resolved. The precise cause of the USB communication errors remains unverified.

## Environment

- **Hypervisor:** Proxmox VE host `prodesk`
- **Processor:** Intel Core i5-10500T (6 cores / 12 logical threads)
- **UPS:** CyberPower CP1600EPFCLCD, connected via USB
- **NUT version:** 2.8.1
- **Driver:** `usbhid-ups` (CyberPower HID 0.8 subdriver)
- **Systemd unit:** `nut-driver@cp1600.service`

## Detection and Investigation

1. **Node metrics:** Proxmox displayed approximately 10% CPU usage on `prodesk`. The history graph showed an elevated baseline compared with earlier readings.
2. **Guest review:** The running VMs and LXC containers individually reported low CPU usage. These metrics did not identify a guest as the obvious source.
3. **Host process inspection:** `top -o %CPU` identified `usbhid-ups` at **89.4% CPU** (one-thread basis). The host CPU summary showed **92.0% idle**, equivalent to about 8.0% utilization across the host at that sample.
4. **Log review:** The journal contained repeated messages such as:

   ```text
   usbhid-ups: nut_libusb_get_string: Input/Output Error
   usbhid-ups: nut_libusb_get_report: Input/Output Error
   usbhid-ups: nut_libusb_get_string: Pipe error
   usbhid-ups: nut_libusb_get_report: Pipe error
   ```

   The reviewed log excerpt included errors on November 1–2, 2026. The exact onset of the fault could not be established from this excerpt alone.

5. **UPS telemetry:** `upsc` returned live UPS values before remediation, showing that the device remained at least intermittently accessible despite the errors:

   | Metric | Observed value |
   |---|---|
   | `ups.status` | `OL` (on line power) |
   | `battery.charge` | `100` (%) |
   | `battery.runtime` | `975` seconds (estimated) |
   | `ups.load` | `32` (%) |
   | `input.voltage` | `242.0` V |

## Remediation

The following checks and change were performed on the **Proxmox host**, not inside a guest:

```bash
# Identify CPU-intensive processes
top -o %CPU

# Review NUT-related errors from the current boot
journalctl -b --no-pager | grep -Ei 'usbhid-ups|nut-driver' | tail -40

# Confirm that UPS readings can be retrieved
upsc cp1600

# Identify the dedicated driver service
systemctl list-units --type=service --all 'nut-driver*'

# Restart only the UPS driver
systemctl restart nut-driver@cp1600.service

# Verify operation after the restart
upsc cp1600 ups.status
journalctl -u nut-driver@cp1600.service --since '5 minutes ago' --no-pager
top -o %CPU
```

The restart affected only the UPS driver service; **the Proxmox host and its guests were not restarted**. UPS monitoring was briefly interrupted while the driver restarted. No changes were made to the NUT configuration.

## Verification and Outcome

| Check | Before | After |
|---|---|---|
| Host CPU utilization (sample) | ~8–10% | **1.7%** |
| Host idle CPU (`top`) | 92.0% | **98.3%** |
| `usbhid-ups` CPU | 89.4% of one logical thread | No longer among top CPU consumers |
| NUT driver logs | Repeated USB I/O and pipe errors | Successful startup; no new USB communication errors in immediate post-restart excerpt |
| UPS status | `OL` | **`OL`** |

The service journal reported a successful driver startup. It also reported approximately **1 month, 2 weeks, and 4 days of accumulated CPU time** for the previous service instance, consistent with substantial historical CPU consumption; this figure alone does not establish precisely when the problem started.

Messages concerning `upsnotify`/systemd notification integration appeared at startup but were not evidence of renewed USB I/O failures. The `fopen ... pid: No such file or directory` message occurred while stopping the previous driver instance; the unit subsequently stopped and restarted successfully.

## Root Cause Assessment

- **Confirmed:** `usbhid-ups` was responsible for most of the unexpectedly high CPU usage at the time of inspection.
- **Confirmed:** The driver had logged recurring USB communication errors.
- **Confirmed:** Restarting the driver eliminated the abnormal CPU consumption in the immediate verification sample.
- **Not confirmed:** Whether the underlying trigger was a USB cable or port issue, USB power management, firmware/device behavior, or a driver defect. A persistent software loop is plausible but was not independently proven.

**Conclusion:** The incident is **mitigated, not conclusively root-caused**. A restart was sufficient to restore normal operation at the time of testing.

## Follow-up / Recurrence Procedure

No further change is warranted while CPU usage and UPS telemetry remain normal. If the symptoms recur:

1. Check whether `usbhid-ups` is again using significant CPU and whether the NUT journal contains fresh USB errors.
2. Verify that `upsc cp1600 ups.status` returns current data; look for errors or stale telemetry.
3. Investigate the USB cable and host USB port, and check for USB resets/disconnects in kernel logs (`journalctl -k`).
4. Consider driver/USB polling options only after gathering repeatable evidence; do not introduce speculative configuration changes preemptively.

This report records a specific operational incident and its resolution, rather than a general recommendation to restart the NUT driver on a schedule.
