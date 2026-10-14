# Network

VLANs, switch and gateway port maps, and Wi-Fi configuration.

Firewall rules are documented separately in [firewall-rules.md](firewall-rules.md).

> **Sanitized:** WAN IP, MAC addresses, SSIDs and passwords are intentionally omitted.

---

## VLANs

All networks are routed by the **UCG Max**, which is also the DHCP server for each one. IPv6 is not configured on any network.

| Name | VLAN ID | Subnet | Gateway | DHCP Range | Zone | Isolated | Internet |
|---|---|---|---|---|---|---|---|
| Default | 1 | 192.168.50.0/24 | 192.168.50.1 | UCG Max (249 leases) | Internal | No | Yes |
| IoT | 2 | 192.168.20.0/29 | 192.168.20.1 | 192.168.20.2 – .6 | Internal | **Yes** | Yes |
| Lab | 100 | 192.168.100.0/24 | 192.168.100.1 | UCG Max (249 leases) | Internal | **Yes** | TODO: confirm |

### Default (VLAN 1)

Trusted network for personal devices and core services.

- Pi-hole (192.168.50.5) provides DNS.
- Home Assistant (192.168.50.2) and Jellyfin (192.168.50.162) live here.
- The switch (192.168.50.216) and access point (192.168.50.118) are managed on this network.

### IoT (VLAN 2)

Isolated network for devices that don't need to reach anything else on the LAN.

| Setting | Value | Why |
|---|---|---|
| Netmask | /29 (5 usable leases) | Small by design; only a few devices live here |
| Auto-Scale Network | On | UniFi grows the subnet if leases run out |
| Isolate Network | On | Blocks IoT from initiating connections to other VLANs |
| Allow Internet Access | On | Devices still need streaming, updates, online play |
| DHCP Guarding | On (trusted: 192.168.20.1) | Blocks rogue DHCP servers on this VLAN |
| Auto DNS Server | Off | DNS is set manually. TODO: confirm it points to Pi-hole (192.168.50.5) |
| Auto Default Gateway | On | Gateway is 192.168.20.1 |

**How devices get here:** there is no separate IoT SSID. Devices connect to the normal Wi-Fi and are moved to IoT with UniFi's per-client **Virtual Network Override** (see [Wi-Fi](#wi-fi)).

**Cross-VLAN access** is only what the firewall rules allow: DNS to Pi-hole, and the Roku TV to Jellyfin.

### Lab (VLAN 100)

Isolated network for virtualization and security tooling.

- MSI Proxmox (192.168.100.186)
- Kali (192.168.100.55)
- Devices land here by being plugged into switch ports with native VLAN set to Lab (ports 8 and 9).
- Only the desktop PC on Default is allowed in (firewall rule `Main to lab`).

---

## Port Maps

### UCG Max

| Port | Connected To | Speed | Notes |
|---|---|---|---|
| 1 | USW-Lite-16-PoE (port 15) | GbE | Uplink to core switch |
| 2 | *Unused* | – | |
| 3 | *Unused* | – | |
| 4 | *Unused* | – | Desktop PC was previously connected here |
| 5 (WAN) | Comcast cable modem | 2.5 GbE | |

### USW-Lite-16-PoE

All ports are STP participants. Ports 1–7 support PoE+.

| Port | Device | IP | Native VLAN | PoE | Speed | Status |
|---|---|---|---|---|---|---|
| 1 | U7 Lite (AP) | 192.168.50.118 | Default | PoE+ (powered) | GbE | Up |
| 2 | Home Assistant | 192.168.50.2 | Default | Off | GbE | Up |
| 3 | *Unused* | – | Default | PoE+ | – | – |
| 4 | *Unused* | – | Default | PoE+ | – | – |
| 5 | *Unused* | – | Default | PoE+ | – | – |
| 6 | *Unused* | – | Default | PoE+ | – | – |
| 7 | *Unused* | – | Default | PoE+ | – | – |
| 8 | MSI Proxmox | 192.168.100.186 | **Lab** | Off | GbE | Up |
| 9 | Kali | 192.168.100.55 | **Lab** | – | GbE | Up |
| 10 | *Unused* | – | Default | – | – | – |
| 11 | *Unused* | – | Default | – | – | – |
| 12 | Desktop PC | 192.168.50.167 | Default | – | GbE | Up |
| 13 | Jellyfin host | 192.168.50.162 | Default | – | GbE | Down (PSU failure, rebuild planned) |
| 14 | homeserver | – | Default | – | – | Down |
| 15 | UCG Max (uplink) | – | Default | – | GbE | Up |
| 16 | Pi-hole (Raspberry Pi) | 192.168.50.5 | Default | – | GbE | Up |

**Free PoE+ ports:** 3–7, available for future APs, cameras, or PoE devices.

---

## Wi-Fi

| Item | Value |
|---|---|
| Access point | UniFi U7 Lite |
| Uplink | Switch port 1, PoE+ powered |
| Bands in use | 2.4 GHz and 5 GHz |
| SSIDs | TODO: document names (kept private) and which network each maps to |

### Client placement

| Device | Band | Network | How assigned |
|---|---|---|---|
| Roku TV | 5 GHz | IoT | Virtual Network Override + `IoT` client group |
| Nintendo Switch 2 | 5 GHz | IoT | Virtual Network Override + `IoT` client group |
| Laptops (x2) | 5 GHz | Default | – |
| Phones (x3) | 5 GHz / 2.4 GHz | Default | – |
| Tablet | 5 GHz | Default | – |
| DD-WRT router | 5 GHz | Default | Wireless client. TODO: document its purpose |

The `IoT` client group on the Roku TV and Switch is used as a source in firewall rules, so new IoT devices should be added to that group as well as overridden to the IoT network.

### Signal baseline

Snapshot of client signal strength, useful for troubleshooting later.

| Device | Signal | Band | Note |
|---|---|---|---|
| Roku TV | -19 dBm | 5 GHz | |
| Nintendo Switch 2 | -22 dBm | 5 GHz | |
| Tablet | -35 dBm | 5 GHz | |
| Phone | -40 dBm | 5 GHz | |
| Phone | -47 dBm | 5 GHz | |
| Laptop | -53 dBm | 5 GHz | |
| Laptop | -62 dBm | 5 GHz | |
| Phone | -62 dBm | 2.4 GHz | Only 2.4 GHz client |
| DD-WRT router | -70 dBm | 5 GHz | Weakest link; check placement if it causes issues |

### Adding a new IoT device

1. Connect the device to Wi-Fi as normal.
2. In UniFi, open the client → **Settings**.
3. Enable **Virtual Network Override** and select **IoT**.
4. Add the client to the **IoT** group.
5. Reconnect the device so it picks up a 192.168.20.x address.
6. If IoT leases are full, confirm Auto-Scale expanded the subnet.
