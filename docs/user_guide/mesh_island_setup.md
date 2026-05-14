# Setting Up a BGP-X Mesh Island

**Version**: 0.1.0-draft  
**License**: MIT (Code), CERN-OHL-S v2 (Hardware), CC BY 4.0 (Documentation)

---

## 1. What is a Mesh Island?

A BGP-X mesh island is a named, addressable community radio network operating as a **Routing Domain** within the BGP-X ecosystem. It is identified by a unique `island_id` (e.g., `lima-san-isidro-norte-2026`) and corresponds to the `mesh` domain type (0x00000003).

Mesh islands communicate via local radio transport—LoRa, WiFi 802.11s, or Bluetooth BLE—without requiring ISP connectivity between member nodes. Traffic within the island is protected by the same onion encryption used across the BGP-X network.

**Key capabilities:**

- **Private intra-island communication**: Members communicate with full end-to-end encryption, unlinkability, and forward secrecy. No island relay can link a sender to a destination.
- **Clearnet access via Gateway**: A gateway node (domain bridge) bridges the island to the BGP-X overlay and the public internet.
- **Hosting `.bgpx` services**: Island services are accessible globally by clearnet BGP-X users via domain bridge nodes—no mesh radio required on the client side.
- **Offline resilience**: The island remains fully operational for internal traffic if gateways lose clearnet connectivity.

You can start an island with as few as **two BGP-X nodes**. A minimal island has one gateway and one relay. A robust island has multiple gateways for redundancy and a distributed network of relay nodes for coverage.

---

## 2. Planning Your Island

### 2.1 Choose an Island ID

The `island_id` is your community's permanent, globally unique identifier within the BGP-X Unified DHT.

**Rules:**
- Lowercase letters, numbers, and hyphens only.
- 3–63 characters.
- Must be globally unique (the DHT detects collisions at registration time).

**Recommendation**: Use a descriptive, geographically specific name to avoid collisions and signal your community's location.

```
Good:     lima-san-isidro-norte-2026
Good:     berlin-friedrichshain-community
Avoid:    mesh-1, my-network, island-2026
```

### 2.2 Determine Your Coverage Goal

BGP-X supports multiple transport technologies. Your hardware and antenna placement determine coverage:

| Coverage Target | Recommended Hardware | Radio Technology | Expected Range |
|---|---|---|---|
| **One building or street** | 2× BGP-X Router v1 (Desktop) | WiFi 802.11s | 200m – 1 km |
| **One neighborhood (1–3 km)** | 1× Gateway Node v1 + 2× Relay Node v1 | LoRa (elevated antennas) | 5 – 15 km |
| **One district (3–10 km)** | 2× Gateway Node v1 + 4–6× Relay Node v1 (masts) | LoRa + WiFi Mesh | 10 – 30 km |
| **One city (10–30 km)** | 3+ Gateways + 10+ Relay Nodes (towers) | LoRa + WiFi | 20 – 50+ km |

**Key physics:**
- **LoRa**: Long range, high latency, low bandwidth (Kbps). Ideal for rural and semi-rural.
- **WiFi 802.11s**: Short range, low latency, high bandwidth (Mbps). Ideal for dense urban "street meshes."
- **Both**: Use LoRa for long-range backhaul and WiFi for local high-speed connectivity.

### 2.3 Transport Technology Decision

| Transport | Pros | Cons | Best For |
|---|---|---|---|
| **LoRa** | Long range (5–30 km); penetrates obstacles; low power | Low bandwidth; high latency (1–5s RTT) | Rural, semi-rural, long-range backhaul |
| **WiFi 802.11s** | High speed; low latency; standard hardware | Short range; line-of-sight helps | Dense urban neighborhoods, events, campuses |
| **BLE** | Low power; ubiquitous in mobile devices | Very short range; low bandwidth | Personal area networks, sensors, wearables |

**Recommendation**: Most community islands should deploy **both LoRa and WiFi mesh**. LoRa provides the long-range backbone; WiFi provides local high-speed coverage.

---

## 3. Hardware Selection

BGP-X provides hardware for every role in the island ecosystem.

### 3.1 Gateway Nodes (Domain Bridges)

**Purpose**: Bridge your mesh island domain (`mesh:<island_id>`) to the clearnet domain (`0x00000001`). Publishes the island's existence to the Unified DHT.

