# Installing BGP-X on a Compatible OpenWrt Router

**Version**: 0.1.0-draft
**License**: MIT (Software), CC BY 4.0 (Documentation)

---

## 1. Overview

If you already own a compatible OpenWrt router, you can install the BGP-X software package without purchasing new hardware. This guide walks through installing BGP-X on GL.iNet, Banana Pi, Raspberry Pi, and similar OpenWrt-compatible devices.

**BGP-X on OpenWrt runs the complete bgpx-node daemon.** It is not a "lite" or limited version of the software. The differences versus native BGP-X Router v1 hardware are primarily in hardware security (TPM), integrated radios, and first-boot setup experience—not in protocol capability.

**Compatible devices**: GL.iNet GL-MT6000 (Flint 2), GL-MT3000 (Beryl AX), AXT1800 (Slate AX), Banana Pi BPi-R3, Raspberry Pi 4/5, and x86 Linux servers. See `hardware/compatible_hardware.md` for the full list.

---

## 2. Hardware Prerequisites

Before installing, verify your hardware meets the minimum requirements for running the full BGP-X daemon:

| Requirement | Minimum | Recommended | Notes |
|---|---|---|---|
| **OpenWrt Version** | 23.05 | 23.05.3+ | Older versions lack required kernel features |
| **RAM** | 256 MB | 512 MB+ | Domain bridge nodes should have 512 MB+ |
| **Storage** | 4 GB | 8 GB+ | DHT cache, logs, keys |
| **CPU** | Dual-core ARM Cortex-A53 | Quad-core | Impacts encryption throughput |
| **WiFi** | 802.11s capable | WiFi 6 (802.11ax) | Only needed for WiFi mesh |
| **USB** | USB 2.0 | USB 3.0 | Required for LoRa/satellite adapters |

### Check Your OpenWrt Version

SSH into your router:

```bash
ssh root@192.168.1.1
cat /etc/openwrt_release
# DISTRIB_RELEASE should be "23.05" or newer
```

### Check RAM and Storage

```bash
free -h
df -h
```

If your device has **less than 256 MB RAM**, it cannot run the full `bgpx-node` daemon. Consider flashing **BGP-X Client Firmware** (Tier 2) instead—see `hardware/client_node_spec.md`.

If your router runs an older OpenWrt version, update OpenWrt first using the official OpenWrt upgrade guide for your device model.

---

## 3. Adding the BGP-X Package Repository

### 3.1 Add Repository Key

Download and install the BGP-X package signing key:

```bash
# Download signing key
wget -O /tmp/bgpx-signing.pub https://packages.bgpx.org/openwrt/key.pub

# Verify key fingerprint (optional but recommended)
# Expected fingerprint: [published at packages.bgpx.org]

# Install key
mkdir -p /etc/opkg/keys
mv /tmp/bgpx-signing.pub /etc/opkg/keys/
```

### 3.2 Add Repository Source

Determine your architecture:

```bash
uname -m
# aarch64: 64-bit ARM (GL-MT6000, BPi-R3, RPi 4/5)
# x86_64: 64-bit Intel/AMD
# armvirt: Virtual ARM
```

Add the repository:

```bash
# Replace $(uname -m) with your architecture if needed
echo "src/gz bgpx https://packages.bgpx.org/openwrt/23.05/$(uname -m)" \
    >> /etc/opkg/customfeeds.conf

# Update package lists
opkg update
```

---

## 4. Installing BGP-X

### 4.1 Core Packages

Install the BGP-X daemon, CLI, and LuCI web interface:

```bash
# Core daemon and CLI
opkg install bgpx-node bgpx-cli

# LuCI web interface plugin
opkg install bgpx-luci
```

### 4.2 WiFi Mesh Support (If Your Device Supports It)

If you plan to use WiFi 802.11s mesh:

```bash
# Check current wpad version
opkg info wpad-basic 2>/dev/null && {
    opkg remove wpad-basic wpad-basic-wolfssl
    opkg install wpad-mesh-openssl
}
```

### 4.3 USB LoRa Adapter Support (Domain Bridge)

