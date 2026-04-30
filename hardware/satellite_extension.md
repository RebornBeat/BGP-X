# BGP-X Satellite Modem Extension

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X Router v1 and BGP-X Node v1 (with WAN) support satellite internet modems via USB, enabling deployment in locations without fiber or cellular connectivity. Satellite internet connections are **clearnet** in BGP-X — domain type 0x00000001. The satellite provides the WAN; BGP-X overlay runs on top identically to fiber or cellular.

This document covers hardware integration specifics: supported satellite modems, USB auto-detection, driver configuration, power requirements, and latency class configuration.

---

## 2. Supported Satellite Modems

### 2.1 Starlink Gen 3 (Recommended)

| Parameter | Value |
|---|---|
| USB Vendor ID | `0x48bf` |
| USB Product ID | `0x0800` |
| Interface | USB Ethernet (RNDIS / CDC-NCM) |
| IP assignment | DHCP from Starlink router |
| Bandwidth | 20-200 Mbps down, 5-20 Mbps up |
| Latency | 25-60 ms |
| BGP-X latency class | `satellite-leo` |
| Power draw | 50-75 W (dish + router combined) |
| Setup | Plug USB-C from dish to BGP-X hardware USB-A port via adapter |

**Driver**: standard Linux USB Ethernet CDC-NCM driver (included in OpenWrt 23.05 and all Linux 5.15+ kernels). No custom driver needed.

**Auto-detection**: BGP-X daemon detects USB vendor/product ID at startup and records `wan_type = starlink` in transport metadata.

```toml
# Auto-generated when Starlink USB detected:
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "starlink"
satellite_orbit = "leo"
latency_class = "satellite-leo"
detected_interface = "usb0"
```

**Starlink Gen 3 dish mounting**: requires clear sky view (>100° azimuth cone). Self-orienting electronically (no mechanical motor). Roof mount, pole mount, or deck mount. BGP-X hardware can be co-located with or remote from the dish (up to 25m Ethernet cable between dish and BGP-X hardware via Starlink's provided cable or Cat6 replacement).

### 2.2 Iridium Certus 100 / 350 / 700

| Parameter | Value |
|---|---|
| Interface | Serial (RS-232 or USB-serial via USB adapter) |
| Linux device | `/dev/ttyUSB0` (USB-serial) or `/dev/ttyS0` (RS-232) |
| Bandwidth | 88 Kbps up / 704 Kbps down (Certus 700 max) |
| Latency | 30-80 ms (LEO, but ground station routing adds latency) |
| BGP-X latency class | `satellite-leo` |
| Power draw | 5-15 W (modem only; antenna separate) |
| Coverage | Global including polar |

**Driver**: standard Linux USB-serial driver (cp210x or ch341 depending on modem's USB-serial chip). Included in OpenWrt 23.05.

**AT command interface**: BGP-X satellite transport driver communicates with Iridium modem via AT commands:
```
AT+IPR=115200        Set baud rate
AT+SBDWT=...         Write data to modem buffer
AT+SBDIX             Initiate SBD session (send/receive)
AT+SBDRT             Read received data
AT+CSQ               Check signal quality
```

**Bandwidth limitation**: at 88-704 Kbps, Iridium Certus is suitable for BGP-X DHT participation and light relay duty. Configure bandwidth cap:

```toml
[clearnet_transport]
wan_type = "satellite"
satellite_provider = "iridium-certus"
satellite_orbit = "leo"
latency_class = "satellite-leo"
max_bandwidth_kbps = 50         # Conservative; leaves headroom
min_session_count = 1           # Limit concurrent BGP-X sessions
```

### 2.3 Inmarsat BGAN / Fleet One

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

GEO latency (600ms+) adds significantly to BGP-X path RTT. All PATH_QUALITY_REPORT entries for this node's clearnet domain will show GEO-class latency — expected and not flagged as congestion. Geo plausibility scoring: satellite-geo exempt (returns 0.5 neutral).

### 2.4 Generic LTE/5G USB Modems (Fallback)

Any USB LTE or 5G modem that presents as a USB Ethernet interface or USB-serial modem is compatible. These are not satellite — they are cellular (clearnet, latency class `cellular`).

```toml
[clearnet_transport]
wan_type = "cellular"
latency_class = "cellular"
```

Auto-detected via USB Ethernet vendor IDs for common modems (Huawei HiLink, Quectel EC25, Sierra Wireless EM7411, etc.).

---

## 3. USB Auto-Detection

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
1. Driver loads automatically (USB Ethernet: CDC-NCM; USB-serial: appropriate serial driver)
2. Network interface created (e.g., `usb0` for Ethernet, BGP-X serial channel for AT command modems)
3. DHCP client started on USB interface
4. BGP-X clearnet transport configured with satellite provider metadata
5. Geo plausibility scoring: satellite latency class automatically exempts this node
6. Node advertisement updated: clearnet endpoint with `latency_class: satellite-leo` metadata

---

## 4. Power Requirements

Satellite terminals have significant power requirements. BGP-X hardware power is additive:

| Configuration | BGP-X Hardware | Satellite Terminal | Total |
|---|---|---|---|
| Node v1 + Starlink Gen 3 | 3-8 W | 50-75 W | 53-83 W |
| Router v1 + Starlink Gen 3 | 5-15 W | 50-75 W | 55-90 W |
| Node v1 + Iridium Certus 100 | 3-8 W | 5-10 W | 8-18 W |
| Node v1 + Inmarsat BGAN | 3-8 W | 15-30 W | 18-38 W |

**Solar sizing for Starlink + BGP-X**:
- Total load: 60-85 W average
- Daily energy at 14 hours operation: 840-1190 Wh
- Solar panel (5 peak sun hours/day): 840 Wh ÷ 5h = 168 W minimum panel
- Recommended: 250-300 W panel array
- Battery (2-day reserve at 85 W average): 85 W × 48 h = 4080 Wh = 170 Ah at 24V LiFePO4
- This is a substantial off-grid power system; plan carefully before remote Starlink deployment

**Iridium + BGP-X solar** is far more achievable:
- Total load: ~15 W average
- 50W panel + 40Ah LiFePO4 = comfortable 24/7 operation with ≥3 hours daily sun

---

## 5. Connectivity Verification

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

## 6. Failover Configuration

BGP-X supports WAN failover between satellite and cellular (or multiple satellite modems):

```toml
[[wan_interfaces]]
name = "primary"
interface = "eth0"            # Fiber or cable WAN
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

Active BGP-X sessions survive WAN failover — path is rebuilt via the new WAN interface. Clients experience a brief reconnection (path rebuild time: typically 2-5 seconds).
