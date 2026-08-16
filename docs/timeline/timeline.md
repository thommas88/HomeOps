# Project Timeline – Homelab NAS

## 13.08.25 - Initial Setup: First Proxmox Installation
**Goal:** Deploy a Proxmox hypervisor to host a Windows VM for running a Satisfactory server and sharing local HDDs.
**Actions:** Installed Proxmox VE 8 on a 230 GB SSD. This was the first time working with Proxmox or Linux.
**Result:** Successful installation and initial host configuration completed. 
**Learning:** Introduced to virtualization concepts, VM lifecycle management, and the Proxmox web interface.

---

## 23.08.25 - System Design: Planning the VM Structure
**Goal:** Define a virtual machine layout to organize system roles and improve scalability.  
**Actions:** Explored Proxmox capabilities in depth and planned three dedicated VMs:  
- **TrueNAS** for centralized storage and backups  
- **Immich** to replace Google Photos for media management  
- **Windows VM** for SteamCMD game servers and testing  
Evaluated existing hardware limitations and ordered two additional HDDs and a new case to support the setup.  
**Result:** Established a clear VM architecture with logical separation of services.  
**Learning:** Isolating roles across individual VMs improves maintainability, flexibility, and system stability.

---

## 29.08.25 - Hardware Upgrade: Expanding Capacity and Cooling
**Goal:** Upgrade the physical setup to support additional drives and improve thermal management.  
**Actions:**  
- Replaced the Corsair 4000D Airflow case with a Fractal Define 7 to increase HDD capacity.  
- Installed **2×8 TB Seagate IronWolf NAS drives** for TrueNAS.  
- Upgraded system memory from **32 GB → 64 GB** to improve VM performance.  
- Reassigned the existing **2×2 TB drives** for use by the Immich photo backup server.  
**Result:** The host now supports more drives with improved airflow and cooling, providing a stable foundation for the NAS environment.  
**Learning:** Proper hardware planning including space, cooling, and memory—is critical when scaling a system with multiple storage devices.

---

## 31.08.25 - Deploying TrueNAS and Windows VMs
**Goal:** Establish a dedicated NAS environment with ZFS storage and shared network access.  
**Actions:**  
- Created a **TrueNAS VM** in Proxmox (4 vCPU, 16 GB RAM, 64 GB system disk).  
- Passed through **2×8 TB drives** and configured a **ZFS mirror pool** (~7.1 TiB usable).  
- Created datasets: `Files`, `Photos`, `Videos`.  
- Configured SMB shares and verified access from a Windows client.  
- Added a **“daily” user** with read-only permissions for security, avoiding reliance on the admin account.  
- Switched from DHCP to a **static IP** with a reserved lease on the router.  
**Result:** Functional TrueNAS instance with accessible SMB shares and stable IP configuration.  
**Learning:** Using datasets instead of sharing the pool root simplifies management and improves security. Gained experience with ACLs and permission handling in TrueNAS.

---


## 05.09.25 - Testing Reboot and Mount Persistence
**Goal:** Verify that all VMs and services restart correctly after a full system reboot.  
**Actions:**  
- Stopped all VMs and restarted the Proxmox host to test automation.  
- Discovered that **Immich** was not mounting the 2×2 TB ZFS pool and had defaulted to the 70 GB system disk.  
**Result:** Identified mount timing issue affecting Docker/VM storage paths; corrective action scheduled.  
**Learning:** Testing reboots is critical to confirm that mounts persist. If ZFS datasets are not available on startup, dependent services may revert to local storage-standardizing mounts prevents data loss.

---

## 08.10.25 - Transition from Windows to Linux Game Hosting
**Goal:** Replace Windows-based game servers with a Linux environment for improved stability and automation.  
**Actions:**  
- Installed Ubuntu Server and migrated all game-hosting services.  
- Deployed **Valheim** and **Minecraft** servers on the new Linux VM.  
- Added a **3 TB HDD** to TrueNAS and configured it as a shared network “Workdrive.”  
**Result:** All game servers running under Linux with centralized storage for backups and configuration files.  
**Challenge:** Encountered connection issues due to university network restrictions, preventing friends from joining externally.

---

## 20.10.25 - Resolving Connectivity and Implementing Backups
**Goal:** Enable external access to game servers despite restrictive university network policies.  
**Actions:**  
- Experienced issues with external players being unable to connect.  
Tested **Tailscale** as a potential solution, but its peer-to-peer access model limits connections to authenticated users within the same network, making it unsuitable for public game hosting.   
- Switched to **Playit.gg**, which successfully established a secure tunnel for external connections.  
- Configured **Rclone** to back up Minecraft world data to Google Drive, ensuring data protection in case of downtime or configuration errors.  
**Result:** External connectivity fully functional through Playit.gg, with automated cloud backups for the Minecraft server.  
**Learning:** Peer-to-peer VPNs like Tailscale are secure but restrictive for public hosting. Tunneling services such as Playit.gg offer an easier, controlled way to expose servers safely while automated backups protect player progress.

---

## 01.11.25 - Restoring and Automating Immich on 2×2 TB Disks
**Goal:** Resolve the previous mount issue and bring Immich fully back online with automated storage handling.  
**Actions:**  
- Deployed a dedicated **Ubuntu VM** for Immich using Docker and Docker Compose.  
- Corrected the earlier mount configuration so Immich now properly uses the **2×2 TB ZFS pool** instead of the system disk.  
- Verified functionality by uploading photos from mobile over local Wi-Fi (home router).  
- Added startup automation to ensure ZFS mounts are ready before Docker starts.  
**Result:** Immich restored and fully operational, automatically syncing photos from mobile devices.  
**Learning:** Addressing mount dependencies during boot prevents service failures. Automating mount readiness made the setup stable and maintenance-free.

---

## 26.10.25 - HP EliteDesk G5 (Secondary Node / Backup Infrastructure)

**Goal:**  
Deploy a secondary low-power node to separate backup workloads from the primary Proxmox host and improve redundancy in the homelab environment.

**Actions:**  
- Deployed HP EliteDesk G5 as dedicated infrastructure node.
- Installed Proxmox Backup Server (PBS) for centralized VM backup handling.
- Configured datastore for VM backup storage.
- Connected node to core network with static IP configuration.
- Verified backup job connectivity from primary Proxmox host.
- Tested snapshot-based backups and restore validation.
- Segregated backup workload from main compute/storage host (TrueNAS VM).

**Result:**  
Dedicated backup node operational with isolated backup workload handling. VM snapshots and scheduled backups stored independently from primary NAS storage.

**Learning:**  
- Separating backup infrastructure reduces risk in single-node homelab environments.
- Proxmox Backup Server provides efficient incremental backups via deduplication.
- Backup verification (restore testing) is critical—successful backup ≠ guaranteed recovery.
- Infrastructure layering (Compute vs Storage vs Backup node) improves architectural clarity and resilience.

