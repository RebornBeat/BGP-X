# BGP-X Satellite Modem Extension

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2

---

## 1. Overview

BGP-X Router v1 and BGP-X Node v1 (with WAN) support **satellite internet modems via USB**, enabling deployment in locations without fiber, cellular, or other terrestrial WAN options.

This document covers hardware integration specifics: supported satellite modems, USB auto-detection, driver configuration, power requirements for combined deployments, latency class configuration, and physical deployment considerations.

---

## 2. Hardware Compatibility

Satellite WAN support is available on:

| Hardware | USB Ports | Recommended Satellite Use Case |
|---|---|---|
| **BGP-X Router v1** | 2× USB 3.0 Type-A | Primary WAN; high-throughput (Starlink) |
| **BGP-X Node v1 (with WAN)** | 1× USB 2.0 | Backup WAN; low-throughput (Iridium, Inmarsat) |
| **BGP-X Gateway v1** | 2× USB 3.0 + SFP+ | Multi-WAN failover; provider satellite backup |

**Note on USB Bandwidth**: Router v1 and Gateway v1 have USB 3.0 ports capable of supporting high-throughput satellite connections (Starlink Gen 3, high-speed Ku-band). Node v1 has USB 2.0, which is sufficient for lower-bandwidth satellite services (Iridium Certus, Inmarsat BGAN) but may limit peak throughput for Starlink if multiple BGP-X sessions saturate the link.

**Reference Documentation**:
- `router_spec.md` — BGP-X Router v1 full specification.
- `node_spec.md` — BGP-X Node v1 full specification.
- `gateway_spec.md` — BGP-X Gateway v1 full specification.

---

## 3. Supported Satellite Modems

### 3.1 Starlink Gen 3 (Recommended for High Throughput)

| Parameter | Value |
|---|---|
| USB Vendor ID | `0x48bf` |
| USB Product ID | `0x0800` |
| Interface | USB Ethernet (RNDIS / CDC-NCM) |
| IP assignment | DHCP from Starlink router (or dish in bypass mode) |
| Bandwidth | 20-220 Mbps down, 5-25 Mbps up (varies by location/plan) |
| Latency | 25-60 ms |
| BGP-X latency class | `satellite-leo` |
| Power draw | 50-75 W (dish + Starlink router combined) |
| Setup | USB-C from dish to BGP-X hardware USB-A port via USB-C to A adapter |

**Driver**: Standard Linux USB Ethernet CDC-NCM driver (included in OpenWrt 23.05 and all Linux 5.15+ kernels). No custom driver needed.

**Auto-detection**: BGP-X daemon detects USB vendor/product ID at startup and records `wan_type = satellite` in transport metadata.

```toml
# Auto-generated when Starlink USB detected:
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
satellite_orbit = "leo"
latency_class = "satellite-leo"
detected_interface = "usb0"
```

**Starlink Gen 3 Dish Mounting**:
- Requires clear sky view (>100° azimuth cone for optimal coverage).
- Self-orienting electronically (no mechanical motor). Roof mount, pole mount, or flat surface.
- BGP-X hardware can be co-located with or remote from the dish.
- Maximum Ethernet cable length: up to 100m (Cat6 recommended).
- **Outdoor IP67 Carrier Compatibility**: If using BGP-X Router v1 Outdoor IP67 carrier, the Starlink dish should be mounted externally while the BGP-X router can be protected inside a waterproof box or indoor space, connected via waterproof Ethernet cable gland. Power for Starlink requires separate outdoor-rated cabling (50-75W draw).

**Bypass Mode**: Starlink Gen 3 dishes can operate in "bypass" mode, disabling the built-in router and presenting a direct Ethernet interface. This is preferred for BGP-X integration to avoid double-NAT.

### 3.2 Iridium Certus 100 / 350 / 700

| Parameter | Value |
|---|---|
| Interface | Serial (RS-232 or USB-serial via adapter) |
| Linux device | `/dev/ttyUSB0` (USB-serial) or `/dev/ttyS0` (RS-232) |
| Bandwidth | 88 Kbps up / 704 Kbps down (Certus 700 max) |
| Latency | 30-80 ms (LEO, ground station routing adds latency) |
| BGP-X latency class | `satellite-leo` |
| Power draw | 5-15 W (modem only; antenna separate) |
| Coverage | Global including polar regions |