If adding a USB LoRa adapter for domain bridge capability:

```bash
# USB serial drivers
opkg install kmod-usb-serial kmod-usb-serial-cp210x kmod-usb-serial-ftdi

# BGP-X modem driver interface
opkg install bgpx-modem-driver
```

### 4.4 Satellite Modem Support (Clearnet WAN)

For USB satellite modems (Starlink, Iridium):

```bash
# USB network drivers (usually present by default)
opkg install kmod-usb-net kmod-usb-net-cdc-ether
```

---

## 5. Initial Configuration

BGP-X configuration is stored in `/etc/bgpx/config.toml`. A default configuration is created during installation.

### 5.1 Generate Your Node Identity

```bash
bgpx-keygen
# Output:
# Generating Ed25519 keypair...
# Key backend: software (no TPM detected)
# Key stored at: /etc/bgpx/node.key
# Node ID: a3f2b9c1... (record this)
```

**Note on key security**: On third-party OpenWrt hardware without a TPM, your node's private key is stored in software. It is protected by `mlock()` (prevents swapping to disk) and explicit zeroization, but lacks hardware isolation. Keep your router physically secured.

### 5.2 Basic Configuration for Clearnet Relay

Edit the configuration file:

```bash
vi /etc/bgpx/config.toml
```

Minimal configuration for a clearnet relay node:

```toml
[node]
key_path = "/etc/bgpx/node.key"
key_backend = "software"
role = "relay"

[network]
listen_addr = "0.0.0.0"
listen_port = 7474

[[routing_domains]]
domain_type = "clearnet"
endpoints = [
  { protocol = "udp", address = "0.0.0.0", port = 7474 }
]

[sessions]
max_sessions = 10000
idle_timeout_secs = 90
```

---

## 6. Routing Policy Configuration

On a dual-stack router (your existing home router with BGP-X added), you need routing policy to decide which traffic goes through BGP-X and which goes directly.

### 6.1 Create Routing Policy File

```bash
vi /etc/bgpx/routing_policy.toml
```

### 6.2 Example: Privacy-First Policy

Route all traffic through BGP-X except local network and gaming:

```toml
# Default action: route through BGP-X
default_action = "bgpx"

# Bypass for local network (prevent routing loops)
[[rules]]
priority = 0
match.destination_cidr = [
    "192.168.0.0/16",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "127.0.0.0/8"
]
action = "bypass"

# Bypass BGP-X overlay traffic itself (prevent loop)
[[rules]]
priority = 1
match.destination_port = [7474]
match.protocol = "udp"
action = "bypass"

# Gaming: direct path for low latency
[[rules]]
priority = 10
match.destination_port = [3074, 27015, 27016, 27017, 27018, 27019, 27020]
match.protocol = "udp"
action = "standard"
reason = "gaming_low_latency"
```

### 6.3 Verify Policy

```bash
# Test which rule matches a destination
bgpx-cli policy test --destination 8.8.8.8
# Expected: action=bgpx (from default)

# Test gaming port
bgpx-cli policy test --destination 203.0.113.50 --port 3074
# Expected: action=standard (from gaming rule)
```

---

## 7. Pool Membership Configuration

If you want your node to participate in a curated or private pool:

### 7.1 Join a Pool

Edit `/etc/bgpx/config.toml`:

```toml
[pools]
member_pools = ["bgpx-default"]
# For a curated pool, replace with:
# member_pools = ["curated-eu-relays"]

# Fallback if pool insufficient
fallback_to_default = true
```

### 7.2 Verify Pool Membership

```bash
bgpx-cli pools list
bgpx-cli pools verify --pool-id bgpx-default
```

---

## 8. WiFi 802.11s Mesh Configuration

If your device WiFi supports 802.11s mesh:

### 8.1 Check Hardware Support

```bash
iw phy | grep mesh
# If output shows "mesh point": hardware supports 802.11s
```

### 8.2 Configure Mesh Interface

Edit `/etc/config/wireless`:

```bash
uci set wireless.mesh=wifi-iface
uci set wireless.mesh.device=radio0
uci set wireless.mesh.mode=mesh
uci set wireless.mesh.mesh_id="bgpx-mesh"
uci set wireless.mesh.encryption=sae
uci set wireless.mesh.key="your-secure-mesh-password"
uci commit wireless
wifi reload
```

### 8.3 Add Mesh Domain to BGP-X Config

Edit `/etc/bgpx/config.toml`:

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-id"  # Choose a unique, descriptive ID
transports = ["wifi-mesh"]
wifi_interface = "mesh0"
```

---

## 9. LoRa Domain Bridge Configuration

With both clearnet (WAN) and LoRa configured, your router becomes a **domain bridge node**, enabling clearnet users to reach LoRa mesh islands and vice versa.

### 9.1 Connect USB LoRa Adapter

Connect your Meshtastic USB adapter (flashed with BGP-X modem firmware) or other SX1262-based USB LoRa adapter:

```bash
# Verify device detected
ls /dev/ttyUSB* /dev/ttyACM*
# Expected: /dev/ttyUSB0 or /dev/ttyACM0

# Test communication
echo "VER" > /dev/ttyUSB0
cat /dev/ttyUSB0
# Expected: BGP-X Modem v0.1.0
```

### 9.2 Configure LoRa Domain

Edit `/etc/bgpx/config.toml`:

```toml
[[routing_domains]]
domain_type = "lora-regional"
region_id = "eu-868"               # Or "na-915", adjust for your region
transport = "serial"
serial_device = "/dev/ttyUSB0"
spreading_factor = 9
bandwidth_khz = 125
tx_power_dbm = 14
duty_cycle_percent = 1.0           # EU regulatory limit
```

### 9.3 Verify Domain Bridge Status

With both clearnet and LoRa domains configured:

```bash
bgpx-cli domains list
# Expected: clearnet, lora-regional