---

## 21.02.26 - ZFS Pool Redesign and HBA Migration

**Goal:**  
Redesign storage architecture from a 2×8TB ZFS mirror to a 4×8TB RAIDZ1 configuration for improved storage density and scalability.

**Actions:**  
- Physically installed LSI SAS2308 HBA (IT-mode) for expanded SATA connectivity.
- Migrated all 8TB drives from motherboard SATA to HBA controller.
- Verified disk health using SMART (`smartctl -a`) before reconfiguration.
- Confirmed stable disk mapping via `/dev/disk/by-id` passthrough in Proxmox.
- Detached legacy `pool1` after ensuring dependent services were migrated.
- Created new ZFS pool `tank` using 4×8TB RAIDZ1 (~21.8 TiB usable).
- Preserved separate pools:
  - `immich` (2×2TB mirror)
  - `pool_3tb` (single-disk workdrive)
- Validated pool integrity with `zpool status` and scrub verification.
- Rebuilt dataset structure and prepared network share layout.

**Result:**  
Stable 4×8TB RAIDZ1 pool operational in TrueNAS VM with raw disk passthrough via Proxmox. All disks online and verified healthy.

**Learning:**  
- RAIDZ1 provides significantly better storage efficiency compared to mirrors but increases resilver duration on large disks.
- Using `/dev/disk/by-id` prevents device reordering issues after reboot.
- ZFS pool deletion requires detaching system datasets and active SMB dependencies first.
- Physical disk migration requires both hardware verification and logical validation inside the hypervisor layer.


---


## 28.02.26 – ThinkCentre M900 (Offsite Monitor Node med Tailscale + Uptime Kuma)

**Goal:**  
Deploy a dedicated low-maintenance monitoring node at a separate location (“Adresse 2”) to verify real-world availability of the homelab without port-forwarding or subscriptions.

**Actions:**  
- Configured BIOS for resilience:  
  - **AC Power Recovery: ON** (auto-boot after power loss)  
  - **Wake-on-LAN: Enabled** (optional remote wake)  
  - **Deep Sleep: Disabled** (prevent unexpected low-power states)  
- Installed **Debian 12 (minimal)**, enabling only **SSH** during setup.  
- Updated system packages and installed **Docker** + **Docker Compose** for lightweight service deployment.  
- Deployed **Uptime Kuma** via Docker Compose (`/opt/uptime-kuma`) with persistent data volume.  
- Installed and authenticated **Tailscale** to connect the node to the tailnet (no port-forwarding).  
- Verified resilience:  
  - Docker enabled on boot and confirmed active after reboot.  
  - Uptime Kuma restart policy validated and confirmed healthy after reboot.  
  - Tailscale auto-reconnected and retained its 100.x IP after reboot.  
- Expanded monitoring scope by installing **Tailscale on the Proxmox host** and monitoring Proxmox via HTTPS (port 8006) over the tailnet.  
- Tuned monitors and alerts (60s interval, retries, Discord notifications) to reduce false positives and avoid notification spam.  
- Increased history retention to support longer-term uptime tracking (e.g., 365 days).

**Result:**  
A stable offsite monitoring node that survives reboots/power events and can reach internal services over Tailscale. Provides independent “outside the network” verification of uptime for Proxmox and key VMs.

**Learning:**  
- Minimal OS + Docker + Tailscale is a robust pattern for “set-and-forget” infrastructure monitoring.  
- BIOS resilience settings matter as much as software when the node is offsite and unattended.  
- Alert tuning is crucial: fewer, higher-quality alerts are more useful than frequent spam.


---


## 10.03.26 - Immich Rebuild on Debian VM with TrueNAS NFS Storage

**Goal:**  
Rebuild Immich into a cleaner and more robust architecture where the application runs in its own VM, the database stays local to the VM, and photo/video storage is moved off the VM disk onto dedicated TrueNAS-backed mirrored storage.

**Actions:**  
- Created a fresh **Debian 12 VM** in Proxmox for Immich with **4 vCPU, 8 GB RAM, and 80 GB disk**.  
- Installed a minimal Debian system with **SSH only**, then added **Docker Engine**, **Docker Compose**, and **qemu-guest-agent**.  
- Deployed Immich under **`/opt/immich`** using Docker Compose and configured the upload path to **`/srv/immich`**.  
- Confirmed the initial Immich deployment worked, then paused before real use after identifying that media would otherwise be stored on the VM disk.  
- Investigated the old 2×2 TB Immich disks in TrueNAS and found stale metadata from a previous exported/faulted ZFS pool.  
- Verified that only the two 2 TB disks were targeted and cleared their labels/signatures without touching the other pools.  
- Created a brand-new TrueNAS mirror pool **`immichpool`** and dataset **`immich-data`** for clean Immich storage.  
- Configured a new **NFS share** for **`/mnt/immichpool/immich-data`** and mounted it inside the Immich VM at **`/srv/immich`**.  
- Troubleshot multiple NFS issues, including:  
  - incorrect TrueNAS target IP during mount attempts  
  - missing network access rules on the NFS share  
  - `root_squash` preventing Immich from writing required marker files  
  - missing writable `.immich` integrity files in storage subfolders  
- Fixed final permission issues by setting ownership to **UID/GID 1000:1000** on the dataset and changing the NFS export to **`no_root_squash`**.  
- Made the NFS mount persistent through **`/etc/fstab`** and validated it after reboot.  
- Verified end-to-end functionality with:
  - successful Immich startup  
  - successful web access  
  - successful test upload  
  - files appearing under **`/srv/immich/upload`**  
  - all containers reaching **healthy** state after reboot

**Result:**  
Immich is now running successfully in a dedicated Debian VM with **local database storage** and **media stored on a TrueNAS-backed mirrored NFS share**. The setup survives reboot, passes upload testing, and no longer relies on the VM disk for photo/video storage.

**Learning:**  
- Separating **application/database storage** from **bulk media storage** is a much safer Immich design in a homelab.  
- Reused ZFS disks can leave behind misleading pool metadata, so validation via CLI is essential before rebuilding storage.  
- NFS troubleshooting should always verify the basics first: **correct IP, export state, network access, and squash behavior**.  
- Immich is strict about storage integrity and may fail startup if its expected `.immich` marker files are missing or not writable.  
- A deployment is not really complete until it passes:  
  - mount validation  
  - upload validation  
  - reboot validation  
  - container health validation


---


## 11.03.26 - Omada Controller Deployment and Device Adoption

**Goal:**  
Deploy a centralized Omada management platform to simplify future VLAN setup, improve visibility across network devices, and move from standalone device management to a more structured controller-based network design.

**Actions:**  
- Evaluated **VM vs container** for Omada Controller and selected a **dedicated Debian 12 VM** in Proxmox for better isolation, simpler backups, and easier troubleshooting.  
- Created the VM with a lightweight server profile:  
  - **2 vCPU**  
  - **4 GB RAM**  
  - **30 GB disk**  
  - **VirtIO networking on vmbr0**  
