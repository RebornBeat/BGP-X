# BGP-X Mesh Transport Specification

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

This document specifies the BGP-X mesh transport abstraction layer — the interface for transporting BGP-X datagrams over non-IP transport media including WiFi 802.11s, LoRa radio, Bluetooth BLE, and Ethernet point-to-point. It also specifies how mesh islands function as first-class routing domains within the unified BGP-X network.

---

## 1. Mesh as a First-Class Routing Domain

BGP-X mesh transport is a first-class routing domain — named, addressable, discoverable from all other domains, and capable of hosting BGP-X services accessible from anywhere in the BGP-X network.

A mesh island is:
- Identified by a unique `island_id` string (operator-chosen, globally unique via DHT collision detection)
- Advertised in the unified BGP-X DHT when a bridge node is online
- Reachable from clearnet clients without any special client hardware
- Capable of hosting BGP-X native services accessible from any domain
- An independent operating unit when bridge nodes are offline

This is foundational: mesh is not an add-on to BGP-X — it is an equal entry point.

---

## 2. Transport Abstraction

### 2.1 MeshTransport Interface

All mesh transport implementations MUST implement this interface:

```rust
pub trait MeshTransport: Send + Sync {

    /// Send a BGP-X datagram to a mesh peer (identified by NodeID)
    async fn send(
        &self,
        peer_node_id: &NodeId,
        data: &[u8],
    ) -> Result<(), TransportError>;

    /// Receive a BGP-X datagram (returns sender NodeID + data)
    async fn recv(&self) -> Result<(NodeId, Vec<u8>), TransportError>;

    /// Broadcast to all reachable mesh peers (for MESH_BEACON)
    async fn broadcast(&self, data: &[u8]) -> Result<usize, TransportError>;

    /// Get this node's mesh address on this transport
    fn local_addr(&self) -> MeshAddr;

    /// Transport capabilities
    fn capabilities(&self) -> TransportCapabilities;

    /// Maximum payload size for this transport (before fragmentation)
    fn mtu(&self) -> usize;

    /// Transport type identifier
    fn transport_type(&self) -> TransportType;

    /// Returns the mesh island domain ID if this transport is part of a mesh domain
    fn domain_id(&self) -> Option<DomainId>;
}

pub enum TransportType {
    UdpIp,
    WifiMesh,
    LoRa,
    BluetoothBle,
    EthernetP2P,
}

pub struct TransportCapabilities {
    pub mtu_bytes: usize,
    pub bandwidth_kbps: f32,
    pub latency_ms_typical: f32,
    pub broadcast_support: bool,
    pub max_range_meters: Option<f32>,
    pub requires_fragmentation: bool,  // true if MTU < 1280
    pub latency_class: LatencyClass,
}

pub enum LatencyClass {
    LowLatency,    // <20ms typical (WiFi, Ethernet)
    MediumLatency, // 20-200ms (some radio links)
    HighLatency,   // 200ms-5s (LoRa, satellite)
}
```

---

## 3. MeshAddr Format

Mesh nodes are addressed by NodeID (32 bytes), not IP addresses. The transport layer handles mapping NodeID to transport-specific addresses.

```rust
pub struct MeshAddr {
    pub node_id: NodeId,                     // Primary identifier
    pub transport_hint: Option<TransportType>, // Preferred transport
    pub transport_addr: Option<Vec<u8>>,     // Transport-specific address
}
```

Transport-specific address formats:

| Transport | Address Format | Size |
|---|---|---|
| WiFi 802.11s | MAC address (6 bytes) | 6 bytes |
| LoRa | LoRa device address (4 bytes) | 4 bytes |
| Bluetooth BLE | BLE MAC address (6 bytes) | 6 bytes |
| Ethernet P2P | MAC address (6 bytes) | 6 bytes |

NodeIDs are the primary addresses. Transport-specific addresses are cached from MESH_BEACON messages and used as delivery hints.

---

## 4. Supported Transports

### 4.1 WiFi 802.11s Mesh