bgpx-cli domains bridges --health
# Expected: clearnet ↔ lora-regional: ONLINE
```

### 9.4 Test Cross-Domain Path

```bash
# Test path to a mesh island service
bgpx-cli paths test --destination "service.mesh.lima-district-1.bgpx"
```

---

## 10. Satellite WAN Integration

For users in remote locations, a USB satellite modem provides clearnet WAN.

### 10.1 Connect Satellite Modem

Connect your satellite terminal's USB port to the router. BGP-X auto-detects:

- **Starlink Gen 3**: Appears as USB Ethernet adapter (`usb0` or `eth1`)
- **Iridium Certus**: Appears as USB serial modem

### 10.2 Configure as Clearnet WAN

Satellite internet is **clearnet domain** (0x00000001) with high latency. No special BGP-X domain configuration is required—configure the interface in OpenWrt as you would any WAN connection.

### 10.3 Latency Considerations

BGP-X automatically detects the high latency and adjusts:
- Keepalive timeouts extended
- Congestion control baselines adjusted
- Geographic plausibility scoring **exempted** for satellite-class nodes

---

## 11. SDK Socket Access for LAN Devices

If you have applications on LAN devices (laptops, servers) that use the BGP-X SDK to control paths, they need to connect to the router's daemon.

### 11.1 Option A: SSH Socket Forwarding (Recommended for Development)

From your LAN device:

```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock root@192.168.1.1
```

Then configure your SDK app to connect to `/tmp/bgpx-router.sock`.

### 11.2 Option B: TCP Endpoint (For Production)

Edit `/etc/bgpx/config.toml`:

```toml
[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"
tcp_listen = "192.168.1.1:7475"  # LAN IP only, never WAN
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/sdk_auth_token"
```

Generate auth token:

```bash
openssl rand -hex 32 > /etc/bgpx/sdk_auth_token
chmod 600 /etc/bgpx/sdk_auth_token
```

Restart daemon:

```bash
/etc/init.d/bgpx restart
```

---

## 12. Geographic Plausibility (Optional)

Geographic plausibility scoring is **OPTIONAL**. If you want your node's claimed location to be verifiable via RTT measurements:

### 12.1 Declare Jurisdiction

Edit `/etc/bgpx/config.toml`:

```toml
[node]
# ... existing config ...
jurisdiction = "EU-DE"  # Your legal jurisdiction
```

### 12.2 Enable Scoring

```toml
[reputation]
geo_plausibility_enabled = true
geo_plausibility_warn_threshold = 2.0
geo_plausibility_alert_threshold = 3.0
```

If you **do not declare a jurisdiction**, geo plausibility scoring does not apply. This is the default and recommended for most users.

---

## 13. Encrypted Client Hello (ECH) Support

If your node is configured as an **exit node**, you can enable ECH to hide domain names from SNI:

### 13.1 Enable ECH

```toml
[gateway]
# ... exit policy config ...
ech_capable = true

[gateway.dns]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
```

### 13.2 Verify ECH Support

```bash
bgpx-cli exit-policy show
# Look for: ech_capable: true
```

---

## 14. Starting and Verifying

### 14.1 Start BGP-X

```bash
/etc/init.d/bgpx enable
/etc/init.d/bgpx start
```

### 14.2 Verify Operation

```bash
# Check daemon status
bgpx-cli node stats

# Check domains
bgpx-cli domains list

# Check DHT connectivity
bgpx-cli node dht-stats

# Check identity
bgpx-cli node identity
```

---

## 15. Accessing the Web Interface

The BGP-X LuCI plugin is available at your router's web interface:

`http://192.168.1.1` → **BGP-X** section in LuCI navigation

Features:
- Node status and statistics
- Domain bridge health
- Mesh island status
- Pool management
- Routing policy editor
- Reputation overview

---

## 16. Keeping BGP-X Updated

### 16.1 Manual Update

```bash
opkg update
opkg list-upgradable | grep bgpx
opkg upgrade bgpx-node bgpx-cli bgpx-luci
/etc/init.d/bgpx restart
```

### 16.2 Automatic Security Updates

```bash
opkg install attended-sysupgrade
# Configure auto-upgrade via LuCI → System → Attended Sysupgrade
```

---

## 17. Limitations vs. BGP-X Native Hardware

| Feature | BGP-X OpenWrt Package | BGP-X Router v1 / Node v1 |
|---|---|---|
| **TPM hardware key isolation** | ❌ (software key) | ✅ |
| **Integrated LoRa radio** | ❌ (USB adapter required) | ✅ |
| **Native 802.11s WiFi** | Depends on WiFi chip | ✅ |
| **Encrypted eMMC storage** | ❌ | ✅ |
| **Secure boot** | ❌ (depends on device) | ✅ |
| **First-boot setup wizard** | ❌ (manual config) | ✅ |
| **Factory test verification** | ❌ | ✅ |
| **Open hardware design** | N/A | ✅ (CERN-OHL-S) |
| **Full bgpx-node daemon** | ✅ | ✅ |
| **Domain bridge capability** | ✅ (with adapters) | ✅ (integrated) |
| **Pool membership** | ✅ | ✅ |
| **Cross-domain routing** | ✅ | ✅ |

**The most significant difference is key storage.** On third-party OpenWrt hardware, the BGP-X node key is stored in software—protected by OS-level memory locking, but without hardware isolation. For most community relay use cases, this is acceptable. For gateway nodes handling high-value community infrastructure, consider upgrading to BGP-X Router v1 with TPM.

---

## 18. Legal and Jurisdiction

- **You are responsible** for complying with local regulations regarding radio operation (LoRa frequencies, power limits, duty cycles) and encryption use.
- **Export controls** may apply to cryptographic software in your jurisdiction.
- BGP-X does not make illegal activities legal.

---

## 19. Next Steps

- **Configure routing policy**: `docs/routing_policy.md`
- **Set up domain bridge**: `docs/user_guide/domain_bridge_operator.md`
- **Join a pool**: `docs/user_guide/pool_operator.md`
- **Understand BGP-X Router v1 hardware**: `hardware/router_spec.md`
- **Deploy mesh island**: `docs/user_guide/mesh_island_setup.md`
