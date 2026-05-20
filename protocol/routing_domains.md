# BGP-X Routing Domains

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

This is the **authoritative specification** for BGP-X routing domain types and cross-domain routing. All other documents referring to routing domains defer to this file.

---

## 1. Overview

A **Routing Domain** is a named, addressable network segment with its own transport characteristics. BGP-X routes packets *between* routing domains — not just *through* one.

This document specifies:
- The definition and wire format of routing domain identifiers
- All defined domain types and their properties
- Domain bridge nodes and their operation
- Cross-domain path construction and routing
- The unified DHT model spanning all domains
- Privacy properties of cross-domain paths

---

## 2. Three Equal Entry Points

BGP-X treats three network classes as equal first-class citizens:

| Entry Point | Description | Hardware |
|---|---|---|
| **Clearnet** | BGP-routed public internet (fiber, cable, cellular, satellite internet) | Any internet-connected device |
| **BGP-X Overlay** | Onion-encrypted relay network | Any BGP-X node |
| **Mesh Islands** | Community radio networks (WiFi mesh, LoRa, BLE) | BGP-X Router v1, BGP-X Node v1, or Client Node |

No entry point is primary. A mesh-only user in a remote village and a fiber-connected user in a city participate in the same network. A clearnet client with no mesh hardware can reach services hosted in mesh islands via domain bridge nodes.

**Satellite internet (Starlink, Iridium, Inmarsat, etc.) is clearnet.** Commercial satellite services provide BGP-routed IP connectivity. From BGP-X's perspective, satellite internet is clearnet domain type `0x00000001` — the same as fiber or cellular, just with higher latency.

---

## 3. N-Hop Unlimited Policy

BGP-X imposes **no protocol-level maximum on path hop count** at any level:

- No maximum on total path hops
- No maximum on pool segment count
- No maximum on routing domain traversal count
- No maximum on mesh relay hops within an island
- No maximum on domain bridge transitions per path

The common header contains no hop counter field. No protocol mechanism decrements or enforces a hop maximum. Only the physical minimum (3 hops) and practical performance constraints apply.

**Minimum enforced**: 3 hops (absolute protocol minimum for meaningful anonymity).

**Default configurations**:
- Single-domain clearnet paths: 4 hops
- Cross-domain paths: 6–8 hops
- Application-specified: any count supported

Implementations MUST NOT enforce a maximum hop count at the protocol level. Path complexity is an application choice based on threat model and latency tolerance.

---

## 4. Routing Domain Definitions

### 4.1 Domain Types

| Domain Type | Type ID | Description | Transport |
|---|---|---|---|
| `clearnet` | `0x00000001` | BGP-routed public internet | UDP/IP |
| `bgpx-overlay` | `0x00000002` | BGP-X onion layer (virtual; spans all physical) | Any |
| `mesh` | `0x00000003` | Named WiFi/LoRa/BLE mesh island | Radio |
| `lora-regional` | `0x00000004` | LoRa-only regional coverage zone | LoRa |
| `satellite` | `0x00000005` | RESERVED — future BGP-X-native satellite network | Satellite |

### 4.2 Domain Identifier Wire Format

All routing domain references in BGP-X packets use an 8-byte **Domain ID**:

```
Domain ID: 8 bytes total

Bytes 0-3:  Type (uint32 big-endian) — domain type constant
Bytes 4-7:  Instance (uint32 big-endian) — BLAKE3(instance_string) truncated to 4 bytes

Special cases:
  Clearnet:       instance = 0x00000000 (singleton; only one clearnet)
  BGP-X Overlay:  instance = 0x00000000 (singleton; the overlay is one domain)
  Mesh:           instance = BLAKE3(island_id_utf8_string)[0:4]
  LoRa Regional:  instance = BLAKE3(region_id_utf8_string)[0:4]
  Satellite:      instance = BLAKE3(orbit_id_utf8_string)[0:4]
```

Wire diagram:

```
 0               1               2               3
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
├───────────────────────────────────────────────────────────────────┤
│                     domain_type (4 bytes BE)                      │
├───────────────────────────────────────────────────────────────────┤
│                   instance_hash (4 bytes BE)                      │
└───────────────────────────────────────────────────────────────────┘

Total: 8 bytes
```

### 4.3 Domain Instance Identifiers

**Mesh islands** use a string identifier chosen by the island operator:
- Format: `[a-z0-9][a-z0-9-]{2,62}` (DNS label-style)
- Examples: `lima-san-isidro-2026`, `bogota-community`, `rural-kenya-north`
- Must be unique within the BGP-X network (enforced by DHT collision detection)
- Immutable after first DHT registration