```
Protocol:          IEEE 802.11s (mesh networking extension to 802.11)
Addressing:        MAC-based within mesh; NodeID at BGP-X layer
MTU:               1280 bytes (same as UDP/IP target)
Bandwidth:         10-100+ Mbps per hop
Typical latency:   1-20ms per hop
Range:             50-200m per hop (indoor/urban), km with directional antennas
Multi-hop:         Yes (802.11s handles IP routing within segment)
Regulatory:        2.4 GHz and 5 GHz ISM bands, license-free globally
Fragmentation:     Not required (MTU sufficient)
```

**Interface**: Linux kernel mac80211_hwsim or actual wireless hardware configured in mesh mode via iw/hostapd.

**BGP-X binding**: BGP-X daemon creates a raw socket on the mesh interface and sends/receives BGP-X datagrams directly, bypassing the IP layer.

### 4.2 LoRa Radio

```
Protocol:          Raw LoRa (not LoRaWAN)
Addressing:        LoRa device address (4 bytes) + NodeID mapping
MTU:               Frequency/SF-dependent:
                   SF7:  ~200 bytes usable
                   SF8:  ~200 bytes usable
                   SF9:  ~150 bytes usable
                   SF10: ~100 bytes usable
                   SF11: ~80 bytes usable
                   SF12: ~50 bytes usable
Bandwidth:         0.3-50 Kbps (SF-dependent)
Typical latency:   100ms-5s per hop (duty cycle constraints)
Range:             5-50 km (rural open terrain), 1-5 km urban
Multi-hop:         Yes (BGP-X handles routing; LoRa handles individual hops)
Regulatory:        Sub-GHz ISM bands; duty cycle limits (1% in EU)
Fragmentation:     REQUIRED (MESH_FRAGMENT 0x19)
```

**Interface**: SPI or USB connection to LoRa module (SX1262, SX1276, etc.) via Linux driver or userspace library.

**BGP-X binding**: BGP-X daemon communicates with LoRa module via LoRa driver. Sends BGP-X MESH_FRAGMENT packets (one LoRa frame per fragment).

**Duty cycle compliance**: BGP-X LoRa implementation respects duty cycle limits by rate-limiting transmission. A token bucket controls transmission rate:

```
EU duty cycle: 1% → 36 seconds/hour transmit time
At SF7 (~200 bytes, ~250ms airtime): ~144 packets/hour maximum
BGP-X LoRa mode: prioritize DHT sync and key exchange; data transfer limited
```

### 4.3 Bluetooth BLE (Bluetooth Low Energy)

```
Protocol:          Bluetooth mesh profile or custom BLE
MTU:               100-200 bytes (Bluetooth ATT MTU, typically 100-244 bytes)
Bandwidth:         1-2 Mbps (but ATT MTU limits effective throughput)
Typical latency:   10-100ms per hop
Range:             10-100m per hop
Multi-hop:         Yes (Bluetooth mesh routing)
Regulatory:        2.4 GHz ISM band, globally license-free
Fragmentation:     REQUIRED (MESH_FRAGMENT 0x19)
```

**Use case**: Short-range IoT device mesh; mobile device pairing; building-scale coverage.

**Limitation**: Bluetooth mesh has significantly more overhead than WiFi. BGP-X over BLE is suitable for control traffic and small messages, not for high-throughput data.

### 4.4 Ethernet Point-to-Point

```
Protocol:          Standard Ethernet (802.3) or fiber
MTU:               1500 bytes (Ethernet standard)
                   BGP-X still targets 1280 bytes for inter-transport compatibility
Bandwidth:         100 Mbps to 10 Gbps
Typical latency:   <1ms (copper), <5ms (moderate fiber runs)
Range:             100m copper, 100km+ fiber with repeaters
Multi-hop:         N/A (direct P2P links)
Regulatory:        No RF regulatory issues
Fragmentation:     Not required
```

**Use case**: High-bandwidth backbone between fixed mesh locations (buildings, neighborhoods, data centers).

