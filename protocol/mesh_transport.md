# BGP-X Mesh Transport Specification

**Version**: 0.1.0-draft

---

## 1. Mesh as a First-Class Routing Domain

BGP-X mesh transport is a first-class routing domain — named, addressable, discoverable from all other domains, and capable of hosting BGP-X services accessible from anywhere in the BGP-X network.

A mesh island is:
- Identified by a unique `island_id` string (operator-chosen, globally unique via DHT collision detection)
- Advertised in the unified BGP-X DHT when a bridge node is online
- Reachable from clearnet clients without any special client hardware
- Capable of hosting BGP-X native services accessible from any domain
- An independent operating unit when bridge nodes are offline

---

## 2. Transport Abstraction

```rust
pub trait MeshTransport: Send + Sync {
    async fn send(&self, peer_node_id: &NodeId, data: &[u8]) -> Result<(), TransportError>;
    async fn recv(&self) -> Result<(NodeId, Vec<u8>), TransportError>;
    async fn broadcast(&self, data: &[u8]) -> Result<usize, TransportError>;
    fn local_addr(&self) -> MeshAddr;
    fn capabilities(&self) -> TransportCapabilities;
    fn mtu(&self) -> usize;
    fn transport_type(&self) -> TransportType;
    fn domain_id(&self) -> Option<DomainId>;  // Returns mesh island domain ID
}

pub struct TransportCapabilities {
    pub mtu_bytes: usize,
    pub bandwidth_kbps: f32,
    pub latency_ms_typical: f32,
    pub broadcast_support: bool,
    pub max_range_meters: Option<f32>,
    pub requires_fragmentation: bool,
    pub latency_class: LatencyClass,
}

pub enum LatencyClass {
    LowLatency,    // <20ms typical (WiFi, Ethernet)
    MediumLatency, // 20-200ms
    HighLatency,   // 200ms-5s (LoRa, satellite)
}
```

---

## 3. Supported Transports

### WiFi 802.11s Mesh

```
Protocol:  IEEE 802.11s mesh networking
MTU:       1280 bytes (same as UDP/IP target)
Bandwidth: 10-100+ Mbps per hop
Latency:   1-20ms per hop
Range:     50-200m indoor, km with directional antennas
Fragmentation: Not required
```

BGP-X binding: Linux mac80211 mesh mode via iw/hostapd. BGP-X daemon creates raw socket on mesh interface.

### LoRa Radio

```
Protocol:  Raw LoRa (not LoRaWAN)
MTU:       SF7: ~200 bytes usable; SF12: ~50 bytes usable
Bandwidth: 0.3-50 Kbps (SF-dependent)
Latency:   100ms-5s per hop (duty cycle constraints)
Range:     5-50km rural open terrain; 1-5km urban
Fragmentation: REQUIRED (MESH_FRAGMENT 0x19)
Regulatory: Sub-GHz ISM bands; 1% duty cycle limit (EU)
```

BGP-X binding: SPI or USB connection to LoRa module (SX1262, SX1276) via Linux driver or userspace library.

**EU duty cycle compliance**: 1% → ~36 seconds transmit time per hour at SF7. BGP-X LoRa mode prioritizes DHT sync and key exchange. Token bucket controls transmission rate.

### Bluetooth BLE

```
Protocol:  Bluetooth mesh profile or custom BLE
MTU:       100-200 bytes
Bandwidth: 1-2 Mbps
Latency:   10-100ms
Range:     10-100m per hop
Fragmentation: REQUIRED
```

### Ethernet Point-to-Point

```
Protocol:  Standard Ethernet / fiber
MTU:       1500 bytes (BGP-X still targets 1280)
Bandwidth: 100 Mbps to 10 Gbps
Latency:   <1ms copper, <5ms moderate fiber
Fragmentation: Not required
```

---

## 4. Mesh Island as Routing Domain

### Island Identity

Each mesh island has:
- An `island_id` string (operator-chosen, globally unique)
- A `domain_id` (type `0x00000003` + BLAKE3(island_id)[0:4])
- A set of relay nodes discoverable via unified DHT with domain filter
- Zero or more domain bridge nodes connecting to other domains
- An optional service registry published in DHT records

### Island DHT Participation

When a bridge node is online (internet-connected):
- Bridge node publishes island advertisement to unified internet DHT (MESH_ISLAND_ADVERTISE)
- Bridge node stores mesh-domain node advertisements in internet DHT on behalf of mesh-only nodes
- Internet clients can discover mesh island relay nodes via domain-filtered DHT queries
- Mesh-only nodes' advertisements are served from internet DHT by the bridge node

When no bridge node is online (offline mode):
- Island operates with local mesh DHT (bootstrap via MESH_BEACON broadcast)
- Internet clients cannot reach island
- Intra-island traffic fully functional

### Clearnet Client Accessing Mesh Island Service

This is the key capability. A clearnet client with no mesh hardware:

1. Queries unified DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
2. Constructs path: `[clearnet relays] → [B: domain bridge] → [mesh relays] → [service]`
3. Bridge node B receives clearnet UDP, decrypts DOMAIN_BRIDGE layer, forwards via radio to first mesh relay
4. Service receives traffic with no knowledge of the client's clearnet IP
5. Return path via cross-domain path_id routing

**No BGP-X router required on the client.** The BGP-X daemon on the client's device handles path construction. The bridge node handles all radio transmission.

---

## 5. Multi-Transport Operation

### Transport Selection Algorithm

