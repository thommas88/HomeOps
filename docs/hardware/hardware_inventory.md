# Hardware Inventory

Physical hardware overview of six systems. Virtual machines, containers, and hosted services are intentionally excluded.

**Last updated:** 20 October 2026  
**Reference:** [HomeOps](https://github.com/thommas88/HomeOps)

---

## PC 1 — Gaming Workstation

**Host:** Fedora workstation

| Component | Model / Specification |
| --- | --- |
| CPU | AMD Ryzen 7 9800X3D |
| Motherboard | ASUS TUF Gaming X670E-PLUS WIFI |
| Memory | 32 GB G.Skill Flare X5 DDR5-6000 |
| GPU | ASUS TUF Gaming GeForce RTX 4070 Ti OC |
| CPU Cooler | Noctua NH-D15 |
| Case | Fractal Design North Charcoal Black Mesh |
| Power Supply | Corsair RM850x — 850 W |
| Storage | Samsung 990 PRO 2 TB NVMe SSD |
| Storage | Samsung 870 SSD (capacity not confirmed) |
| Storage | Seagate 4 TB HDD (model not confirmed) |

## PC 2 — Proxmox Main

**Host:** `pve` (`pve-main` in HomeOps)

| Component | Model / Specification |
| --- | --- |
| CPU | Intel Core i9-9900K |
| Motherboard | ASUS ROG STRIX Z390-F GAMING |
| Memory | 64 GB Kingston FURY Beast DDR4-3200 (2 × 32 GB) |
| CPU Cooler | Noctua NH-D15 (not physically confirmed) |
| Case | Fractal Design Define 7 |
| Power Supply | Corsair 850 W (exact model not confirmed; possibly RM850i) |
| Storage (OS) | Samsung 970 EVO Plus 1 TB NVMe SSD |
| Storage | 4 × Seagate IronWolf ST8000VN004 8 TB HDD |
| Storage | WD Black WD2002FAEX 2 TB HDD |
| Storage | WD Green WD20EARS 2 TB HDD |
| Storage | WD WD30EZRX 3 TB HDD |
| Storage | Samsung 850 EVO 250 GB SSD |
| Network | Intel I226-LM 2.5 GbE (active); Intel I219-V 1 GbE |
| HBA | LSI 9207-8i (IT mode; documented in HomeOps, not independently verified) |

## PC 3 — Proxmox Game Server

**Host:** `pve-game-server`

| Component | Model / Specification |
| --- | --- |
| CPU | Intel Core i5-14500 |
| Motherboard | MSI MAG B760M MORTAR WIFI II |
| Memory | 32 GB G.Skill Trident Z5 DDR5-6400 (moved from PC 1) |
| CPU Cooler | Noctua NH-U12A |
| Case | Lian Li A3 mATX |
| Power Supply | Corsair RM650e — 650 W |
| Storage (OS) | Samsung 860 EVO 1 TB SATA SSD (moved from PC 2) |
| Network | Realtek RTL8125 2.5 GbE |
| Remote Management | JetKVM v1 |

## PC 4 — HP ProDesk 600 G6 Mini

**Host:** `prodesk`

| Component | Model / Specification |
| --- | --- |
| CPU | Intel Core i5-10500T |
| Motherboard | HP 8715 |
| Memory | 16 GB (module specifications not confirmed) |
| Storage (OS) | WD_BLACK SN7100 1 TB NVMe SSD |
| Storage | Samsung 990 PRO 2 TB NVMe SSD |
| Network | Intel I219-LM 1 GbE |
| Power Adapter | Model / wattage not confirmed |

## PC 5 — HP EliteDesk 800 G5 Mini

**Host:** `elitedesk-old`

| Component | Model / Specification |
| --- | --- |
| CPU | Intel Core i5-9600T |
| Motherboard | HP 8595 |
| Memory | 16 GB (module specifications not confirmed) |
| Storage (OS) | Toshiba / Kioxia KBG40ZNV256G 256 GB NVMe SSD |
| Network | Intel I219-LM 1 GbE |
| Power Adapter | Model / wattage not confirmed |

## PC 6 — Spare PC

**Status:** Not currently in use

| Component | Model / Specification |
| --- | --- |
| CPU | Intel LGA1151 (exact model unknown) |
| Motherboard | ASUS ROG STRIX Z270F GAMING |
| Memory | 16 GB DDR4 (module specifications unknown) |
| CPU Cooler | Noctua (model unknown) |
| Case | Corsair 4000D Airflow |
| Power Supply | Corsair 650 W (model unknown; separate unit from PC 3) |
| Storage | Not confirmed |

---

## Documentation Notes

- **PC 3 motherboard:** `inxi` confirms the MSI MAG B760M MORTAR WIFI II. Any references to ASUS Prime B760M-A WIFI in HomeOps are outdated.
- **PC 2 network adapter:** `inxi` identifies the controller as Intel I226-LM. The add-in card's retail model has not been independently confirmed.
- **Unconfirmed entries** are labeled in the tables rather than filled in with assumptions.
- Hardware details are based on owner confirmations, the supplied purchase overview, and `inxi` reports. Serial numbers, MAC addresses, and UUIDs are excluded.