**LoRa regional zones** use frequency + region identifiers:
- Format: `[region]-[freq-band]-[index]`
- Examples: `eu-868-1`, `us-915-3`
- Purpose: distinguishes non-overlapping LoRa coverage zones operating independently

**Satellite segments** use constellation + orbit identifiers:
- Examples: `starlink-leo`, `iridium-ird`, `inmarsat-geo`

### 4.4 Domain ID Examples

```
clearnet:                    0x00000001 00000000
bgpx-overlay:                0x00000002 00000000
mesh "lima-district-1":      0x00000003 <BLAKE3("lima-district-1")[0:4]>
lora-regional "eu-south-1":  0x00000004 <BLAKE3("eu-south-1")[0:4]>
satellite "starlink-leo":    0x00000005 <BLAKE3("starlink-leo")[0:4]>
```

### 4.5 Invalid Domain IDs

- **All-zeros domain_id** (`0x0000000000000000`): invalid; MUST drop silently.
- **Unknown domain_type**: any value not in the table above; MUST drop silently.
- **Reserved domain_type 0x00000005**: MUST drop in current deployments (not yet active).

---

## 5. Domain Type Specifications

### 5.1 clearnet (Type 0x00000001)

**Domain ID**: `0x00000001-00000000`

**Definition**: The BGP-routed public internet. All traffic in this domain travels over standard UDP/IP using existing internet infrastructure routed by BGP. This is the default domain for all internet-connected BGP-X nodes.

**What clearnet IS**:
- Standard ISP fiber, cable, or cellular internet connections
- Satellite internet connections (Starlink, Iridium, Inmarsat, HughesNet, Viasat — all commercial satellite internet services)
- Any IP-routed network where BGP-X UDP packets are routable to destination IP

**What clearnet is NOT**:
- A specific physical medium (fiber is clearnet; satellite is clearnet; cellular is clearnet — all are clearnet)
- A BGP-X mesh domain
- A separate domain per ISP or per satellite provider

**Node advertisement for clearnet**:
```json
{
  "domain_type": "clearnet",
  "domain_id": "0x00000001-00000000",
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.45", "port": 7474 },
    { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
  ],
  "latency_class": "fiber"
}
```

**Latency classes for clearnet** (informational; affects congestion control thresholds and geo plausibility):

| Class | Typical RTT | Example |
|---|---|---|
| `fiber` | <10ms regional | Datacenter with fiber uplink |
| `broadband` | 10-50ms | Home cable/DSL |
| `cellular` | 20-80ms | 4G/5G mobile |
| `satellite-leo` | 20-60ms | Starlink, Iridium LEO |
| `satellite-meo` | 100-200ms | O3b mPOWER |
| `satellite-geo` | 550-700ms | Inmarsat, Viasat, HughesNet GEO |

---

### 5.2 bgpx-overlay (Type 0x00000002)

**Domain ID**: `0x00000002-00000000`

**Definition**: The BGP-X encrypted overlay itself. In multi-domain path construction, a segment may be designated as "overlay-only" — nodes selected only from the BGP-X relay pool, no exit to clearnet within this segment. This is used when the client wants explicit control over which path segment reaches an exit node vs. which remains within the overlay.

This domain type is implicit in all single-domain paths. In cross-domain path specifications, it may be used explicitly to indicate "BGP-X relay nodes only, no exit" for a segment.

**Node advertisement**: standard clearnet endpoint. The bgpx-overlay domain type is a path-construction concept, not a physical transport. All bgpx-overlay nodes have clearnet endpoints.

---

### 5.3 mesh (Type 0x00000003)

**Domain ID**: `0x00000003-BLAKE3(island_id_string)[0:4]`

**Definition**: A BGP-X mesh island — a community of BGP-X nodes communicating via radio transport (WiFi 802.11s, LoRa, Bluetooth BLE, or Ethernet P2P). Each mesh island is identified by its `island_id` string and has a unique Domain ID derived from it.

**What mesh IS**:
- WiFi 802.11s mesh networks (community WiFi)
- LoRa radio mesh networks (long-range low-bandwidth)
- Bluetooth BLE mesh networks (short-range)
- Ethernet point-to-point links between BGP-X nodes (treated as mesh when not routed via internet)
- Any collection of BGP-X nodes communicating via direct radio/physical links without traversing the internet

**What mesh is NOT**:
- Any satellite network (satellite internet is clearnet, not mesh)
- A VPN or overlay over clearnet (that is clearnet domain)

**Island ID format**: operator-chosen string; lowercase alphanumeric + hyphens; globally unique via DHT collision detection. Example: `lima-san-isidro-2026`.

