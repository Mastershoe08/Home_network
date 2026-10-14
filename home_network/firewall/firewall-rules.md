# Firewall Rules

Zone-based firewall rules on the UCG Max.

All three VLANs are in the **Internal** zone. IoT and Lab have **Isolate Network** enabled, so by default they cannot talk to other VLANs. The rules below are the only exceptions.

## Summary

| ID | Name | Action | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|---|
| 10000 | Tv to Jelly | Allow | Roku TV | Jellyfin (192.168.50.162) | 8096 | Roku streams from Jellyfin |
| 10001 | IoT to Pihole | Allow | IoT network | Pi-hole (192.168.50.5) | 53 | IoT devices use Pi-hole for DNS |
| 10002 | Main to lab | Allow | Desktop PC | Lab network | Any | Manage Proxmox and lab from main PC |

## Traffic Matrix

What can initiate connections to what ( allowed,  blocked,  only by rule):

| From ↓ / To → | Default | IoT | Lab | Internet |
|---|---|---|---|---|
| **Default** |  |  (isolated) |  Desktop PC only (10002) |  |
| **IoT** |  DNS to Pi-hole, Roku to Jellyfin (10000, 10001) |  |  |  |
| **Lab** |  (isolated) |  |  | TODO: confirm |

---

## Rule Details

### 10000 — Tv to Jelly

| Field | Value |
|---|---|
| Action | Allow |
| IP Version | IPv4 |
| Protocol | All |
| Source Zone | Internal |
| Source | Roku TV (client device) |
| Source Port | Any |
| Destination Zone | Internal |
| Destination | 192.168.50.162 |
| Destination Port | 8096 |

**Why:** The Roku TV is on the isolated IoT network but needs to reach the Jellyfin server on Default.

**Depends on:**
- Jellyfin staying at **192.168.50.162**. Give the host a DHCP reservation or static IP, especially after the Ubuntu rebuild, when its MAC address will change.
- The Roku being identified as the same client device. If it is replaced or reset with a new MAC, update the source.

### 10001 — IoT to Pihole

| Field | Value |
|---|---|
| Action | Allow |
| IP Version | Both |
| Protocol | All |
| Source Zone | Internal |
| Source | IoT network |
| Source Port | Any |
| Destination Zone | Internal |
| Destination | 192.168.50.5 |
| Destination Port | 53 |

**Why:** IoT clients use Pi-hole for DNS and ad blocking. Without this rule, isolation would block DNS and IoT devices would lose name resolution.

**Depends on:**
- Pi-hole staying at **192.168.50.5**.
- Pi-hole being set to accept queries from non-local subnets (IoT is 192.168.20.x).

### 10002 — Main to lab

| Field | Value |
|---|---|
| Action | Allow |
| IP Version | Both |
| Protocol | All |
| Source Zone | Internal |
| Source | Desktop PC (client device) |
| Source Port | Any |
| Destination Zone | Internal |
| Destination | Lab network |
| Destination Port | Any |

**Why:** Lab is isolated. Only the main desktop should be able to reach Proxmox, Kali, and other lab machines, for management and testing.

**Depends on:**
- The desktop being identified as the same client device. Replacing its network card or switching it to Wi-Fi would change its MAC and break the rule.

---

## Known Limitations

- **Casting to the Roku** from phones or laptops on Default likely won't work. Default → IoT is blocked, and discovery also needs mDNS/SSDP.
- **Home Assistant** (Default) cannot reach devices on IoT. Smart devices moved to IoT would need new rules.
- **Hit counters** showed zero on rules 10000 and 10002 after a power outage while Jellyfin was off. Zero hits does not necessarily mean a rule is broken. Test the actual traffic.

## Possible Improvements

- [ ] Narrow `IoT to Pihole` from Protocol: All to TCP/UDP.
- [ ] Narrow `Tv to Jelly` from Protocol: All to TCP.
- [ ] Add DHCP reservations for Pi-hole and Jellyfin so destination IPs can't change.
- [ ] Decide whether `Main to lab` should be limited to specific ports (e.g. Proxmox web UI 8006, SSH 22) instead of Any.

## Adding a Rule

When creating a new rule, document it here with:
1. The ID, name, and full field values.
2. **Why** it exists.
3. What it **depends on** (IPs, devices, MACs) so a future change doesn't break it silently.