- Installed **Debian 12 minimal** without a desktop environment.  
- Configured the VM hostname as **`omada`** for short and practical SSH / terminal access.  
- Reserved a **static DHCP lease** on the router to give the controller a consistent management IP.  
- Installed and verified:  
  - **qemu-guest-agent**  
  - **OpenJDK 17**  
  - **JSVC**  
  - **MongoDB 8**  
- Downloaded and installed **Omada Network Application v6.1.0.19** on the Debian VM.  
- Verified successful access to the Omada web interface over HTTPS from the local network.  
- Completed the initial Omada setup wizard with:
  - local controller account
  - cloud access disabled
  - site name set to **`homelab`**
  - application scenario set to **Home**
- Attempted adoption of the existing **ER605 v2.20 gateway** and **SG3210X-M2 v1.0 switch**.  
- Resolved initial **Adopt Failed** state by retrying adoption with the devices’ existing standalone credentials.  
- Successfully adopted both devices into the controller and verified that both now show **Connected** in the Omada device list.

**Result:**  
A fully operational **Omada Controller VM** is now running in Proxmox, with both the **ER605 gateway** and **SG3210X-M2 switch** centrally managed through the controller. The network core has been moved from standalone configuration to controller-based management without disrupting connectivity.

**Learning:**  
- A dedicated VM is a clean and robust way to host Omada Controller in a homelab environment.  
- Minimal Debian with only required dependencies provides a lean and maintainable platform.  
- Existing Omada devices may fail adoption initially if the controller is not given the devices’ current standalone credentials.  
- Adopting the **switch first** was a safer and more controlled approach before attempting gateway adoption.  
- Centralized management is now in place, creating a much better foundation for future **VLAN design**, **port profiles**, and network segmentation.


---


## 12.03.26 - Home Assistant VM Deployment and Zigbee Integration

**Goal:**  
Deploy a dedicated Home Assistant VM in Proxmox and establish a stable Zigbee stack for future smart home integration in the homelab.

**Actions:**  
- Created a dedicated **Home Assistant OS VM** on the primary Proxmox host.  
- Configured the VM with **q35**, **OVMF (UEFI)**, **2 vCPU**, **4 GB RAM**, and **32 GB disk**.  
- Imported the official **Home Assistant OS qcow2** image instead of using a traditional ISO installer.  
- Resolved UEFI boot issues by disabling pre-enrolled Secure Boot keys on the EFI disk.  
- Assigned the VM a **DHCP reservation** to keep a stable IP address (`192.168.0.105`).  
- Created an initial Proxmox backup after successful deployment.  
- Attached the **Sonoff Zigbee 3.0 USB Dongle Plus V2** using **USB passthrough**.  
- Installed and configured **Mosquitto broker** and **Zigbee2MQTT**.  
- Configured Zigbee2MQTT with a stable `/dev/serial/by-id/...` path, **ember** adapter, and MQTT connection through `mqtt://core-mosquitto:1883`.  
- Verified Zigbee functionality by pairing the first device, an **IKEA STARKVIND air purifier**, and exposing it to Home Assistant via MQTT discovery.

**Result:**  
Home Assistant is now running as a dedicated VM in Proxmox with a working Zigbee stack based on Sonoff + Zigbee2MQTT. MQTT integration is functional, and the first Zigbee device has been successfully onboarded and exposed inside Home Assistant.

**Learning:**  
- A dedicated **VM** is more robust than CT/LXC for Home Assistant when USB passthrough and appliance-style management are required.  
- Importing the official Home Assistant disk image is cleaner than using a standard ISO workflow.  
- UEFI/Secure Boot settings in Proxmox can prevent boot until configured correctly.  
- Using `/dev/serial/by-id/...` is more reliable than `/dev/ttyUSB0` for persistent Zigbee adapter mapping.  
- Zigbee2MQTT provides a flexible foundation for mixed-vendor Zigbee devices such as IKEA and Philips Hue without relying on separate bridges.

---

## 14.05.26 - JetKVM Out-of-Band Server Management

**Goal:**
Add an independent remote-management path to the primary homelab server for troubleshooting situations where Proxmox, the host operating system, or normal network management is unavailable.

**Actions:**

* Acquired and installed a **JetKVM** hardware KVM solution.
* Connected the device to the server’s HDMI output and USB interface for remote keyboard and mouse control.
* Connected JetKVM to the managed network infrastructure.
* Verified remote access to:

  * BIOS and UEFI configuration
  * Proxmox boot output
  * Local host console
  * Recovery and troubleshooting environments
* Assigned the device a documented network identity and switch port.
* Added JetKVM to the physical topology and network inventory.

**Result:**
The primary server can now be accessed remotely at the hardware-console level, even when Proxmox or the normal host network stack is unavailable.

**Learning:**

* SSH and web interfaces are insufficient when a host fails before the operating system starts.
* Out-of-band management provides access to BIOS, boot menus, encryption prompts, and local recovery consoles.
* Hardware-level remote access substantially reduces the need for physical intervention during upgrades and failures.


---

## 13.07.26 - Dedicated Proxmox Game Server Node

**Goal:**
Deploy a third Proxmox node dedicated to game hosting, separating public-facing and resource-intensive game services from the primary infrastructure host.

**Actions:**

* Deployed a dedicated Proxmox VE game-server node using separate hardware and a 1 TB SSD.
* Configured the host as `pve-game-server`, later using the shorter hostname `gameserver`.
* Created a dedicated Linux VM for hosting multiple game servers.
* Migrated the existing **Palworld Dedicated Server** to the new Linux environment.
* Configured the Palworld installation under `/opt/palworld/server`.
* Created and enabled a dedicated `palworld.service` systemd unit.
* Retained **Playit.gg** as the external connectivity solution because the university network does not allow traditional port forwarding.
* Verified that the server started automatically and was reachable externally through the existing tunnel.

**Result:**
The homelab now consists of three Proxmox nodes with clearer workload separation:

* Primary Proxmox node for storage and core services
* EliteDesk node for backup, DNS, monitoring, and lightweight infrastructure
* Dedicated game-server node for externally accessible game workloads

**Learning:**

* Separating game servers from core infrastructure reduces the impact of updates, crashes, and resource spikes.
* Dedicated systemd services provide more predictable startup and recovery than manually launched server processes.
* A separate game-server node also makes future firewall and VLAN isolation significantly easier.

---

## 16.07.26 - Proxmox VE 9 Upgrade Readiness and Infrastructure Audit

**Goal:**
Prepare the primary Proxmox host for a future upgrade from Proxmox VE 8 to Proxmox VE 9 while documenting the current host configuration.

**Actions:**

* Updated the existing Proxmox VE 8 installation and reviewed package status.

* Ran the full Proxmox upgrade compatibility check using:

  ```bash
  pve8to9 --full
  ```