**Hardware Options:**

| Hardware | Pros | Cons | Best For |
|---|---|---|---|
| **BGP-X Router v1** | All-in-one; easy setup; outdoor variant available | Higher cost ($80–120) | Primary gateway for robust islands |
| **BGP-X Node v1** | Lower cost; outdoor-native; solar-ready | No built-in LAN switch | Secondary gateways, remote sites |
| **OpenWrt-compatible router + USB LoRa** | Reuses existing hardware | More setup; no integrated TPM | Budget deployments, experimentation |

**Clearnet Uplink**: A gateway requires an ISP connection. This can be:
- Fiber, cable, or DSL
- Cellular (4G/5G)
- **Satellite internet** (Starlink, Iridium, Inmarsat). **Note**: BGP-X treats commercial satellite internet as clearnet domain (0x00000001), same as fiber, just with higher latency. This is ideal for remote islands.

### 3.2 Relay Nodes

**Purpose**: Extend mesh coverage. Forward encrypted traffic. Participate in path construction. No clearnet connection needed.

**Hardware Options:**

| Hardware | Pros | Cons | Best For |
|---|---|---|---|
| **BGP-X Node v1** | Outdoor IP67; solar-native; full bgpx-node daemon | Requires antenna, power setup | Community relays on rooftops, masts |
| **BGP-X Router v1 (Outdoor)** | AIO; simple deployment | Higher power draw | If you have power, want simpler setup |
| **OpenWrt router + USB LoRa** | Low cost | No outdoor enclosure by default | Temporary relays, indoor window setups |

**Important**: BGP-X Node v1 runs the **complete** bgpx-node daemon. It is not a "lite" device. It performs full onion encryption, DHT participation, and path construction.

### 3.3 Client Nodes (Community Members)

**Purpose**: Individual users connecting to the island. Do not route for others.

| Hardware | Description | Cost |
|---|---|---|
| **BGP-X Client Node (Tier 2)** | LILYGO T3S3, T-Beam, Heltec V3, RAK WisBlock. Battery-powered. Runs BGP-X Client Firmware (subset protocol). | $25–40 |
| **BGP-X Adapter/Dongle (Tier 3)** | USB LoRa modem. Host computer (laptop/desktop) runs full BGP-X daemon. | $15–25 |
| **BGP-X Router v1** | Future-proof; can become a relay if user moves | $80–120 |

---

## 4. Setting Up the First Gateway Node

The gateway establishes your island's bridge to the global BGP-X network.

### 4.1 BGP-X Router v1 as Gateway

1.  **Physical Setup**: Follow the Quick Start Guide. Connect WAN to your ISP (fiber, cellular, or satellite).
2.  **Web UI Setup**:
    - Navigate to **BGP-X → Node Setup**.
    - **Node Role**: Select `Relay` and `Domain Bridge`.
    - **Island ID**: Enter your chosen ID (e.g., `lima-san-isidro-norte-2026`).
    - **Routing Domains**: Enable `Mesh` and configure transports (LoRa frequency, WiFi mesh SSID).
    - **Exit Policy**: Configure if providing clearnet exit for island members.
3.  **Apply and Restart**.

The router configures itself as a **domain bridge node** between the clearnet and your mesh island. It automatically publishes a `MESH_ISLAND_ADVERTISE` record to the **Unified DHT**, making your island discoverable globally.

### 4.2 BGP-X Node v1 as Gateway

SSH into the node and edit `/etc/bgpx/config.toml`:

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "0.0.0.0", port = 7474 }]

[[routing_domains]]
domain_type = "mesh"
island_id = "lima-san-isidro-norte-2026"
transports = ["lora", "wifi-mesh"]
wifi_interface = "mesh0"
lora_device = "/dev/ttyUSB0"     # Adjust based on your hardware
lora_frequency_mhz = 915.0      # Set to your region's frequency
lora_spreading_factor = 10      # Adjust for range vs speed