```python
function select_transport(peer_node_id, packet_size, latency_class_required, target_domain):

    # If routing to another domain, prefer that domain's transport
    if target_domain and target_domain != current_domain:
        bridge_transport = get_bridge_transport_to(target_domain)
        if bridge_transport:
            return bridge_transport

    # Within-domain transport selection
    available = transports_for_peer(peer_node_id)

    if latency_class_required == LOW_LATENCY:
        preferred = [t for t in available if t.latency_class == LOW_LATENCY]
    else:
        preferred = available

    if not preferred:
        preferred = available

    return max(preferred, key=lambda t: t.capabilities.bandwidth_kbps)
```

### Inter-Island Routing

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

## 6. Unified DHT for Mesh (Replaces Two-DHT Model)

BGP-X operates ONE unified DHT. The previous two-DHT model (internet DHT + mesh DHT with gateway sync) is replaced:

- All nodes participate in the same Kademlia key space
- Bridge nodes (including gateway nodes) store and serve records for mesh-only nodes in the unified internet DHT
- Mesh-only nodes access the unified DHT via their bridge nodes' caches
- No synchronization between separate DHTs needed — there is only one DHT

When bridge node is online: any pending mesh island record updates published to unified DHT automatically.
When bridge node is offline: records remain valid until TTL; new records cannot be published until bridge reconnects.

---

## 7. MESH_BEACON Protocol

Beacons enable DHT bootstrap in mesh-only networks. Broadcast every 30 seconds ± 5 seconds jitter.

Wire format: see `/protocol/packet_format.md` Section 17. Extended to include bridge capability flag and served domain IDs.

Receiving nodes MUST verify signature before adding sender to DHT routing table and before updating transport address cache.

---

## 8. MESH_FRAGMENT Protocol

Fragmentation for low-MTU transports. Domain-agnostic.

```
function fragment_and_send(bgpx_packet, transport):
    if len(bgpx_packet) <= transport.mtu:
        transport.send(peer, bgpx_packet)
        return

    packet_id = CSPRNG.generate_bytes(4)
    max_fragment_payload = transport.mtu - 38  # 6 MESH_FRAGMENT overhead + 32 BGP-X common header
    fragments = chunk(bgpx_packet, max_fragment_payload)
    total = len(fragments)

    if total > 16:
        log("Packet too large to fragment")
        return

    for i, fragment in enumerate(fragments):
        frame = construct_mesh_fragment(packet_id, i, total, fragment)
        transport.send(peer, frame)
```

Reassembly timeout: 10 seconds. After timeout: discard fragment buffer, log internally.

---

## 9. Store-and-Forward for High-Latency Transports

Store-and-forward for LoRa/satellite duty cycle constraints and intermittent connectivity.

Eligible for store-and-forward (time-tolerant):
- DHT_FIND_NODE, DHT_GET, DHT_PUT
- MESH_BEACON
- NODE_ADVERTISE
- DOMAIN_ADVERTISE
- MESH_ISLAND_ADVERTISE
- POOL_KEY_ROTATION

NOT eligible (time-sensitive):
- RELAY (onion packets)
- HANDSHAKE messages (timestamp validation)
- KEEPALIVE

For application-layer store-and-forward (offline island): SDK `StreamConfig.store_and_forward = true` queues stream open requests and retries when island becomes reachable.

---

## 10. Meshtastic Hardware Adaptation Layer

Meshtastic devices (ESP32/nRF52840 + LoRa radio) serve as LoRa radio modems for BGP-X nodes via an adaptation layer.

Architecture:
```
BGP-X Daemon (Raspberry Pi / GL.iNet router)
         │ USB/Serial
         ▼
Meshtastic Device (ESP32 + SX1262)
         │ LoRa radio
         ▼
Other mesh nodes
```

BGP-X Meshtastic firmware removes the Meshtastic routing protocol and exposes raw LoRa send/receive via serial interface.

Serial protocol:
```
Host → Device:   [0x42 0x47] [length: 2 bytes BE] [payload: N bytes]
Device → Host:   [0x42 0x47] [length: 2 bytes BE] [payload: N bytes]
Status response: [0x53 0x54] [status: 1 byte]
```

Compatible Meshtastic hardware: LILYGO T-Beam Supreme, T-Echo, Heltec LoRa32 v3, RAK WisBlock 4631.

---

## 11. Node Advertisement Extensions for Mesh

Node advertisements use `routing_domains` instead of legacy `mesh_endpoints`:

```json
{
  "routing_domains": [
    {
      "domain_type": "mesh",
      "domain_id": "0x00000003-a1b2c3d4",
      "island_id": "lima-district-1",
      "mesh_transport": [
        { "transport": "wifi_mesh", "mac": "aa:bb:cc:dd:ee:ff", "mesh_id": "bgpx-lima-1" },
        { "transport": "lora", "lora_addr": "0xAABBCCDD", "frequency_mhz": 868.0, "sf": 7, "region": "EU" }
      ]
    }
  ],
  "bridge_capable": false,
  "bridges": []
}
```

For bridge-capable nodes, `routing_domains` contains both clearnet and mesh entries, and `bridges` lists active bridge pairs.

---

## 12. Test Vectors

Required mesh transport test vectors:

1. MESH_BEACON construction and verification (with domain fields and bridge flag): 3 vectors
2. MESH_BEACON with bridge_capable=1 and served domains: 2 vectors
3. MESH_FRAGMENT fragmentation: 5 vectors (various packet sizes, transport MTUs)
4. MESH_FRAGMENT reassembly including out-of-order: 5 vectors
5. Fragment timeout: 2 vectors
6. Multi-transport selection including cross-domain: 5 vectors
7. Meshtastic serial protocol: 3 vectors
8. Island domain ID derivation (BLAKE3(island_id)[0:4] + type): 5 vectors
9. Cross-domain DOMAIN_BRIDGE forwarding at bridge node: 3 vectors
10. Cross-domain path_id routing return traffic across domain boundary: 3 vectors