* Completed the compatibility check without critical blockers.

* Documented the primary host network configuration, including `vmbr0` and its management address.

* Reviewed the VM inventory and identified the major workloads running on the node:

  * TrueNAS
  * Immich
  * Home Assistant
  * Omada Controller
  * Linux infrastructure services
  * Previous Kubernetes test nodes

* Reviewed storage mappings and raw-disk passthrough used by the TrueNAS VM.

* Confirmed that backups and recovery options should be validated before performing the final upgrade.

**Result:**
The primary Proxmox host passed the upgrade-readiness checks and was documented sufficiently to support a controlled upgrade and rollback process.

**Learning:**

* Major hypervisor upgrades should begin with compatibility validation rather than immediately changing repositories.
* Documenting networking, storage passthrough, and VM roles before an upgrade makes troubleshooting significantly easier.
* A successful compatibility check reduces risk but does not replace verified backups and restore testing.

---

## 28.07.26 - VLAN Architecture and Network Segmentation

**Goal:**
Replace the mostly flat homelab network with a structured VLAN design that separates infrastructure, trusted clients, game servers, IoT devices, and guests.

**Actions:**

* Expanded the physical Omada network infrastructure with:

  * **TP-Link EAP670** Wi-Fi 6 access point
  * **TP-Link ES216G** 16-port managed switch
  * Existing **TP-Link SG3210X-M2** 2.5 GbE core switch
  * Existing **TP-Link ER605 v2** gateway
* Replaced the previous simpler wireless and switching layout with centrally managed Omada equipment.
* Adopted the EAP670 and ES216G into Omada Controller.
* Configured the EAP670 to carry multiple SSIDs mapped to separate VLANs.
* Configured tagged uplinks between the router, core switch, secondary switch, and wireless access point.
* Used the SG3210X-M2 as the primary/core switch and the ES216G as an access-layer expansion switch for additional wired devices.

* Designed the following VLAN structure:

  | VLAN | Name     | Subnet            | Purpose                                 |
  | ---: | -------- | ----------------- | --------------------------------------- |
  |   10 | INFRA    | `192.168.10.0/24` | Hypervisors and infrastructure services |
  |   20 | TRUSTED  | `192.168.20.0/24` | Personal computers and trusted devices  |
  |   25 | GAME-DMZ | `192.168.25.0/24` | Publicly reachable game servers         |
  |   30 | IOT      | `192.168.30.0/24` | Smart-home and media devices            |
  |   40 | GUEST    | `192.168.40.0/24` | Guest devices and isolated access       |

* Created a structured IP-address plan for the main infrastructure:

  | System                | Address          |
  | --------------------- | ---------------- |
  | Primary Proxmox       | `192.168.10.10`  |
  | EliteDesk Proxmox     | `192.168.10.20`  |
  | Game-server Proxmox   | `192.168.10.30`  |
  | TrueNAS               | `192.168.10.40`  |
  | Proxmox Backup Server | `192.168.10.50`  |
  | Home Assistant        | `192.168.10.70`  |
  | Immich                | `192.168.10.71`  |
  | Pi-hole 2             | `192.168.10.90`  |
  | Pi-hole 1             | `192.168.10.91`  |
  | Omada Controller      | `192.168.10.100` |
  | Game Server VM        | `192.168.25.10`  |
  | Linux Game Server     | `192.168.25.20`  |

* Configured the VLAN networks and DHCP scopes through Omada Controller.

* Created switch port profiles for access ports, infrastructure ports, and tagged uplinks.

* Documented the physical switch layout across:

  * TP-Link SG3210X-M2
  * TP-Link ES216G
  * ER605 gateway
  * EAP670 access point

* Migrated the three Proxmox hosts and core infrastructure services to the new infrastructure VLAN.

* Migrated game-server workloads to the dedicated Game-DMZ VLAN.

* Used temporary permissive firewall rules during troubleshooting to distinguish routing problems from ACL problems.

* Verified connectivity between required infrastructure services after migration.

**Result:**
The network upgrade included both logical segmentation and a significant physical infrastructure expansion. The ER605 gateway, SG3210X-M2 core switch, ES216G access switch, and EAP670 wireless access point are now centrally managed through Omada Controller and provide the physical foundation for the VLAN architecture.


**Learning:**

* VLAN migrations should be performed in stages, beginning with management access and core services.
* DHCP reservations, switch port profiles, firewall rules, and VM bridge settings must all agree for a migration to work correctly.
* Temporary broad firewall rules are useful for diagnosis but should be replaced with explicit least-privilege rules after testing.
* A written IP plan is essential once infrastructure spans multiple hypervisors and VLANs.

---

## 29.07.26 - IoT Migration and Redundant Pi-hole DNS

**Goal:**
Move untrusted smart-home devices away from the main client network and establish redundant internal DNS filtering.

**Actions:**

* Created a dedicated wireless SSID mapped to the IoT VLAN.
* Migrated devices including:

  * Roborock vacuum
  * Google Nest devices
  * Samsung TV
  * Other smart-home and media devices
* Factory-reset and reconfigured devices that could not change wireless networks directly.
* Configured the television switch port with the IoT VLAN as its native network.
* Verified that migrated devices received addresses from the `192.168.30.0/24` DHCP scope.
* Configured two independent Pi-hole instances:

  * Primary DNS: `192.168.10.91`
  * Secondary DNS: `192.168.10.90`
* Updated DHCP DNS settings so clients could use both Pi-hole instances.
* Reviewed Pi-hole interface listening settings after the VLAN migration.
* Restarted Pi-hole FTL where required and verified DNS resolution from the migrated networks.
* Retained upstream DNS access for situations where the local filtering services were unavailable.

**Result:**
IoT devices are now separated from trusted computers and infrastructure, while clients have redundant filtered DNS through two Pi-hole instances.

**Learning:**

* IoT devices should not share unrestricted network access with administrative clients or hypervisors.
* DNS redundancy is only effective when both servers are independently reachable and correctly advertised through DHCP.
* Wireless migration often requires device resets because many consumer IoT devices provide no direct method for changing SSIDs.
* Network segmentation must account for discovery protocols and integrations that may cross VLAN boundaries.

---

## 30.07.26 - Power Recovery and DHCP Reservation Validation

**Goal:**
Validate how the network and virtual infrastructure recover after an uncontrolled power interruption.

**Actions:**

* Powered down the infrastructure during a local storm to protect the equipment.
* Restarted the router, switches, access point, Proxmox hosts, and virtual services from a shared power source.
* Discovered that one Pi-hole VM received an unexpected address instead of its reserved infrastructure address.
* Investigated:

  * DHCP reservation matching
  * VM network interface identity
  * VLAN tagging
  * bridge configuration
  * router and switch startup timing
* Renewed the VM network configuration and confirmed that it returned to its expected address, `192.168.10.90`.
* Compared the behavior with the second Pi-hole VM, which recovered correctly and retained its reservation.