[exit]
enabled = true
policy_path = "/etc/bgpx/exit_policy.toml"
```

Restart the BGP-X daemon:
```bash
/etc/init.d/bgpx restart
```

### 4.3 Verify Island Publication

Your gateway should publish your island to the Unified DHT. Verify:

```bash
bgpx-cli islands show lima-san-isidro-norte-2026 --dht-freshness
```

Expected output indicates the island record was found and is fresh (< 48 hours old).

### 4.4 Antenna Placement

Gateway LoRa antennas should be elevated:
- **Minimum**: 5–8 meters above ground level (rooftop).
- **Optimal**: 15–30 meters (mast or tower).
- **Line of sight** to relay nodes maximizes range.

---

## 5. Adding Relay Nodes

Relay nodes extend your island's coverage. They participate fully in the BGP-X overlay, forwarding traffic and helping construct paths.

### 5.1 Configuration for Relay Node (No WAN)

Edit `/etc/bgpx/config.toml`:

```toml
[node]
role = "relay"

[[routing_domains]]
domain_type = "mesh"
island_id = "lima-san-isidro-norte-2026"  # Must match your gateway's island_id
transports = ["lora", "wifi-mesh"]
```

No `exit` section is needed—relay nodes do not provide clearnet access.

### 5.2 Automatic Bootstrap via MESH_BEACON

When a relay node powers on within radio range of any existing island node:

1.  It listens for **MESH_BEACON** broadcasts.
2.  It verifies Ed25519 signatures on received beacons.
3.  It joins the island's DHT routing table via the beaconing node.
4.  It begins accepting paths and forwarding traffic.

**No manual join is required.** Authentication is handled by cryptographic signatures on beacons and node advertisements.

### 5.3 Verify Relay Joined

From any island node:

```bash
bgpx-cli node ping <new_relay_node_id> --domain mesh:lima-san-isidro-norte-2026
```

From the gateway:

```bash
bgpx-cli islands show lima-san-isidro-norte-2026
# active_relay_count should have increased.
```

---

## 6. Connecting Community Members (Client Nodes)

### 6.1 BGP-X Client Node (Tier 2)

1.  **Flash BGP-X Client Firmware** (ESP-IDF/Arduino) onto your LILYGO T-Beam/T3S3.
2.  **Configure Island ID** via the device's USB serial console or companion app.
3.  **Power On** within LoRa range of any island relay or gateway.
4.  The client node establishes a path through the island to reach services.

**Limitation**: Client nodes do not route traffic for others. They are endpoints.

### 6.2 BGP-X Adapter/Dongle (Tier 3)

1.  Plug the dongle into a laptop or desktop.
2.  The host computer detects it as a USB serial device.
3.  Run the BGP-X daemon on the host in "adapter mode":
    ```bash
    bgpx-node --config /etc/bgpx/adapter.toml
    ```
4.  The daemon routes BGP-X traffic through the USB LoRa interface.

---

## 7. Satellite WAN for Remote Islands

If your island is in a location without fiber or cellular, use satellite internet for the gateway's clearnet uplink.

**Supported satellite services**:
- **Starlink**: Connect Starlink Gen 3 terminal's Ethernet port to your gateway's WAN port. BGP-X treats this as clearnet with LEO latency (20–60ms).
- **Iridium Certus**: Use a compatible Iridium terminal as a WAN interface via serial/IP bridge. BGP-X treats this as clearnet with GEO latency (600ms+).

**Configuration**:
No special BGP-X configuration is needed. The gateway sees a standard IP connection. Your `island_id` remains the same. Clearnet users will experience higher latency when accessing your island services, but connectivity is established.

---

## 8. Setting Up a Second Gateway (Redundancy)

A single gateway is a **single point of failure**. If it goes offline, your island loses clearnet connectivity. **Redundant gateways are strongly recommended.**

1.  Deploy a second BGP-X Node v1 or Router v1 at a different physical location, ideally with a different ISP or transport type.
2.  Configure it with the **same `island_id`**.
3.  BGP-X automatically discovers both gateways via the Unified DHT.
4.  Path construction selects between them based on latency and availability.

**Verify**:
```bash
bgpx-cli islands bridges lima-san-isidro-norte-2026
# Should show 2 bridge nodes, both ONLINE.
```

---

## 9. Hosting .bgpx Services Within the Island

Any BGP-X node in your island can host a native `.bgpx` service. These services are accessible:
- From within the island (by island members).
- From the clearnet (by any BGP-X user worldwide, via your domain bridge node).

### 9.1 Register a Service

Using the SDK (Rust example):

```rust
let listener = client.register_service(ServiceConfig {
    name: "island-news".to_string(),
    advertise: true,    // Published to Unified DHT via your gateway
    ..Default::default()
}).await?;