**Domain ID derivation**:
```
island_id = "lima-san-isidro-2026"
hash = BLAKE3("lima-san-isidro-2026")   # 32-byte hash
instance = hash[0:4]                     # first 4 bytes as uint32
domain_id = 0x00000003 || instance       # 8 bytes total
```

**Node advertisement for mesh**:
```json
{
  "domain_type": "mesh",
  "domain_id": "0x00000003-a1b2c3d4",
  "island_id": "lima-san-isidro-2026",
  "mesh_transports": [
    { "type": "wifi-mesh", "ssid": "bgpx-mesh-lima", "channel": 6 },
    { "type": "lora", "frequency_mhz": 915.0, "spreading_factor": 9 }
  ]
}
```

**Mesh transports supported**:

| Transport | MTU | Bandwidth | Range | Use Case |
|---|---|---|---|---|
| `wifi-mesh` (802.11s) | 1280 bytes | 10-100+ Mbps | 100m-2km | Community WiFi |
| `lora` (raw) | ~200 bytes usable | 0.3-50 Kbps | 2-15+ km | Long-range |
| `ble` (BLE 5.0+) | 244 bytes | 1-2 Mbps | 100m | Short-range |
| `ethernet-p2p` | 1500 bytes | 100+ Mbps | Physical | Point-to-point |

LoRa and BLE transports require the **MESH_FRAGMENT** message (type 0x19) for packets larger than the transport MTU.

---

### 5.4 lora-regional (Type 0x00000004)

**Domain ID**: `0x00000004-BLAKE3(region_id)[0:4]`

**Definition**: A LoRa regional channel plan domain. Used when LoRa nodes in a geographic area share a common channel plan but are not organized into a single named island. This is a more loosely organized version of the mesh domain specifically for wide-area LoRa deployment.

**Region IDs**: `eu-868`, `na-915`, `asia-433`, `au-915`, `india-865`, etc.

**Domain ID derivation**: same as mesh domain; uses BLAKE3 of region_id string.

**When to use lora-regional vs. mesh**: Use `mesh` for organized community islands with known members and a gateway. Use `lora-regional` for open LoRa channel plans where any compatible node can participate without prior island membership.

---

### 5.5 bgpx-satellite (Type 0x00000005) — RESERVED

**Domain ID**: `0x00000005-BLAKE3(orbit_id)[0:4]`

**Status**: RESERVED for future BGP-X-native satellite network. **Not active in current deployments.**

**What this domain represents (future)**:
A constellation of BGP-X relay nodes physically in orbit, communicating via inter-satellite links carrying BGP-X protocol directly, with BGP-X-native ground terminals. This domain would provide relay nodes in orbit with no ground-based network infrastructure involvement in the satellite segment of a path.

---

## 6. Domain Bridge Nodes

### 6.1 Definition

A **Domain Bridge Node** is any BGP-X node that:
- Has active endpoints in two or more routing domains
- Advertises bridge capability in its node advertisement
- Actively forwards traffic between those domains

Domain bridge nodes are the routing fabric that makes inter-domain paths possible.

### 6.2 Bridge Types

| Bridge Type | From Domain | To Domain | Example |
|---|---|---|---|
| Internet Exit | clearnet | clearnet (different AS) | Standard relay/exit node |
| Mesh Gateway | clearnet | mesh:X | Community internet gateway |
| Mesh Bridge | mesh:X | mesh:Y | Island-to-island bridge |
| Satellite Gateway | clearnet | satellite:X | Satellite internet node |
| Satellite-Mesh | satellite:X | mesh:Y | Remote area access |
| Multi-Island | mesh:X | mesh:Y | Two-radio bridge node |

### 6.3 Domain Bridge Advertisement

Domain bridge capability is expressed in the node advertisement:

```json
{
  "routing_domains": [
    {
      "domain_type": "clearnet",
      "domain_id": "0x00000001-00000000",
      "endpoints": [
        { "protocol": "udp", "address": "203.0.113.1", "port": 7474 }
      ]
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
      "available": true,
      "geo_plausibility_exempt": false
    }
  ]
}
```

### 6.4 Bridge Discovery

Domain bridge nodes are indexed in the unified DHT by their bridge pair:

```
DHT storage key = BLAKE3("bgpx-domain-bridge-v1" || ordered(from_domain_id || to_domain_id))

Ordering: lexicographically smaller domain_id first. This ensures:
  bridge(A→B) and bridge(B→A) use the same key (bridges are bidirectional).
```

Clients discover bridge nodes by querying `DHT_GET(bridge_key)` where `bridge_key` matches their required domain transition.

Bridge discovery response format:
```json
{
  "bridge_key": "...",
  "from_domain": "0x00000001-00000000",
  "to_domain": "0x00000003-a1b2c3d4",
  "bridge_nodes": ["node_id_hex_1", "node_id_hex_2", "..."],
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "..."
}
```