**BGP-X binding**: BGP-X daemon creates a raw Ethernet socket. Each BGP-X node on the link is identified by MAC address + NodeID mapping from MESH_BEACON.

---

## 5. Mesh Island as Routing Domain

### 5.1 Island Identity

Each mesh island has:
- An `island_id` string (operator-chosen, globally unique)
- A `domain_id` (type `0x00000003` + BLAKE3(island_id)[0:4])
- A set of relay nodes discoverable via unified DHT with domain filter
- Zero or more domain bridge nodes connecting to other domains
- An optional service registry published in DHT records

### 5.2 Island DHT Participation

When a bridge node is online (internet-connected):
- Bridge node publishes island advertisement to unified internet DHT (MESH_ISLAND_ADVERTISE)
- Bridge node stores mesh-domain node advertisements in internet DHT on behalf of mesh-only nodes
- Internet clients can discover mesh island relay nodes via domain-filtered DHT queries
- Mesh-only nodes' advertisements are served from internet DHT by the bridge node

When no bridge node is online (offline mode):
- Island operates with local mesh DHT (bootstrap via MESH_BEACON broadcast)
- Internet clients cannot reach island
- Intra-island traffic fully functional

### 5.3 Clearnet Client Accessing Mesh Island Service

This is the key capability. A clearnet client with no mesh hardware:

1. Queries unified DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
2. Constructs path: `[clearnet relays] → [B: domain bridge] → [mesh relays] → [service]`
3. Bridge node B receives clearnet UDP, decrypts DOMAIN_BRIDGE layer, forwards via radio to first mesh relay
4. Service receives traffic with no knowledge of the client's clearnet IP
5. Return path via cross-domain path_id routing

**No BGP-X router required on the client.** The BGP-X daemon on the client's device handles path construction. The bridge node handles all radio transmission.

---

## 6. Unified DHT for Mesh (Replaces Two-DHT Model)

BGP-X operates ONE unified Kademlia-style DHT spanning all routing domains. All nodes participate in the same key space regardless of transport or domain. There is no separate mesh DHT.

**Previous model (deprecated)**: Internet DHT + mesh DHT with gateway sync. This is replaced.

**Current model (unified)**:
- All nodes participate in the same Kademlia key space
- Bridge nodes (including gateway nodes) store and serve records for mesh-only nodes in the unified internet DHT
- Mesh-only nodes access the unified DHT via their bridge nodes' caches
- No synchronization between separate DHTs needed — there is only one DHT

When bridge node is online: any pending mesh island record updates are published to unified DHT automatically.
When bridge node is offline: records remain valid until TTL; new records cannot be published until bridge reconnects.

---

## 7. Multi-Transport Operation

### 7.1 Transport Selection Algorithm

A single BGP-X node may operate multiple transports simultaneously. The node advertises all available transports in its mesh_endpoints or routing_domains.

When a node needs to send a packet to a peer:

```python
function select_transport(peer_node_id, packet_size, latency_class_required, target_domain):

    # If routing to another domain, prefer that domain's transport
    if target_domain and target_domain != current_domain:
        bridge_transport = get_bridge_transport_to(target_domain)
        if bridge_transport:
            return bridge_transport

    # Get all transports that can reach this peer
    available = transports_for_peer(peer_node_id)

    if not available:
        return ERROR("Peer not reachable")

    # For DHT control traffic: prefer LoRa (long range) or WiFi mesh
    if latency_class_required == HIGH_LATENCY_OK:
        preferred = [t for t in available if t.type in [LORA, WIFI_MESH]]

    # For data traffic: prefer low-latency transports
    elif latency_class_required == LOW_LATENCY:
        preferred = [t for t in available if t.latency_class == LOW_LATENCY]

    # For fragmented packets: prefer transport with suitable MTU
    if packet_size > 200 and LORA in available:
        preferred = [t for t in preferred if t.type != LORA or t.mtu >= packet_size / 16]

    # Fall back to any available if preferred empty
    if not preferred:
        preferred = available

    # Select transport with highest available bandwidth
    return max(preferred, key=lambda t: t.capabilities.bandwidth_kbps)
```