**Result:**
The Pi-hole VM was restored to the correct infrastructure address, and the event exposed the importance of validating service startup after simultaneous power restoration.

**Learning:**

* DHCP reservations depend on the correct MAC address, VLAN, and DHCP server being available when a client requests its lease.
* Simultaneously powering the entire network can expose timing dependencies between the gateway, switches, hypervisors, and VMs.
* Recovery testing should verify actual service addresses and availability rather than assuming that automatic startup was successful.

---

## 31.07.26 - Centralized Homepage Dashboard and API Integration

**Goal:**
Create a central read-only dashboard showing the state of the entire homelab without requiring manual login to every service.

**Actions:**

* Deployed and configured a **Homepage** dashboard on the dedicated dashboard VM.

* Planned dashboard sections for:

  * Proxmox nodes
  * Storage and backups
  * Networking and DNS
  * Monitoring
  * Home automation
  * Media services
  * Game servers

* Added or prepared integrations for:

  * Three Proxmox nodes
  * Proxmox Backup Server
  * TrueNAS
  * Pi-hole
  * Omada Controller
  * Immich
  * Home Assistant
  * Game-server services

* Created read-only users, API tokens, and restricted credentials where supported.

* Tested Omada Controller authentication through its API.

* Diagnosed an initial Omada API response:

  ```text
  errorCode: -30109
  Invalid username or password
  ```

* Corrected the credentials and successfully received an authentication token:

  ```text
  errorCode: 0
  Log in successfully
  Token received: true
  ```

* Began refining the visual structure and widget configuration after confirming API access.

**Result:**
The dashboard foundation is operational, and the Omada API authentication required for network-status widgets has been validated. Further layout and integration work remains.

**Learning:**

* Dashboards should use dedicated read-only accounts instead of administrative credentials.
* Successful web login does not automatically guarantee that API authentication uses the same endpoint or credential format.
* Building integrations one service at a time makes it easier to isolate authentication, certificate, and formatting problems.

---

## 01.08.26 - Valheim Server Deployment and Automated Updates

**Goal:**
Deploy Valheim Dedicated Server on the existing Linux game-server VM and automate maintenance through systemd.

**Actions:**

* Installed Valheim Dedicated Server through SteamCMD.

* Configured it on the same Linux VM already used for Palworld rather than creating another VM.

* Created a dedicated Valheim server service for controlled startup and shutdown.

* Created an update script at:

  ```text
  /usr/local/bin/update-valheim.sh
  ```

* Created a systemd update service and timer.

* Verified successful execution of the update service:

  ```text
  status=0/SUCCESS
  ```

* Confirmed that `valheim-update.timer` triggered the update process correctly.

* Retained Playit.gg for external connectivity without router port forwarding.

* Structured the VM so multiple game servers could coexist while remaining independently manageable.

**Result:**
Valheim is installed on the Linux game-server VM with automated update handling through systemd. Palworld and Valheim can now be managed as separate services on the same virtual machine.

**Learning:**

* Multiple game servers can share a VM when their services, ports, users, and directories are clearly separated.
* Systemd timers provide a more observable and reliable update mechanism than manual updates or loosely managed cron jobs.
* Update scripts should stop the game service cleanly before replacing binaries and restart it only after a successful update.

---

## 01.08.26 - Restoring Proxmox Backup Jobs After VLAN Migration

**Goal:**
Restore backup connectivity after moving Proxmox hosts and Proxmox Backup Server to the new infrastructure VLAN.

**Actions:**

* Detected that existing Proxmox backup jobs could no longer reach Proxmox Backup Server after the address migration.
* Verified network connectivity between:

  * Primary Proxmox
  * EliteDesk Proxmox
  * Proxmox Backup Server
* Checked the configured PBS datastore endpoint and identified references to the previous network configuration.
* Updated the backup-server connection to use the new infrastructure address.
* Validated authentication and datastore availability.
* Re-ran backup jobs from the relevant Proxmox nodes.
* Restored backup operation for the primary infrastructure while deliberately postponing the game-server backup configuration until the remaining game-node setup was complete.

**Result:**
Backup connectivity was restored for the main Proxmox environment after the VLAN migration. The game-server node remained an explicitly documented follow-up task.

**Learning:**

* Changing management subnets affects more than web access; backup jobs, mounts, API integrations, and monitoring endpoints may all contain hard-coded addresses.
* Infrastructure migrations require a dependency checklist covering every system that communicates with the changed service.
* Backup validation should be performed immediately after network changes rather than waiting for the next scheduled job to fail.

---

## 02.08.26 - Network Documentation and GitHub Portfolio Consolidation

**Goal:**
Create a reliable single source of truth for the homelab and prepare the project for use as a technical portfolio.

**Actions:**

* Reviewed previously created topology and VLAN documents and identified that they did not contain every networked device.
* Replaced the incomplete documents with a consolidated network-state document.
* Documented all known devices that receive an IP address, including:

  * Routers
  * Switches
  * Wireless access points
  * Physical computers
  * Proxmox hosts
  * Virtual machines
  * LXC containers
  * Storage systems
  * Backup systems
  * Game servers
  * IoT devices
* Included both:

  * Current VLAN placement
  * Planned VLAN placement for devices not yet migrated
* Documented:

  * Static and reserved IP addresses
  * Service URLs and ports
  * Physical switch ports
  * Native and tagged VLANs
  * Host and VM roles
  * DNS configuration
  * Remaining migration work
* Began reorganizing the documentation for GitHub and Obsidian.
* Evaluated repository structure and decided to present the homelab as a coherent infrastructure project rather than a collection of unrelated machines.
* Began improving the project README and timeline for use in future job applications.

**Result:**
The homelab now has a more complete documentation baseline that reflects the real environment rather than only selected servers and services.

**Learning:**

* Infrastructure documentation should describe relationships between systems, not only list individual machines.
* Every device with an IP address can affect routing, security, DHCP, or troubleshooting and should therefore be included.
* Documentation becomes significantly more useful when it records both the current state and the intended target state.
* A well-structured GitHub repository can demonstrate architecture, troubleshooting, security, automation, and operational thinking to potential employers.

---

## 02.08.26 - EliteDesk NVMe Expansion and Migration Planning

**Goal:**
Expand local storage on the EliteDesk Proxmox node and prepare for a controlled migration of its virtual machines and infrastructure services.

**Actions:**

* Audited the existing NVMe and storage configuration on the EliteDesk node.
* Compared several affordable 1 TB NVMe options available from Norwegian retailers.
* Evaluated:

  * Interface compatibility
  * Performance
  * Brand reliability
  * Failure-rate considerations
  * Price and availability
* Selected and ordered a **WD Black SN7100 1 TB NVMe SSD**.
* Created an initial migration strategy covering:

  * Verified PBS backups
  * Storage creation
  * Controlled VM and LXC shutdown
  * Disk migration
  * Configuration validation
  * Service startup order
  * Rollback options