### 6.5 Bridge Node Requirements

**Requirements for domain bridge operation**:
- BGP-X node with `bridge_capable = true` in node advertisement
- Active transport interfaces for each served domain
- `routing_domains` array in advertisement contains entries for each served domain
- `bridges` array lists each bridge pair with advertised latency and availability
- Extension flag bit 10 (`domain_bridge`) MUST be set
- Extension flag bit 11 (`cross_domain_routing`) MUST be set

**Examples of valid domain bridge configurations**:
- clearnet + mesh (most common: internet gateway for mesh island)
- mesh + lora-regional (bridge between WiFi mesh island and LoRa regional network)
- clearnet + lora-regional (clearnet gateway for LoRa nodes without WiFi mesh)
- clearnet + mesh + lora-regional (full triple-domain bridge node)
- satellite-clearnet + mesh (remote off-grid island with satellite WAN)

**Not a domain bridge** (because satellite internet is clearnet):
- Starlink WAN + LoRa mesh: this IS a domain bridge (clearnet via Starlink + LoRa mesh domain)
- Two Starlink terminals: both clearnet, communicating via clearnet domain — not a bridge

---

## 7. Mesh Island Records

### 7.1 Mesh Island Advertisement

A mesh island is itself an addressable entity in the BGP-X network. It advertises its existence, gateway nodes, and available services.

```json
{
  "version": 1,
  "island_id": "lima-district-1",
  "domain_id": "0x00000003-a1b2c3d4",
  "node_count": 12,
  "active_relay_count": 8,
  "bridge_nodes": [
    {
      "node_id": "hex...",
      "bridges": ["clearnet", "mesh:lima-district-1"],
      "clearnet_endpoint": "203.0.113.1:7474"
    }
  ],
    "services": [
        {
        "service_id": "hex...",
        "name": "community-forum",
        "advertise": true
        }
    ],
    "dht_coverage": "full",
    "offline_capable": true,
    "signed_at": "2026-04-24T12:00:00Z",
    "curator_public_key": "...",
    "signature": "..."
    }
```

### 7.2 Island DHT Key

```
island_key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8)

TTL: 24 hours
Re-publication: every 8 hours (more frequent than pool; topology changes quickly)
```

### 7.3 Island Self-Registration

When a mesh island has a bridge node with internet connectivity, that bridge node publishes the island advertisement to the unified DHT automatically. When no bridge node is available, the island operates in offline mode — island advertisement is only in the local mesh DHT.

---

## 8. Onion Layer Hop Types for Cross-Domain Routing

Cross-domain routing requires four new hop types beyond the existing 0x01–0x05:

| Type ID | Name | Description |
|---|---|---|
| `0x01` | `RELAY` | Forward within current domain |
| `0x02` | `EXIT_TCP` | Forward to clearnet IPv4 destination (TCP) |
| `0x03` | `EXIT_UDP` | Forward to clearnet IPv6 destination |
| `0x04` | `EXIT_RESOLVE` | Forward to clearnet domain name |
| `0x05` | `DELIVERY` | Deliver to BGP-X native service |
| `0x06` | `DOMAIN_BRIDGE` | Transition to another routing domain |
| `0x07` | `MESH_ENTRY` | Enter a mesh island from overlay |
| `0x08` | `MESH_EXIT` | Exit a mesh island to overlay |
| `0x09` | `MESH_RELAY` | Relay within mesh island |

### 8.1 DOMAIN_BRIDGE Hop (0x06) — Wire Format

The `next_hop` field (40 bytes) for DOMAIN_BRIDGE:

```
Bytes 0-7:   Target domain ID (8 bytes — domain type + instance)
Bytes 8-39:  Bridge node ID (32 bytes — NodeID of the bridge node)
```

Flags field for DOMAIN_BRIDGE:
```
Bit 0: ENTER_DOMAIN  — this packet is entering the target domain
Bit 1: EXIT_DOMAIN   — this packet is exiting the source domain
Bit 2: BIDIRECTIONAL — bridge maintains path_id in both domains
Bits 3-15: Reserved
```

When a relay node encounters `hop_type = 0x06`:
1. Parse target domain ID from bytes 0-7 of `next_hop`
2. Parse bridge node ID from bytes 8-39
3. Record `path_id → predecessor_addr` in current domain's path table
4. Record `path_id → {source_domain_endpoint, target_domain}` in cross-domain path table
5. Select transport appropriate to target domain
6. Forward remaining onion payload to bridge node via target-domain transport

### 8.2 MESH_ENTRY Hop (0x07)