**Driver**: Standard Linux USB-serial driver (`cp210x`, `ch341`, or `ftdi_sio` depending on modem's USB-serial chip). Included in OpenWrt 23.05.

**AT Command Interface**: BGP-X satellite transport driver communicates with Iridium modem via AT commands:
```
AT+IPR=115200        Set baud rate
AT+SBDWT=...         Write data to modem buffer
AT+SBDIX             Initiate SBD session (send/receive)
AT+SBDRT             Read received data
AT+CSQ               Check signal quality
```

**Hardware Interface**: On BGP-X Router v1 and Node v1, Iridium modems connect via USB-serial adapter to a USB port. Ensure the adapter uses a kernel-supported chipset.

**Bandwidth Limitation**: At 88-704 Kbps, Iridium Certus is suitable for BGP-X DHT participation and light relay duty. Configure bandwidth cap:

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "iridium-certus"
satellite_orbit = "leo"
latency_class = "satellite-leo"
max_bandwidth_kbps = 50         # Conservative; leaves headroom
min_session_count = 1           # Limit concurrent BGP-X sessions
```

**Use Case**: Remote monitoring, sensor networks, backup WAN in extreme latitudes, polar research, maritime backup.

### 3.3 Inmarsat BGAN / Fleet One

| Parameter | Value |
|---|---|
| Interface | Ethernet (from terminal) or USB |
| Bandwidth | 32-432 Kbps (BGAN Standard) |
| Latency | 550-700 ms (GEO) |
| BGP-X latency class | `satellite-geo` |
| Power draw | 10-30 W |
| Coverage | Near-global (latitude-limited GEO) |

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "inmarsat"
satellite_orbit = "geo"
latency_class = "satellite-geo"
max_bandwidth_kbps = 30
```

GEO latency (600ms+) adds significantly to BGP-X path RTT. All `PATH_QUALITY_REPORT` entries for this node's clearnet domain will show GEO-class latency — expected and not flagged as congestion. Geo plausibility scoring: `satellite-geo` exempt (returns 0.5 neutral).

### 3.4 Generic LTE/5G USB Modems (Not Satellite)

Any USB LTE or 5G modem that presents as a USB Ethernet interface or USB-serial modem is compatible. These are NOT satellite — they are cellular (clearnet, latency class `cellular`).

```toml
[clearnet_transport]
wan_type = "cellular"
latency_class = "cellular"
```

Auto-detected via USB Ethernet vendor IDs for common modems (Huawei HiLink, Quectel EC25, Sierra Wireless EM7411, etc.).

---

## 4. USB Auto-Detection

BGP-X daemon performs USB device detection at startup and on USB hotplug events:

```rust
// In bgpx-transport-satellite/src/detector.rs:

const KNOWN_SATELLITE_DEVICES: &[UsbDevice] = &[
    UsbDevice {
        vendor_id:    0x48bf,
        product_id:   0x0800,
        provider:     "starlink",
        orbit:        "leo",
        latency_class: LatencyClass::SatelliteLeo,
        interface:    InterfaceType::UsbEthernet,
    },
    // Iridium Certus adapters (various USB-serial chips)
    UsbDevice {
        vendor_id:    0x10c4,    // CP210x
        product_id:   0xea60,
        provider:     "iridium-certus",
        orbit:        "leo",
        latency_class: LatencyClass::SatelliteLeo,
        interface:    InterfaceType::UsbSerial,
    },
    // Additional modems added as supported
];

fn detect_satellite_modem() -> Option<SatelliteModemConfig> {
    for device in enumerate_usb_devices() {
        for known in KNOWN_SATELLITE_DEVICES {
            if device.vendor_id == known.vendor_id
               && device.product_id == known.product_id {
                return Some(SatelliteModemConfig::from_known(known, device));
            }
        }
    }
    None
}
```

When a satellite modem is detected:
1. Driver loads automatically (USB Ethernet: CDC-NCM; USB-serial: appropriate serial driver).
2. Network interface created (e.g., `usb0` for Ethernet, BGP-X serial channel for AT command modems).
3. DHCP client started on USB interface (for Ethernet-type modems).
4. BGP-X clearnet transport configured with satellite provider metadata.
5. Geo plausibility scoring: satellite latency class automatically exempts this node.
6. Node advertisement updated: clearnet endpoint with `latency_class: satellite-leo` or `satellite-geo` metadata.

---

## 5. Power Requirements

Satellite terminals have significant power requirements. BGP-X hardware power is additive. Combined deployments require careful power system design.

### 5.1 Power Budget Table

| Configuration | BGP-X Hardware | Satellite Terminal | Total (Peak) |
|---|---|---|---|
| **Router v1 (Desktop)** + Starlink Gen 3 | 5-15 W | 50-75 W | **55-90 W** |
| **Router v1 (Outdoor IP67)** + Starlink Gen 3 | 5-15 W | 50-75 W | **55-90 W** |
| **Node v1** + Starlink Gen 3 | 3-12 W | 50-75 W | **53-87 W** |
| **Node v1** + Iridium Certus 100 | 3-8 W | 5-10 W | **8-18 W** |
| **Node v1** + Inmarsat BGAN | 3-8 W | 15-30 W | **18-38 W** |

**Note on Node v1 Power**: The BGP-X Node v1 specification indicates 3-12W consumption depending on active radios (LoRa, WiFi, BLE). For satellite deployments, WiFi is typically unnecessary if LoRa is used for the mesh transport, reducing Node v1 consumption to the lower end (3-5W). This is critical for off-grid solar design.

### 5.2 Solar Sizing — Starlink + BGP-X Router/Node

Starlink Gen 3 requires **substantial power**.

**Example Calculation (Starlink + BGP-X Router v1 Outdoor IP67)**:
- Average Load: 65W (Starlink) + 8W (Router, conservative) = **73W**.
- Daily Energy (24hr operation): 73W × 24h = **1752 Wh/day**.
- Solar Generation (5 peak sun hours/day): 1752 Wh ÷ 5h = **350W panel minimum**.
- **Recommended Panel Array**: **500W-600W** to account for efficiency losses, cloudy days, and latitude variations.
- Battery Storage (2-day reserve, no sun): 1752 Wh/day × 2 = 3504 Wh.
  - At 24V LiFePO4: 3504 Wh ÷ 24V = **146 Ah**.
  - At 12V LiFePO4: 3504 Wh ÷ 12V = **292 Ah**.

**Recommendation**: A permanent off-grid Starlink + BGP-X Router deployment is a significant solar project. It is suitable for community mesh gateways with dedicated solar infrastructure, but not trivial for portable/temporary setups.

### 5.3 Solar Sizing — Iridium + BGP-X Node v1

Iridium Certus is **far more achievable** for portable/off-grid operation.

**Example Calculation (Iridium Certus 100 + BGP-X Node v1 LoRa-only)**:
- Average Load: 10W (Iridium) + 4W (Node v1 LoRa-relay) = **14W**.
- Daily Energy: 14W × 24h = **336 Wh/day**.
- Solar Generation: 336 Wh ÷ 5h = **67W panel minimum**.
- **Recommended Panel**: **100W**.
- Battery (1-day reserve): 336 Wh ÷ 12V = **28 Ah**.

**Result**: A 100W portable solar panel + 30Ah LiFePO4 battery provides reliable 24/7 BGP-X + Iridium operation. This is viable for field researchers, journalists, and portable deployments.

---

## 6. Physical Deployment

### 6.1 Starlink Gen 3 + BGP-X Router (Outdoor IP67)

```
[Solar Array 500W]
        │
        ▼
[Charge Controller]
        │
        ▼
[LiFePO4 Battery Bank 150Ah 24V]
        │
        ├───► [Starlink Gen 3 Dish] (pole/mast mount)
        │              │ (USB-C cable, weather-sealed)
        │              ▼
        └──► [BGP-X Router v1 Outdoor IP67] (PoE or DC input)
                        │ (optional: LoRa antenna, WiFi antennas)
                        ▼
                    [BGP-X Mesh] (LoRa/WiFi radio)
```

**Key Considerations**:
- **Cable Length**: USB-C cable from dish to router should be high-quality, shielded, and within 3m if possible. USB 3.0 signal degrades over distance.
- **Weatherproofing**: USB connections must be weather-sealed with silicone boots or IP-rated USB connectors.
- **Antenna Separation**: Starlink dish and LoRa antenna should be separated by >1m to minimize interference.
- **Thermal**: Outdoor IP67 enclosure provides passive cooling. Ensure enclosure is not in direct sunlight (or add sun shade) to avoid overheating in hot climates.

### 6.2 Iridium Certus + BGP-X Node v1

```
[Solar Panel 100W]
        │
        ▼
[BGP-X Node v1 (with WAN, solar-powered)]
        │
        ├── USB-Serial Adapter
        │       │
        │       ▼
        └── [Iridium Certus Modem + Antenna]
                        │
                        └──► [Iridium LEO Constellation]
```

**Key Considerations**:
- **Antenna Placement**: Iridium antenna needs clear sky view. Mount at highest point.
- **USB-Serial Adapter**: Ensure it is weather-sealed or placed in the Node v1's IP67 enclosure with a cable gland for the Iridium antenna cable.

---

## 7. Connectivity Verification

After satellite modem detection and configuration:

```bash
# Verify satellite modem detected
bgpx-cli domains list | grep clearnet
# Expected: clearnet domain active; latency_class: satellite-leo (or satellite-geo)

# Check clearnet WAN connectivity
bgpx-cli node ping <bootstrap_node_id>
# Expected: pong received; RTT matches satellite latency class

# Verify geo plausibility exempt
bgpx-cli reputation show --self | grep geo_plausibility
# Expected: geo_plausibility: 0.50 (neutral/exempt for satellite nodes)

# Verify satellite metadata in advertisement
bgpx-cli node show --advertisement | grep satellite
# Expected: latency_class: satellite-leo

# Check bandwidth limit (if configured)
bgpx-cli node stats | grep bandwidth_cap
```

---

## 8. Failover Configuration

BGP-X supports WAN failover between satellite, cellular, and terrestrial links:

```toml
[[wan_interfaces]]
name = "fiber"
interface = "eth0"            # Primary terrestrial WAN
priority = 1                  # Highest priority (used first)
latency_class = "broadband"

[[wan_interfaces]]
name = "cellular"
interface = "wwan0"           # LTE modem
priority = 2                  # Used if eth0 fails
latency_class = "cellular"

[[wan_interfaces]]
name = "satellite"
interface = "usb0"            # Starlink USB
priority = 3                  # Used if cellular fails
latency_class = "satellite-leo"
wan_type = "satellite"
satellite_provider = "starlink"
```

Failover is automatic. BGP-X daemon monitors WAN interface health via keepalive probes to bootstrap nodes. If the primary interface becomes unreachable, it switches to the next priority interface and republishes its node advertisement with updated endpoint information.

**Session Resilience**: Active BGP-X sessions survive WAN failover — the path is rebuilt via the new WAN interface. Clients experience a brief reconnection (path rebuild time: typically 2-5 seconds).

**Note on Cost**: In failover configurations, satellite should typically have the lowest priority (highest number) due to higher latency and power consumption. Use it as backup for terrestrial/cellular failure.

---

## 9. Configuration Examples

### 9.1 BGP-X Router v1 as Mesh Island Gateway with Starlink WAN

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [
    { protocol = "udp", address = "0.0.0.0", port = 7474 }
]
wan_interface = "usb0"  # Starlink USB Ethernet

[[routing_domains]]
domain_type = "mesh"
island_id = "remote-mountain-community"
transports = ["lora", "wifi-mesh"]

[wan_interfaces]
primary = "usb0"
satellite = { provider = "starlink", latency_class = "satellite-leo" }

[exit]
enabled = true
policy_path = "/etc/bgpx/exit_policy.toml"
ech_capable = true
```

### 9.2 BGP-X Node v1 as Portable Field Relay with Iridium Backup WAN

```toml
[node]
role = "relay"

[[routing_domains]]
domain_type = "clearnet"
wan_interface = "ttyUSB0" # Iridium USB-serial

[[routing_domains]]
domain_type = "mesh"
island_id = "field-team-alpha"
transports = ["lora"]

[clearnet_transport]
wan_type = "satellite"
satellite_provider = "iridium-certus"
satellite_orbit = "leo"
latency_class = "satellite-leo"
max_bandwidth_kbps = 50
```

---

## 10. Bill of Materials — Satellite Extension (Per-Deployment)

| Component | Approximate Cost |
|---|---|
| Starlink Gen 3 Kit | $599 (hardware) + $120/mo service |
| USB-C to USB-A adapter | $5-10 |
| Weatherproof USB seal | $10-20 |
| Solar array (500W) | $300-600 |
| LiFePO4 battery (150Ah 24V) | $400-700 |
| Charge controller (MPPT) | $50-100 |
| Mounting hardware | $50-150 |
| **Total (Off-Grid Starlink Gateway)** | **$1,414-2,179** |

**Iridium Setup (Lower Cost)**:
| Component | Approximate Cost |
|---|---|
| Iridium Certus modem | $800-1500 |
| Antenna | $100-300 |
| Service (monthly) | $50-200 |
| Solar (100W panel + 30Ah battery) | $150-250 |
| **Total (Portable Iridium Relay)** | **$1,100-2,250** |