* Deliberately postponed the actual workload migration until the SSD is physically installed and verified.

**Result:**
The new NVMe drive has been selected and ordered, and the EliteDesk migration has a defined low-risk implementation plan. No production workloads have been moved yet.

The migration is part of a broader three-node architecture in which each Proxmox host will have a clearer responsibility:

* **Primary Proxmox:** Storage-heavy and core application workloads, including TrueNAS, Immich, and Home Assistant
* **EliteDesk Proxmox:** Lightweight infrastructure services, including DNS, backup, monitoring, and management tools
* **Game-server Proxmox:** Dedicated game-server workloads and services exposed through external tunnels

The final VM and service placement will be documented after the new NVMe drive is installed and the migration begins.

**Learning:**

* Storage migration should be treated as an infrastructure change rather than a simple file copy.
* Verified backups and a rollback path are required before moving DNS, backup, monitoring, or controller services.
* Separating planning from execution reduces the chance of introducing several simultaneous failures.
* Assigning each hypervisor a defined infrastructure role simplifies capacity planning, security segmentation, maintenance, and troubleshooting.

---

## 08.08.26 - VLAN Hardening, ACL Validation, and ES216G Recovery

**Goal:**  
Complete the VLAN migration by replacing the temporary permissive network state with explicit inter-VLAN access control, restricting the legacy default network, and validating the final segmented architecture.

**Actions:**

* Reviewed the completed VLAN migration and confirmed that normal clients and infrastructure services had been moved away from the legacy `DEFAULT` network.
* Disabled DHCP on `DEFAULT` to prevent devices from unintentionally falling back to the old flat network.
* Finalized the security model around the existing VLAN structure:

  | VLAN | Name | Role |
  | ---: | --- | --- |
  | 1 | DEFAULT | Restricted legacy / gateway-management network |
  | 10 | INFRA | Hypervisors, servers, and network infrastructure |
  | 20 | TRUSTED | Administrative and trusted personal devices |
  | 25 | GAME-DMZ | Externally reachable game-server workloads |
  | 30 | IOT | Smart-home and lower-trust consumer devices |
  | 40 | GUEST | Isolated guest access |

* Replaced temporary broad firewall permissions with ordered **Gateway ACLs** using a least-privilege model.
* Configured explicit DNS exceptions so isolated networks could reach the two Pi-hole servers without receiving unrestricted access to the infrastructure VLAN.
* Established `TRUSTED` as the primary administrative network with access to infrastructure, game-server, and IoT systems where required.
* Restricted reverse communication so infrastructure systems could not freely initiate connections toward trusted clients.
* Created a dedicated exception allowing **Home Assistant → IOT** communication while keeping general `INFRA → IOT` traffic blocked.
* Isolated `GAME-DMZ` from the private infrastructure while preserving:
  * Internet access
  * Pi-hole DNS
  * Playit.gg connectivity
  * administrative access originating from `TRUSTED`
* Isolated the `GUEST` network from internal systems while preserving Internet and DNS access.
* Function-tested the resulting policy in both directions, confirming that required traffic was allowed and prohibited traffic was actually blocked.

During the hardening work, a configuration change on the uplink between the **SG3210X-M2** core switch and **ES216G** access switch interrupted the ES216G management path to Omada Controller.

* Diagnosed the resulting `Adopt Failed` condition as a management-VLAN/uplink problem rather than a controller failure.
* Regained local access to the ES216G.
* Restored its management configuration on `INFRA`.
* Verified that the infrastructure VLAN was correctly transported across the inter-switch trunk.
* Restored connectivity between the ES216G and Omada Controller.
* Successfully re-adopted the switch and returned it to `CONNECTED` state.
* Revalidated downstream devices, VLAN assignments, DNS, Internet access, and inter-VLAN ACL behavior after recovery.
* Disabled all unused switch ports and documented the final port, trunk, VLAN, and management configuration.

**Result:**  
The VLAN migration progressed from basic logical separation to a fully enforced segmentation model. Infrastructure, trusted clients, game servers, IoT devices, and guests now operate in separate trust zones with explicit Gateway ACL rules controlling communication between them.

The legacy `DEFAULT` network is restricted rather than being available as an unrestricted fallback, both Pi-hole instances remain reachable through controlled DNS exceptions, and administrative access is centralized through `TRUSTED`.

The ES216G management incident was fully recovered without redesigning the network, and the final working topology, ACL policy, IP plan, port mapping, and recovery procedure were documented for future maintenance.

**Learning:**

* VLAN creation alone does not provide meaningful security; segmentation only becomes effective when routed traffic between zones is explicitly controlled.
* Least-privilege ACL design works best when narrow permit exceptions are placed before broader deny rules.
* Administrative access does not need to be symmetric: trusted clients can manage infrastructure without allowing infrastructure systems unrestricted access back to trusted endpoints.
* Management VLANs and inter-switch trunks are critical dependencies and should be changed one variable at a time with a documented rollback path.
* An Omada `Adopt Failed` state can be a symptom of broken network transport rather than a problem with the controller or adoption mechanism itself.
* Testing must verify both sides of the policy: required traffic must work, but traffic intended to be isolated must also be actively tested and confirmed blocked.
* Incident documentation is valuable when it captures not only the fix, but also the root cause, failed assumptions, recovery procedure, and preventive actions.

---

## 12.08.26 - ProDesk Infrastructure Migration and Core Service Consolidation

**Goal:**  
Replace the previous EliteDesk infrastructure role with the newer ProDesk node, preserve existing backup data, and consolidate lightweight infrastructure services onto the new platform without changing their established network identities.

**Actions:**

* Deployed a fresh **Proxmox VE 9** installation on the new **HP ProDesk 600 G6 Mini**.
* Assigned the ProDesk a dedicated infrastructure address at **`192.168.10.20`**.
* Reused the existing **1 TB WD Black SN7100 NVMe** as the Proxmox system disk.
* Installed the existing **2 TB Samsung 990 Pro** as the dedicated backup-storage disk.
* Detected duplicate Proxmox LVM volume-group names after attaching the Samsung disk and resolved the conflict by renaming the old volume group before accessing its contents.
* Recovered the existing Proxmox Backup Server datastore from the old storage layout:
  * identified the previous PBS virtual disk inside the old LVM thin pool
  * mounted the ext4 filesystem read-only
  * verified the existing PBS structure through `.chunks`, `vm`, and `ct`
  * confirmed approximately **291 GB** of existing backup data
* Created a temporary storage volume on the WD disk and copied the complete PBS datastore there.
* Verified the temporary copy using size comparison and checksum-based `rsync`.
* Wiped and rebuilt the full **2 TB Samsung 990 Pro** as a dedicated **ext4 PBS datastore**.
* Copied the PBS data back to the Samsung disk and performed a second checksum verification before removing the temporary copy.