`next_hop` field:
```
Bytes 0-7:   Mesh island domain ID
Bytes 8-39:  First mesh relay node ID within island
```

Entering a mesh island: the bridge node forwards using the mesh transport (WiFi mesh MAC, LoRa addr) corresponding to the specified first relay node.

### 8.3 MESH_EXIT Hop (0x08)

`next_hop` field:
```
Bytes 0-7:   Destination domain ID (after exiting mesh)
Bytes 8-39:  Exit point node ID (bridge node with overlay or clearnet endpoint)
```

Exiting a mesh island: the final mesh relay forwards to the bridge node, which then uses clearnet or overlay transport for the next segment.

### 8.4 MESH_RELAY Hop (0x09)

Identical to RELAY (0x01) but signals to the recipient that this packet arrived via mesh transport and should be forwarded via mesh transport. Used when all hops within a mesh island should stay on radio transport.

---

## 9. Cross-Domain Onion Construction

### 9.1 Client-Side Construction

For a path spanning multiple domains, the client constructs the onion identically to single-domain paths — one encryption layer per hop, including domain bridge hops. The DOMAIN_BRIDGE hop is just another layer.

Example 4+bridge+3 path (clearnet 4 hops → bridge → mesh 3 hops):

```
Client constructs 8 onion layers (4 clearnet relays + 1 bridge node + 3 mesh relays):

Layer 8 (outermost): encrypted for clearnet relay 1
  Layer 7: encrypted for clearnet relay 2
    Layer 6: encrypted for clearnet relay 3
      Layer 5: encrypted for clearnet relay 4
        Layer 4: encrypted for bridge node (hop_type = DOMAIN_BRIDGE)
          Layer 3: encrypted for mesh relay 1 (hop_type = MESH_ENTRY / MESH_RELAY)
            Layer 2: encrypted for mesh relay 2 (hop_type = MESH_RELAY)
              Layer 1: encrypted for mesh service (hop_type = DELIVERY)
```

All layers share the same `path_id`. The `path_id` is the return routing key across all domains.

### 9.2 Bridge Node Onion Processing

When the bridge node decrypts its layer:
1. Finds `hop_type = 0x06` (DOMAIN_BRIDGE)
2. Extracts target domain ID and records path_id → clearnet predecessor
3. Records path_id → clearnet side in cross-domain path table
4. Selects mesh transport to reach next relay (using mesh transport address from DHT)
5. Forwards remaining onion payload via mesh transport

The remaining payload is ALREADY encrypted for the mesh relays — the bridge node does not need to re-encrypt. It simply delivers the inner layers via the appropriate transport.

### 9.3 Cross-Domain path_id Routing for Return Traffic

Return traffic must traverse domain boundaries using path_id routing:

```
Mesh service generates response
         │ encrypts with K_service (session key shared with client)
         ▼
Mesh relay 2 → path_id lookup → mesh relay 1 (opaque forward)
         │
Mesh relay 1 → path_id lookup → bridge node (opaque forward, via mesh transport)
         │
Bridge node → cross-domain path table lookup → clearnet relay 4 (opaque forward, via UDP)
         │
Clearnet relay 4 → path_id lookup → relay 3 → relay 2 → relay 1 → client
         │
Client decrypts with K_service
```

**Cross-domain path table**: the bridge node maintains a second path table:
```
cross_domain_path_table[path_id] = {
    clearnet_predecessor: SocketAddr,     // Where to send return traffic in clearnet
    mesh_predecessor: MeshAddr,           // Where to send return traffic in mesh
}
```

When return traffic arrives from the mesh side with a known path_id: bridge node looks up `clearnet_predecessor` and forwards opaque blob via clearnet UDP.

When return traffic arrives from the clearnet side with a known path_id: bridge node looks up `mesh_predecessor` and forwards opaque blob via mesh transport.

---

## 10. Latency Class Handling

Different domains and clearnet variants have different expected latency characteristics. BGP-X uses latency class information for:
- Congestion control baseline calibration
- PATH_QUALITY_REPORT latency bucket boundaries
- Geographic plausibility scoring (satellite-class clearnet: exempt)
- Cover traffic minimum rate per domain type
- KEEPALIVE interval and timeout adjustment

### 10.1 Latency Classes

| Latency Class | Typical RTT | Example |
|---|---|---|
| `fiber` | <10ms regional | Datacenter with fiber uplink |
| `broadband` | 10-50ms | Home cable/DSL |
| `cellular` | 20-80ms | 4G/5G mobile |
| `satellite-leo` | 20-60ms | Starlink, Iridium LEO |
| `satellite-meo` | 100-200ms | O3b mPOWER |
| `satellite-geo` | 550-700ms | Inmarsat, Viasat, HughesNet GEO |
| `wifi-mesh` | 50 ms | WiFi 802.11s per-hop |
| `lora` | 5000 ms | LoRa inherently high latency |
| `ble` | 200 ms | Bluetooth BLE |