### 7.2 Transport Failover

If a transport fails (no response within timeout):

1. Mark transport as temporarily unavailable
2. Retry with alternative transport if available
3. If all transports fail: KEEPALIVE timeout → path rebuild

### 7.3 Inter-Island Routing

Two mesh islands connect via:

1. **Via clearnet overlay**: mesh-A → bridge A → clearnet relays → bridge B → mesh-B. Both islands need at least one internet-connected bridge node.

2. **Via direct island-to-island bridge**: a node with two mesh transport interfaces (e.g., WiFi mesh for island A + LoRa for island B) bridges the islands without clearnet:

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "island-a"
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"

[[routing_domains]]
domain_type = "mesh"
island_id = "island-b"
transports = ["lora"]
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true
bridges = [
    { from = "mesh:island-a", to = "mesh:island-b" }
]
```

This node bridges two mesh islands directly using radio. No clearnet needed. The bridge pair is published to the unified DHT — clients can use this direct bridge for mesh-A → mesh-B paths.

---

## 8. MESH_BEACON Protocol

Beacons enable DHT bootstrap in mesh-only networks and advertise transport capabilities.

### 8.1 Beacon Timing

- Broadcast every 30 seconds on all active transports
- Jitter: ±5 seconds to prevent synchronized broadcasts

### 8.2 Wire Format

```
Common Header (Msg Type = 0x18, Session ID = 0x00...00)
├───────────────────────────────────────────────────────────────┤
│                   NodeID (32 bytes)                           │
├───────────────────────────────────────────────────────────────┤
│                   Ed25519 Public Key (32 bytes)               │
├───────────────────────────────────────────────────────────────┤
│              Transports Supported (1 byte bitmask)            │
│    Bit 0: UDP/IP  Bit 1: WiFi mesh  Bit 2: LoRa              │
│    Bit 3: BLE     Bit 4: Ethernet P2P                         │
├───────────────────────────────────────────────────────────────┤
│              DHT Routing Table Size (2 bytes, uint16)         │
├───────────────────────────────────────────────────────────────┤
│              bridge_capable (1 byte): 0x00 = no, 0x01 = yes   │
├───────────────────────────────────────────────────────────────┤
│              served_domains_count (1 byte)                    │
├───────────────────────────────────────────────────────────────┤
│              served_domains (served_domains_count × 8 bytes)  │
├───────────────────────────────────────────────────────────────┤
│              Timestamp (8 bytes, uint64, Unix epoch)          │
├───────────────────────────────────────────────────────────────┤
│              Signature (64 bytes)                             │
│   Signs: NodeID || PublicKey || TransportsMask ||             │
│           RoutingTableSize || bridge_capable ||               │
│           served_domains || Timestamp                         │
└───────────────────────────────────────────────────────────────┘
```

### 8.3 Beacon Processing

Receiving a beacon:
1. Verify signature: `ed25519_verify(beacon.public_key, beacon.signature, beacon_content)`
2. Derive NodeID from public key: `BLAKE3(beacon.public_key)` — verify matches `beacon.node_id`
3. Add node to DHT routing table
4. Update transport address cache: `node_id → transport_addr`
5. If own routing table < 20 entries: send DHT_FIND_NODE to discovered node
6. If `bridge_capable = 0x01`: note bridge availability for served domains

The `bridge_capable` and `served_domains` fields allow mesh nodes to discover cross-domain bridge nodes without internet DHT access.

---

## 9. MESH_FRAGMENT Protocol

Fragmentation is required for LoRa, Bluetooth, and other low-MTU transports.

### 9.1 Wire Format

```
Common Header (Msg Type = 0x19, Session ID = 0x00...00)
├───────────────────────────────────────────────────────────────┤
│              Original Packet ID (4 bytes, random)             │
├───────────────────────────────────────────────────────────────┤
│              Fragment Number (1 byte, 0-indexed)              │
├───────────────────────────────────────────────────────────────┤
│              Total Fragments (1 byte, 1-16)                   │
├───────────────────────────────────────────────────────────────┤
│              Fragment Payload (variable)                      │
└───────────────────────────────────────────────────────────────┘
```

### 9.2 Fragmentation at Sender

```python
function fragment_and_send(bgpx_packet, transport):

    if len(bgpx_packet) <= transport.mtu:
        transport.send(peer, bgpx_packet)
        return

    # Fragment
    packet_id = CSPRNG.generate_bytes(4)  # 4-byte random per original packet
    max_fragment_payload = transport.mtu - 38  # MTU - MESH_FRAGMENT header (6B) - BGP-X header (32B)

    fragments = chunk(bgpx_packet, max_fragment_payload)
    total = len(fragments)

    if total > 16:
        # Packet too large even after fragmentation — drop
        log("Packet exceeds max fragmentable size")
        return

    for i, fragment in enumerate(fragments):
        frame = construct_mesh_fragment(
            original_packet_id = packet_id,
            fragment_num = i,
            total_fragments = total,
            payload = fragment
        )
        transport.send(peer, frame)