* Migrated the **Homepage dashboard VM** from the primary Proxmox host to the ProDesk:
  * created a `vzdump` backup
  * transferred the backup to the ProDesk
  * restored it as VM **700**
  * retained its existing address at **`192.168.10.80`**
  * verified network connectivity after startup

* Migrated the secondary **Pi-hole** instance from the primary Proxmox host to the ProDesk:
  * backed up and restored the existing LXC container
  * retained its existing MAC address and DHCP reservation
  * verified that it returned to **`192.168.10.90`**
  * confirmed that Pi-hole FTL was listening on DNS port 53
  * verified real DNS resolution with `dig`
* Renumbered the two Pi-hole containers to make their Proxmox IDs reflect their infrastructure IP addresses:
  * **CT 1090 → `192.168.10.90`** on ProDesk
  * **CT 1091 → `192.168.10.91`** on the primary Proxmox host
* Removed the obsolete Pi-hole and dashboard instances from the primary Proxmox host after successful validation.

* Migrated **Home Assistant VM 150** from the primary Proxmox host to the ProDesk:
  * created and transferred a complete `vzdump` backup
  * restored the VM with its original configuration and network identity
  * preserved USB passthrough configuration using **`10c4:ea60`**
  * shut down the original VM before physically moving the **Silicon Labs CP210x / Zigbee USB coordinator**
  * verified that the ProDesk detected the coordinator before starting Home Assistant
  * confirmed that Home Assistant returned on **`192.168.10.70`**
  * tested Zigbee2MQTT devices including lighting and the IKEA STARKVIND air purifier
  * performed a full ProDesk reboot and confirmed that Home Assistant, USB passthrough, and Zigbee functionality recovered automatically
* Removed the obsolete Home Assistant VM from the primary Proxmox host after successful migration testing.

**Result:**  
The new ProDesk has taken over the lightweight infrastructure role previously planned for the EliteDesk. Dashboard, secondary Pi-hole, and Home Assistant now run successfully on the ProDesk while preserving their existing IP addresses and service configuration.

The Home Assistant Zigbee environment survived both VM migration and physical coordinator relocation without requiring devices to be re-paired. The two Pi-hole instances now use Proxmox container IDs that directly correspond to their infrastructure addresses, making the environment easier to identify and maintain.

The existing PBS backup data was preserved through a verified two-stage migration while the Samsung 990 Pro was rebuilt as a clean dedicated backup datastore.

**Learning:**

* Proxmox `vzdump` backup and restore provides a reliable method for moving both VMs and LXC containers between independent Proxmox hosts.
* Preserving guest MAC addresses and network configuration allows infrastructure services to retain their established DHCP reservations and IP identities after migration.
* USB passthrough based on **vendor/product ID** simplifies migration of hardware-dependent VMs because the same device can be attached on a different hypervisor without changing the guest configuration.
* Home Assistant and Zigbee migrations should be validated with actual device control and a complete host reboot rather than relying only on VM startup status.
* Duplicate LVM volume-group names must be resolved carefully before accessing disks originating from another Proxmox installation.
* Backup-storage migrations should be verified before and after destructive disk operations; checksum validation provides much stronger assurance than file-count or size comparison alone.
* Using a consistent VM/CT numbering convention tied to service IP addresses improves operational readability and reduces ambiguity during troubleshooting.

---

## 13.08.26 - PBS Recovery, Multi-Node Backup Validation, and Backup Job Cleanup

**Goal:**  
Bring Proxmox Backup Server fully back online on the new ProDesk, reconnect all Proxmox nodes to the recovered datastore, and rebuild the backup schedule around the current VM/LXC placement after the recent infrastructure migration.

**Actions:**

* Restored the existing **Proxmox Backup Server LXC (CT 1000)** from the latest available `vzdump` backup onto the ProDesk.
* Inspected the archived PBS configuration before starting the container and confirmed that the datastore expected:
  * datastore name: **`backup-local`**
  * internal path: **`/mnt/backup-local`**
* Connected the recovered 2 TB Samsung PBS datastore on the ProDesk host to CT 1000 using a bind mount:
  * host path: **`/mnt/pbs-datastore`**
  * container path: **`/mnt/backup-local`**
* Verified the datastore structure before first startup, including:
  * `.chunks`
  * `ct`
  * `vm`
  * `.gc-status`
* Started PBS and confirmed:
  * CT 1000 running normally
  * address **`192.168.10.50`** reachable
  * datastore `backup-local` registered correctly
  * no failed systemd units
* Opened the PBS web interface and confirmed that the historical VM and CT backup groups were visible.

* Ran a full **Verify All** against the recovered datastore.
* Completed verification across **13/13 backup groups** with the task ending in:
  ```text
  TASK OK
  ```
* Confirmed that actively checked snapshots returned **0 errors**, while recently verified snapshots were correctly skipped.
* Enabled automatic startup for PBS after successful validation.

* Reconnected the primary Proxmox host and ProDesk to the recovered PBS datastore.
* Tested new backups from both nodes:
  * **ProDesk CT 1090 (Pi-hole)** → successful
  * **Primary PVE CT 1091 (Pi-hole)** → successful
* Removed the temporary **400 GB `pbs-temp` thin volume** after the recovered datastore had passed verification and new writes had been tested.
* Confirmed that ProDesk thin-pool usage dropped from approximately **40% to 2.6%** after cleanup.

* Rebuilt the backup job scope to match the current workload placement:
  * **Primary PVE:** `100, 101, 106, 200, 1091`
  * **ProDesk:** `150, 700, 1090`
  * **Game-server PVE:** `100`
  * **EliteDesk-old:** no backup job yet because the node currently hosts no VMs or LXCs
* Standardized retention to:
  * **Keep Last: 3**
  * **Keep Weekly: 8**
  * **Keep Monthly: 12**
* Staggered backup execution to reduce simultaneous load:
  * **Primary PVE:** Sunday 01:00
  * **Game-server PVE:** Sunday 02:00
  * **ProDesk:** Sunday 03:00
* Deliberately scheduled the game-server backup before its Steam server update window.

* Added the recovered PBS datastore as **`PBS-local-new`** on:
  * Primary PVE
  * ProDesk
  * Game-server PVE
  * EliteDesk-old
* Verified that all configured nodes report the PBS storage as **active**.

* Diagnosed an initial game-server backup failure:
  ```text
  backup connect failed: command error: namespace not found
  ```
* Traced the issue to an incorrectly stored literal namespace value:
  ```text
  namespace Root
  ```
* Removed the explicit namespace entry so the storage correctly uses the PBS root namespace.
* Re-ran the game-server backup and completed a successful **300 GB VM backup** in approximately **2 minutes 12 seconds**, with PBS reusing approximately **94%** of existing data through incremental backup behavior.