### 10.2 Congestion Detection Thresholds

MILD congestion triggers at: measured_rtt > 2.0 × baseline.
SEVERE congestion triggers at: measured_rtt > 4.0 × baseline.

Using domain-appropriate baselines prevents false SEVERE triggers on LoRa paths or satellite paths. A LoRa path with 3000ms RTT is NOT severe congestion — it is expected behavior.

---

## 11. Domain Routing Policy

### 11.1 Any-to-Any Connectivity

BGP-X guarantees any-to-any connectivity across domains subject to:
- At least one domain bridge node existing between the required domain pair
- Sufficient active relay nodes in each domain segment
- Standard diversity constraints satisfied across all hops

If no bridge node exists between two specific domains, cross-domain connectivity is not possible for that pair. This is an operational gap, not a protocol limitation.

### 11.2 Domain Ordering Freedom

No domain ordering constraint exists at the protocol level. All of the following are valid path configurations:

```
clearnet → mesh → clearnet
mesh → clearnet → mesh
clearnet → mesh:A → mesh:B → clearnet
mesh:A → clearnet → mesh:B → clearnet → mesh:C
clearnet → satellite → mesh → clearnet
```

The client constructs any ordered sequence of domains. The protocol enforces no ordering.

### 11.3 Domain Isolation

Privacy isolation properties across domains:
- A node in domain A cannot see traffic in domain B that does not pass through it
- Domain bridge nodes see traffic entering and exiting the bridge, but not the full path in either domain
- path_id enables return routing across domains without revealing path composition
- The same adversary model applies cross-domain: entry/exit of the full path remain isolated

---

## 12. Unified DHT for All Domains

### 12.1 Single Logical DHT

BGP-X operates **one unified DHT**. All routing domains participate in the same key space.

Previous versions of this specification described two separate DHTs (internet and mesh) with gateway synchronization. This model is **replaced** by the unified DHT model:

- All nodes, regardless of routing domain, publish to and query the same DHT
- Domain bridge nodes (with clearnet endpoints) serve as the physical DHT storage infrastructure accessible from the internet
- Mesh-only nodes access the unified DHT via their bridge nodes' DHT caches
- There is no "mesh DHT" — there is only the BGP-X DHT, accessible from all domains

### 12.2 Domain-Filtered DHT Queries

Clients can request domain-filtered results:

```
DHT_FIND_NODE request extension:
  domain_filter: optional DomainId
  If present: return only nodes with endpoints in this domain
  If absent: return all nodes regardless of domain
```

This allows clients to efficiently find relay nodes in a specific mesh island without downloading all DHT records.

### 12.3 Mesh-Only Node DHT Participation

Mesh-only nodes cannot directly reach internet DHT storage nodes. Their DHT participation:
1. Local mesh DHT: operates on mesh transport among mesh-island nodes
2. Bridge nodes sync local mesh records to internet DHT storage nodes
3. Mesh-only nodes query their bridge node's DHT cache for internet-hosted records
4. When mesh-only nodes have records to publish (their own advertisement), bridge nodes publish on their behalf

This is automatic — no configuration required. When a bridge node is present in the mesh island, DHT sync is continuous. When no bridge node is present, the island operates in local-mesh-only DHT mode.

### 12.4 Record Types in Unified DHT

```
Node advertisement:
  key = BLAKE3("bgpx-node-advert-v1" || node_id)

Pool advertisement:
  key = BLAKE3("bgpx-pool-v1" || pool_id)

Domain bridge advertisement:
  key = BLAKE3("bgpx-domain-bridge-v1" || ordered(from_domain_id || to_domain_id))

Mesh island advertisement:
  key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8)

Pool curator key rotation:
  key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || timestamp_BE8)
```

---

## 13. Cross-Domain Session Establishment

### 13.1 Domain-Agnostic Handshake

The BGP-X handshake protocol is completely domain-agnostic. Whether a session is established over UDP, WiFi mesh, LoRa, or satellite:

- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE message format
- Same X25519 key exchange
- Same ChaCha20-Poly1305 for session key derivation
- Same HKDF-SHA256 key derivation with identical salts and info strings

The transport layer (UDP socket, WiFi mesh raw socket, LoRa serial port) is abstracted below the session layer. The handshake does not know or care which transport it runs on.

### 13.2 Cross-Domain Session Continuation

A BGP-X path crossing domain boundaries uses multiple sessions — one per hop as with single-domain paths. The session at a domain bridge node uses the transport appropriate to the domain where that node receives traffic.