println!("Service: bgpx://{}.bgpx/", listener.service_id().to_hex());
```

Using the CLI:

```bash
bgpx-serve --dir /var/www/html --name island-news --advertise
# Serving at: bgpx://a3f2b9....bgpx/
```

### 9.2 HTTP/2 Native Transport

`.bgpx` services use **HTTP/2 over BGP-X streams**. This is efficient for LoRa paths because HTTP/2 multiplexes multiple requests over a single stream, reducing round-trips (critical when each RTT is 1–5 seconds).

---

## 10. Cross-Domain Reachability

### 10.1 Clearnet Users Accessing Island Services

A clearnet BGP-X user (on fiber/cellular anywhere in the world) can reach your island's `.bgpx` services without any mesh hardware.

1.  The clearnet user's client queries the **Unified DHT**.
2.  The DHT returns your island's service record, which includes the NodeID of your domain bridge gateway.
3.  The client constructs a cross-domain path: `[Clearnet Relays] → [Your Gateway] → [Island Relay] → [Service]`.
4.  Traffic is onion-encrypted end-to-end. The clearnet user never sees the island's mesh IP addresses.

### 10.2 Inter-Island Connectivity

Two islands that cannot directly communicate via radio can connect via the clearnet overlay.

- Island A's gateway publishes to Unified DHT.
- Island B's gateway discovers Island A via DHT.
- Path: `Island A Member → Island A Gateway → [Clearnet Overlay] → Island B Gateway → Island B Member`.

No special configuration is required—this is native to BGP-X's cross-domain routing architecture.

---

## 11. Monitoring Island Health

### 11.1 Quick Status

```bash
bgpx-cli islands health lima-san-isidro-norte-2026
```

Output:
```
Island: lima-san-isidro-norte-2026
Status: ONLINE
Active relays: 8
Bridge nodes: 2 (both online)
DHT advertisement: fresh (18 hours old)
Last clearnet connectivity test: PASS (2 minutes ago)
Services registered: 3
```

### 11.2 Monitoring Alerts

| Condition | Implication | Action |
|---|---|---|
| **Bridge nodes: 1** | Single point of failure | Contact second gateway operator; diagnose issue |
| **Bridge nodes: 0** | Island isolated from clearnet | Restore gateway connectivity urgently |
| **Active relays declining** | Coverage degradation | Check power to relays; check LoRa reception |
| **DHT advertisement age > 20 hours** | Gateway not republishing | Check gateway's internet connection |

---

## 12. Offline Operation

If all gateways go offline, your island continues to function internally:

- **Intra-island traffic** continues with full privacy.
- **Island DHT** remains accessible via MESH_BEACON bootstrap among island members.
- **`.bgpx` services** within the island remain reachable by island members.
- **Cross-domain traffic** (clearnet access, external services) is unavailable.

**Recovery**: When a gateway comes back online, it immediately republishes `MESH_ISLAND_ADVERTISE` and `DOMAIN_ADVERTISE` records to the Unified DHT. Cross-domain connectivity restores within ~60 seconds.

---

## 13. Inviting Community Members

Share your `island_id` with community members who have BGP-X hardware.

**Connection card template**:

```
Community Mesh: BGP-X
Island ID: lima-san-isidro-norte-2026
Gateway: Operated by [Organization/Individual]
LoRa Frequency: 915 MHz
WiFi Mesh SSID: BGP-X-Island
Contact: [email/matrix/phone]
```

Members set their `island_id` in their device configuration. When within radio range, they automatically join the island via the beacon protocol.

---

## 14. Best Practices

1.  **Elevate antennas**: Every 3m of antenna height significantly improves LoRa range.
2.  **Diversify gateways**: Use different ISPs or transport types (e.g., fiber + cellular).
3.  **Monitor power**: Solar-powered relay nodes need battery reserves for cloudy periods.
4.  **Test paths regularly**: Use `bgpx-cli paths test` to verify clearnet connectivity through your gateways.
5.  **Educate members**: Ensure community members understand that high latency on LoRa paths is normal and expected.