* Reviewed the TrueNAS VM backup configuration and confirmed that only the **40 GB system disk** is included in PBS backups.
* Verified that all physical TrueNAS passthrough disks are explicitly excluded using:
  ```text
  backup=0
  ```
  This prevents the 4×8 TB ZFS pool, 3 TB workdrive, and 2×2 TB Immich disks from being copied into PBS.

* Investigated a stale `elitedesk` entry still visible in the primary PVE interface.
* Confirmed that the current **EliteDesk-old** host contained no VM/LXC 1000 and that the old VM disks no longer existed.
* Identified the entry as leftover Proxmox configuration metadata under the retired `elitedesk` node path.
* Removed the obsolete `1000.conf`, which cleared the phantom PBS container from the primary PVE interface.

**Result:**  
Proxmox Backup Server is fully operational on the ProDesk with its original datastore, historical backups, and verification state successfully recovered. The datastore passed a complete integrity verification and accepts new backups from multiple Proxmox nodes.

The primary PVE, ProDesk, game-server PVE, and EliteDesk-old are now all configured to access the centralized PBS instance at **`192.168.10.50`**. Active backup jobs have been rebuilt around the current workload placement, retention has been standardized, and execution times have been staggered to avoid unnecessary overlap.

The game-server node has completed a successful production-sized backup after correcting its namespace configuration, while the EliteDesk-old is already connected to PBS and ready for future workloads.

**Learning:**

* Recovering PBS requires preserving both the **PBS system configuration** and the **datastore independently**; restoring one without correctly reconnecting the other is not sufficient.
* A datastore migration should not be considered complete until historical backups pass **Verify**, new backups can be written successfully, and the system survives normal startup behavior.
* Proxmox PBS root namespace should be left empty; entering the literal value `Root` creates a namespace reference that does not exist.
* Backup jobs should be rebuilt after workload migrations rather than assuming old VMID lists and node assignments are still valid.
* Staggering backup schedules reduces simultaneous CPU, storage, and network load and avoids conflicts with other automated maintenance windows.
* Raw-disk passthrough devices should remain explicitly marked with **`backup=0`** when only the VM system disk and configuration need protection.
* Incremental PBS backups can make large virtual disks practical to protect because unchanged and sparse data does not need to be retransmitted in full.
* A virtualized PBS should not normally back up its own container to the PBS instance running inside that same container; the PBS system container should have a separate recovery strategy from the datastore it serves.
* Stale Proxmox node metadata can remain visible after hardware and hostname migrations even when no corresponding VM or storage exists, so configuration cleanup should be performed only after verifying that the referenced volumes are gone.

---

## 13.08.26 - Proxmox Workload Migration and VMID Standardization

**Goal:**  
Finalize workload placement across the homelab Proxmox nodes, distribute critical services across separate physical hosts, and introduce a consistent VM/CT numbering scheme that reflects which hypervisor owns each workload.

**Actions:**

* Reviewed the original VM and LXC placement across the primary Proxmox host, ProDesk, EliteDesk, and dedicated game-server node.
* Defined a node-based VMID/CTID convention:

  | Proxmox node | VMID / CTID range |
  | ------------ | ----------------- |
  | Primary PVE | `100–199` |
  | ProDesk | `200–299` |
  | EliteDesk | `300–399` |
  | Game-server PVE | `400–499` |

* Left the experimental Kubernetes environment on **`550–553`** unchanged because it remains a separate project and its final structure has not yet been decided.

* Consolidated lightweight infrastructure services on the **ProDesk**:
  * **VM 200 – Omada Controller**
  * **CT 201 – Pi-hole**
  * **VM 202 – Home Assistant**
  * **VM 203 – Homepage dashboard**
  * **CT 299 – Proxmox Backup Server**

* Distributed redundant DNS across two physical nodes instead of keeping both Pi-hole instances on the same host:
  * **ProDesk:** CT `201`
  * **EliteDesk:** CT `301`

* Kept the storage-heavy workloads on the primary Proxmox host:
  * **VM 100 – Linux server**
  * **VM 101 – TrueNAS**
  * **VM 106 – Immich**

* Standardized the dedicated game-server node by assigning its game-hosting VM to:
  * **VM 400 – GamesVM**

* Renumbered the Proxmox Backup Server container from **CT 1000 → CT 299** on the ProDesk:
  * created a fresh `vzdump` backup of CT 1000
  * confirmed that the external PBS datastore bind mount was correctly excluded from the container backup
  * restored the container as CT 299
  * preserved the existing bind mount:
    ```text
    /mnt/pbs-datastore → /mnt/backup-local
    ```
  * preserved the existing network identity and verified that PBS returned on **`192.168.10.50`**
  * confirmed that `proxmox-backup.service` was running
  * confirmed that datastore **`backup-local`** was registered and mounted correctly
  * verified historical backup groups through the PBS web interface
  * confirmed existing backup verification state remained **All OK**
  * removed the obsolete CT 1000 only after CT 299 had passed all validation checks

* Verified the final active workload layout:

  | Node | VMID / CTID | Workload |
  | ---- | -----------: | -------- |
  | Primary PVE | 100 | Linux server |
  | Primary PVE | 101 | TrueNAS |
  | Primary PVE | 106 | Immich |
  | ProDesk | 200 | Omada Controller |
  | ProDesk | 201 | Pi-hole |
  | ProDesk | 202 | Home Assistant |
  | ProDesk | 203 | Homepage dashboard |
  | ProDesk | 299 | Proxmox Backup Server |
  | EliteDesk | 301 | Pi-hole |
  | Game-server PVE | 400 | GamesVM |

**Result:**  
The homelab now has a clear node-based VMID/CTID structure and a more deliberate workload distribution. Core storage remains on the primary Proxmox host, lightweight infrastructure is consolidated on the ProDesk, redundant DNS is split between ProDesk and EliteDesk, and game workloads use the dedicated game-server node.

The PBS renumbering from CT 1000 to CT 299 was completed without moving or rewriting the backup datastore itself. Historical backups remained accessible, the datastore stayed intact, and the PBS service retained its existing network identity.

The resulting numbering scheme makes it possible to identify the physical host of most production workloads directly from the VMID or CTID while keeping the Kubernetes lab separate until its architecture is finalized.

**Learning:**

* VMID and CTID conventions become increasingly valuable as a Proxmox environment grows beyond a single hypervisor.
* Host-based numbering makes inventory, troubleshooting, documentation, and backup-job maintenance easier because workload ownership is immediately visible.
* Redundant services should be distributed across separate physical hosts when possible; two DNS instances on the same hypervisor do not provide meaningful host-level redundancy.
* A Proxmox LXC can be safely renumbered through backup and restore when its configuration and external mount dependencies are validated before the old container is removed.
* Bind-mounted application data should be treated separately from the container root filesystem during migration.
* Infrastructure migrations should be considered complete only after validating the actual application, network identity, mounted storage, historical data, and service startup behavior.
* Experimental environments do not need to follow production conventions before their final architecture is known.

---