The client's perspective:
- Handshake with clearnet relay 1 via UDP to its internet IP
- Handshake with clearnet relay 2 via UDP (relayed through relay 1's established session)
- Handshake with bridge node via UDP to its internet IP
- Handshake with mesh relay 1 via bridge node's established session (the bridge node forwards the handshake packet into the mesh using its radio)

The mesh relay never directly receives a UDP packet from the client. The bridge node delivers the handshake via mesh transport. From the mesh relay's perspective, a session is being established — it doesn't know the session request arrived via clearnet.

### 13.3 path_id in Cross-Domain Paths

path_id is generated once by the client and used throughout the entire cross-domain path. It is embedded in every onion layer including DOMAIN_BRIDGE layers.

At each domain transition:
- The bridge node records path_id in its cross-domain table for both directions
- Return traffic referencing that path_id is forwarded correctly across the domain boundary

path_id has the same privacy properties cross-domain as within a single domain:
- No relay in either domain can determine the full path from path_id alone
- path_id does not encode any information about the source, destination, or other hops

---

## 14. Domain Bridge Selection Algorithm

When constructing a cross-domain path, bridge nodes are selected using a specialized scoring function:

```python
def score_bridge(node, from_domain, to_domain):
    # Uptime in both domains (worst of the two)
    uptime_score = min(
        node.uptime_in_domain(from_domain),
        node.uptime_in_domain(to_domain)
    ) / 100.0

    # Cross-domain bridge latency
    bridge_latency = node.bridge_latency_ms(from_domain, to_domain)
    latency_score = max(0, 1.0 - (bridge_latency / 500.0))

    # Overall reputation (global)
    rep_score = node.reputation_score / 100.0

    # Number of domains served (more domains = more routing flexibility = slight bonus)
    domain_diversity_score = min(1.0, len(node.routing_domains) / 5.0)

    # Geographic spread of domains served
    geo_spread_score = compute_geo_spread(node.routing_domains)

    return (
        0.30 * uptime_score +
        0.25 * latency_score +
        0.20 * rep_score +
        0.15 * domain_diversity_score +
        0.10 * geo_spread_score
    )
```

Diversity constraints for bridge node selection:
- No two bridge nodes with the same operator_id in one path
- No two bridge nodes in the same ASN across the full path
- At least 2 independent bridge nodes available per required domain transition (for redundancy)

---

## 15. Path Failure Handling Across Domains

### 15.1 Bridge Node Failure

If a domain bridge node fails mid-path:
- path_id entries in both domains are orphaned (return traffic stops flowing)
- Client detects via KEEPALIVE timeout (90s default; shorter keepalive for mesh-bridge paths recommended)
- Client rebuilds path: select different bridge node for the failed domain transition
- In-flight data is retransmitted via ordered stream reliability layer

### 15.2 Segment-Level Rebuild Across Domains

BGP-X segment-level rebuild applies cross-domain:
- If only the mesh segment fails: rebuild only the mesh domain portion
- If only the clearnet segment fails: rebuild only the clearnet portion
- Bridge node failure triggers full rebuild of the affected bridge segment

### 15.3 Island Partitioning

If a mesh island loses all bridge nodes to other domains:
- Island continues to operate internally (intra-island paths unaffected)
- Clearnet-to-island traffic is impossible until a bridge node comes online
- Island-to-clearnet traffic is impossible
- Intra-island traffic: fully functional, no impact

This is expected behavior — mesh islands are designed to be operationally independent.

---

## 16. Privacy Properties of Cross-Domain Paths

### 16.1 Source-Destination Isolation

For a path: `clearnet-client → [relays] → bridge → [mesh relays] → mesh-service`

- clearnet relays: know client IP, do not know destination (mesh service address)
- bridge node: knows which clearnet relay forwarded to it (not client IP); knows which mesh relay received from it (not service address)
- mesh relays: know the bridge node (not client IP); know the next mesh relay or service (not origin)
- mesh service: knows the last mesh relay (not client IP)

The privacy model is identical to single-domain paths: split knowledge is maintained across domain boundaries.

### 16.2 Domain Bridge as a Hop

A domain bridge node is treated as a regular relay hop for privacy analysis purposes. It sees only its immediate predecessor in one domain and its immediate successor in the other. It does not see the full path in either domain.

### 16.3 Mesh Island Entry from Clearnet (No BGP-X Router Required)

A standard clearnet client with the BGP-X daemon (not a router — just a daemon on their device) can reach a mesh island service:

1. Client queries unified DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
2. Client constructs path including a DOMAIN_BRIDGE hop to a bridge node
3. Bridge node delivers traffic into the mesh island via radio
4. Mesh service receives traffic with no knowledge of the client's clearnet IP
5. No BGP-X router, no mesh radio, no special hardware required on the client side

This is the core "any entry point can reach any domain" guarantee.

### 16.4 Timing Correlation Across Domains

A global passive adversary observing both clearnet traffic and mesh radio traffic may attempt timing correlation across domain boundaries. Mitigations:

- Cover traffic: same session_key as relay; applies within each domain independently
- KEEPALIVE randomization: applies in all domains (±5 seconds)
- Bridge node processing latency: adds variable delay at domain transitions
- Radio duty cycle variation (LoRa): naturally disrupts timing

Residual risk: acknowledged. Same fundamental limitation as single-domain timing correlation.

---

## 17. Compliance Requirements

### MUST

- Implement DOMAIN_BRIDGE (0x06) hop type in forwarding pipeline
- Maintain cross-domain path_id routing table at bridge nodes
- Publish routing_domains and bridges in node advertisement when serving multiple domains
- Register bridge pairs in unified DHT when bridge capable
- Support domain-filtered DHT queries (domain_filter extension)
- Support N-hop unlimited construction (no protocol-level hop cap)
- Route return traffic across domain boundaries via cross-domain path_id table
- Propagate mesh island advertisements to unified DHT when bridge node is online
- Process MESH_ISLAND_ADVERTISE (0x1B) and DOMAIN_ADVERTISE (0x1A) message types
- Validate domain_id format (type 0x01–0x05 only; reject 0x05 until satellite domain activated)
- Reject domain_id = 0x0000000000000000 (all zeros)
- Reject unknown domain_type values

### MUST NOT

- Enforce a maximum path hop count at the protocol level
- Require clients to use a specific routing domain as entry
- Prevent paths from re-entering a previously traversed domain
- Log domain transition metadata (from/to domain) per-path or per-session
- Maintain separate per-routing-domain DHTs
- Accept DOMAIN_ADVERTISE for domain type 0x00000005 (reserved) in current deployments

### SHOULD

- Support MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09) hop types
- Pre-discover domain bridge nodes for likely domain transitions (cache bridge records)
- Operate in offline (mesh-only) mode when no bridge to other domains is available
- Provide keepalive timeout extension for high-latency domain segments (LoRa, satellite)
- Apply latency-class-appropriate congestion thresholds
- Exempt satellite-class clearnet nodes from geo plausibility scoring
- Verify domain bridge advertisements against node advertisement routing_domains