```

### 9.3 Reassembly at Receiver

```python
function reassemble(fragment):

    packet_id = fragment.original_packet_id

    if packet_id not in reassembly_buffer:
        reassembly_buffer[packet_id] = {
            "received": [],
            "total": fragment.total_fragments,
            "created": current_time()
        }

    reassembly_buffer[packet_id]["received"].append(
        (fragment.fragment_num, fragment.payload)
    )

    # Check timeout (10 seconds)
    if current_time() - reassembly_buffer[packet_id]["created"] > 10:
        del reassembly_buffer[packet_id]
        log("Fragment timeout", packet_id)
        return None

    # Check completion
    received = reassembly_buffer[packet_id]["received"]
    if len(received) == fragment.total_fragments:
        del reassembly_buffer[packet_id]
        # Sort by fragment_num and concatenate
        return b"".join(p for _, p in sorted(received))

    return None  # Not complete yet
```

### 9.4 Fragmentation Limits

| Transport | MTU | Max Fragment Payload | Max Original Packet |
|---|---|---|---|
| LoRa SF7 | ~200 bytes | ~162 bytes | ~2592 bytes (16 fragments) |
| LoRa SF8 | ~200 bytes | ~162 bytes | ~2592 bytes |
| LoRa SF9 | ~150 bytes | ~112 bytes | ~1792 bytes |
| LoRa SF10 | ~100 bytes | ~62 bytes | ~992 bytes |
| LoRa SF11 | ~80 bytes | ~42 bytes | ~672 bytes |
| LoRa SF12 | ~50 bytes | ~12 bytes | ~192 bytes |
| Bluetooth BLE | ~100 bytes | ~62 bytes | ~992 bytes |

For LoRa SF12, the maximum reassembled packet size (192 bytes) is below the BGP-X minimum useful size. LoRa SF12 is suitable only for MESH_BEACON and minimal DHT control messages, not for BGP-X onion-encrypted data packets.

---

## 10. Store-and-Forward for High-Latency Transports

For LoRa and satellite links with duty cycle constraints and high latency, BGP-X supports store-and-forward for latency-tolerant traffic:

### 10.1 Eligible Traffic for Store-and-Forward

- DHT_FIND_NODE / DHT_GET / DHT_PUT
- MESH_BEACON
- NODE_ADVERTISE
- DOMAIN_ADVERTISE
- MESH_ISLAND_ADVERTISE
- POOL_KEY_ROTATION

NOT eligible:
- RELAY (time-sensitive onion packets)
- HANDSHAKE messages (time-sensitive due to timestamp validation)
- KEEPALIVE (time-sensitive)

### 10.2 Store-and-Forward Configuration

```toml
[mesh.lora]
store_and_forward = true
buffer_size_kb = 1024      # Maximum buffer for queued packets
packet_ttl_seconds = 3600  # Discard buffered packets after 1 hour
```

### 10.3 Application-Layer Store-and-Forward

For application-layer store-and-forward (offline island): SDK `StreamConfig.store_and_forward = true` queues stream open requests and retries when island becomes reachable.

---

## 11. Meshtastic Hardware Adaptation Layer

Meshtastic devices (ESP32/nRF52840 + LoRa radio) are too resource-constrained to run the full BGP-X daemon. They can serve as LoRa radio modems via an adaptation layer.

### 11.1 Architecture

```
BGP-X Daemon (Raspberry Pi / GL.iNet router)
         │
         │ USB/Serial
         ▼