---

## 18. Test Vectors

Test vectors for routing domain specification verification:

| Test Category | Count | Description |
|---|---|---|
| Domain ID derivation | 6 | All domain types; verify correct BLAKE3 instance hash |
| DOMAIN_BRIDGE hop encoding | 5 | Valid bridge hops with different domain pairs |
| MESH_ENTRY hop encoding | 3 | Valid mesh island entries |
| MESH_EXIT hop encoding | 3 | Valid mesh island exits |
| Cross-domain path construction | 5 | Multi-domain paths with 2-4 domain transitions |
| Cross-domain return routing | 3 | path_id lookup across domain boundary |
| Bridge node discovery | 3 | DHT query for specific bridge pair |
| Mesh island advertisement | 3 | Valid island records |
| Domain ID rejection | 4 | Zero domain_id, unknown type, reserved type, malformed encoding |
| Latency class handling | 5 | Congestion thresholds per domain type |

Test vectors will be documented in `/protocol/test_vectors/routing_domains/` during implementation.

---

## 19. Glossary

| Term | Definition |
|---|---|
| Routing Domain | A named, addressable network segment with its own transport characteristics |
| Domain ID | 8-byte identifier: type (4B) + instance hash (4B) |
| Domain Bridge Node | Node with endpoints in 2+ domains; forwards traffic between them |
| Mesh Island | Named community of BGP-X nodes communicating via radio transport |
| Island ID | Operator-chosen unique string identifier for a mesh island |
| Bridge Pair | Two routing domains connected by at least one bridge node |
| Unified DHT | Single Kademlia DHT spanning all routing domains |
| Cross-Domain Path | Path traversing multiple routing domains |
| DOMAIN_BRIDGE Hop | Onion layer type 0x06; signals domain transition |
| path_id | 8-byte random identifier enabling return routing across domain boundaries |
| Latency Class | Categorization of expected RTT for congestion and scoring purposes |
| Clearnet | BGP-routed public internet — includes satellite internet |
| Satellite Internet | Commercial services providing BGP-routed IP over satellite (clearnet domain) |
| bgpx-satellite | Reserved domain type 0x00000005 for future BGP-X-native satellite network |