Meshtastic Device (ESP32 + SX1262)
         │
         │ LoRa radio
         ▼
Other mesh nodes
```

### 11.2 BGP-X Meshtastic Firmware

A BGP-X-specific firmware for ESP32/nRF52840 that:
- Removes Meshtastic's routing protocol
- Exposes raw LoRa send/receive via serial interface
- Accepts BGP-X MESH_FRAGMENT packets from host over serial
- Transmits LoRa frames, receives LoRa frames, forwards to host

Serial protocol:
```
→ [0x42 0x47] [length: 2 bytes BE] [payload: N bytes]  # Send frame
← [0x42 0x47] [length: 2 bytes BE] [payload: N bytes]  # Received frame
← [0x53 0x54] [status: 1 byte]                      # Status response
```

### 11.3 Compatible Meshtastic Hardware

| Device | MCU | LoRa Chip | BGP-X Adaptation |
|---|---|---|---|
| LILYGO T-Beam | ESP32 | SX1276 | Yes — radio modem via USB |
| LILYGO T-Beam Supreme | ESP32-S3 | SX1262 | Yes — radio modem via USB |
| LILYGO T-Echo | nRF52840 | SX1262 | Yes — radio modem via USB |
| LILYGO T3S3 | ESP32-S3 | SX1262 | Yes — radio modem via USB |
| Heltec LoRa32 v3 | ESP32-S3 | SX1262 | Yes — radio modem via USB |
| RAK WisBlock 4631 | nRF52840 | SX1262 | Yes — radio modem via USB |

---

## 12. Node Advertisement Extensions for Mesh

### 12.1 Legacy Format (mesh_endpoints)

Nodes with mesh transport capabilities include mesh_endpoints in their advertisement:

```json
{
  "mesh_endpoints": [
    {
      "transport": "wifi_mesh",
      "mesh_id": "bgpx-community-mesh-1",
      "mac_address": "aa:bb:cc:dd:ee:ff"
    },
    {
      "transport": "lora",
      "lora_addr": "0xAABBCCDD",
      "frequency_mhz": 868.0,
      "spreading_factor": 7,
      "bandwidth_khz": 125,
      "region": "EU"
    },
    {
      "transport": "ble",
      "ble_mac": "11:22:33:44:55:66"
    }
  ],
  "is_gateway": false,
  "gateway_for_mesh": null
}
```

Gateway nodes additionally:
```json
{
  "is_gateway": true,
  "gateway_for_mesh": ["bgpx-community-mesh-1", "bgpx-rural-lora"],
  "provides_clearnet_exit": true
}
```

### 12.2 Current Format (routing_domains)

The node advertisement now supports `routing_domains` and `bridges` fields. Legacy `endpoints` and `mesh_endpoints` remain valid for backward compatibility.

```json
{
  "routing_domains": [
    {
      "domain_type": "mesh",
      "domain_id": "0x00000003-a1b2c3d4",
      "island_id": "lima-district-1",
      "mesh_transport": [
        { "transport": "wifi_mesh", "mac": "aa:bb:cc:dd:ee:ff", "mesh_id": "bgpx-lima-1" },
        { "transport": "lora", "lora_addr": "0xAABBCCDD", "frequency_mhz": 868.0, "sf": 7 }
      ]
    }
  ],
  "bridge_capable": false,
  "bridges": []
}
```

### 12.3 Domain Bridge Node Advertisement Example

For a domain bridge node serving both clearnet and a mesh island:

```json
{
  "routing_domains": [
    {
      "domain_type": "clearnet",
      "domain_id": "0x00000001-00000000",
      "endpoints": [{ "protocol": "udp", "address": "203.0.113.1", "port": 7474 }]
    },
    {
      "domain_type": "mesh",
      "domain_id": "0x00000003-a1b2c3d4",
      "island_id": "lima-district-1",
      "mesh_transport": [
        { "transport": "wifi_mesh", "mac": "aa:bb:cc:dd:ee:ff", "mesh_id": "bgpx-lima-1" },
        { "transport": "lora", "lora_addr": "0xAABBCCDD", "frequency_mhz": 868.0, "sf": 7 }
      ]
    }
  ],
  "bridge_capable": true,
  "bridges": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "bridge_latency_ms": 25,
      "bridge_transport": "wifi_mesh",
      "available": true
    }
  ],
  "extensions": {
    "domain_bridge": true,
    "cross_domain_routing": true,
    "mesh_island_routing": true
  }
}
```

---

## 13. Geographic Plausibility for Mesh Nodes

Geographic plausibility scoring is an OPTIONAL reputation signal. For mesh nodes:

- If a mesh node declares a jurisdiction: geo plausibility scoring applies
- If a mesh node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply
- Mesh nodes have separate RTT thresholds appropriate to mesh transport (not internet RTT baselines)

Mesh-specific RTT thresholds are substantially higher than internet thresholds due to the inherent latency of radio transport and multi-hop mesh routing. See `/control-plane/geo_plausibility.md` for mesh-specific calibration.

---

## 14. Test Vectors

Required mesh transport test vectors for implementation verification:

### 14.1 MESH_BEACON

| Test ID | Description |
|---|---|
| `mesh_beacon_1` | Construct and verify MESH_BEACON with bridge_capable=false |
| `mesh_beacon_2` | Construct and verify MESH_BEACON with bridge_capable=true and served domains |
| `mesh_beacon_3` | MESH_BEACON with multiple transport types in bitmask |

### 14.2 MESH_FRAGMENT

| Test ID | Description |
|---|---|
| `mesh_fragment_1` | Fragmentation of 512-byte packet for LoRa SF7 MTU (~200 bytes) |
| `mesh_fragment_2` | Fragmentation of 1024-byte packet for BLE MTU (~100 bytes) |
| `mesh_fragment_3` | Reassembly of fragments received in order |
| `mesh_fragment_4` | Reassembly of fragments received out of order |
| `mesh_fragment_5` | Fragment timeout after partial receipt (10 seconds) |

### 14.3 Multi-Transport Selection

| Test ID | Description |
|---|---|---|
| `transport_select_1` | LOW_LATENCY requirement with WiFi + LoRa available → selects WiFi |
| `transport_select_2` | HIGH_LATENCY_OK with WiFi + LoRa available → selects LoRa (for long range) |
| `transport_select_3` | Packet size 1500 bytes with LoRa only → fragmentation triggered |

### 14.4 Cross-Domain

| Test ID | Description |
|---|---|
| `cross_domain_1` | DOMAIN_BRIDGE forwarding at bridge node from clearnet to mesh |
| `cross_domain_2` | Cross-domain path_id routing return traffic across domain boundary |
| `cross_domain_3` | Clearnet client path construction to mesh island service |

### 14.5 Meshtastic Serial Protocol

| Test ID | Description |
|---|---|
| `meshtastic_serial_1` | Send frame via serial protocol |
| `meshtastic_serial_2` | Receive frame via serial protocol |
| `meshtastic_serial_3` | Status response parsing |

### 14.6 Island Domain ID Derivation

| Test ID | Description |
|---|---|
| `island_domain_1` | Derive domain_id for island_id="lima-district-1" |
| `island_domain_2` | Derive domain_id for island_id="nyc-mesh-community" |
| `island_domain_3` | Derive domain_id for island_id="tokyo-shinjuku-west" |
| `island_domain_4` | Verify domain_id collision detection (different island_ids produce different domain_ids) |
| `island_domain_5` | Cross-domain path construction using island domain_id |
