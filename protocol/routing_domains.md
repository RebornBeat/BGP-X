# BGP-X Routing Domains Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

---

## 1. Overview

A **Routing Domain** is a named, addressable network segment with its own transport characteristics. BGP-X routes packets *between* routing domains — not just *through* one.

This document is the authoritative specification for the BGP-X inter-protocol routing model. Every path construction, every forwarding decision, and every node advertisement derives its domain semantics from this document.

### 1.1 Three Equal Entry Points

BGP-X treats three network classes as equal first-class citizens:

1. **Clearnet** — The BGP-routed public internet. Standard UDP/IP transport.
2. **BGP-X Overlay** — The BGP-X onion-encrypted layer spanning all physical transports.
3. **Mesh** — Named mesh island networks using radio transport (WiFi 802.11s, LoRa, BLE).

No domain is secondary. A client in any domain can reach a service in any other domain. A path may traverse any domain any number of times in any order.

### 1.2 N-Hop Unlimited Policy

BGP-X imposes **no global maximum on path hop count** at any level:

- No maximum on total hops across a path
- No maximum on pool segment count
- No maximum on domain traversal count
- No maximum on mesh relay hops within an island
- No maximum on domain bridge count within a path

The only hop constraints are:
- **Minimum**: 3 hops (absolute protocol minimum)
- **Defaults**: 4 hops for single-domain clearnet paths; 6–8 for cross-domain paths (configurable)
- **Application-defined limits**: applications may specify `max_total_hops` as a performance budget, not a privacy constraint

Practical upper bounds are determined by latency budgets and path construction feasibility, never by protocol enforcement.

---

## 2. Routing Domain Definitions

### 2.1 Domain Types

| Domain Type | Type ID | Description | Transport |
|---|---|---|---|
| `clearnet` | `0x00000001` | BGP-routed public internet | UDP/IP |
| `bgpx-overlay` | `0x00000002` | BGP-X onion layer (virtual; spans all physical) | Any |
| `mesh` | `0x00000003` | Named WiFi/LoRa/BLE mesh island | Radio |
| `lora-regional` | `0x00000004` | LoRa-only regional coverage zone | LoRa |
| `satellite` | `0x00000005` | Satellite constellation segment | Satellite |

### 2.2 Domain Identifier Wire Format

```
Domain ID: 8 bytes total

Bytes 0-3:  Type (uint32 big-endian) — domain type constant from table above
Bytes 4-7:  Instance (uint32 big-endian) — BLAKE3(instance_string) truncated to 4 bytes

Special cases:
  Clearnet:   instance = 0x00000000 (singleton; only one clearnet domain)
  Overlay:    instance = 0x00000000 (singleton; the BGP-X overlay is one domain)
  Mesh:       instance = BLAKE3(island_id_utf8_string)[0:4]
  LoRa:       instance = BLAKE3(region_id_utf8_string)[0:4]
  Satellite:  instance = BLAKE3(orbit_id_utf8_string)[0:4]

Examples:
  clearnet:                    0x00000001 00000000
  bgpx-overlay:                0x00000002 00000000
  mesh "lima-district-1":      0x00000003 <BLAKE3("lima-district-1")[0:4]>
  lora-regional "eu-south-1":  0x00000004 <BLAKE3("eu-south-1")[0:4]>
  satellite "starlink-leo":    0x00000005 <BLAKE3("starlink-leo")[0:4]>
```

### 2.3 Domain Instance Identifiers

**Mesh islands** use a string identifier chosen by the island operator:
- Format: `[a-z0-9][a-z0-9-]{2,62}` (DNS label-style)
- Examples: `lima-district-1`, `bogota-community`, `rural-kenya-north`
- Must be unique within the BGP-X network (enforced by DHT collision detection)
- Immutable after first DHT registration

**LoRa regional zones** use frequency + region identifiers:
- Format: `[region]-[freq-band]-[index]`
- Examples: `eu-868-1`, `us-915-3`
- Purpose: distinguishes non-overlapping LoRa coverage zones operating independently

**Satellite segments** use constellation + orbit identifiers:
- Examples: `starlink-leo`, `iridium-ird`, `inmarsat-geo`

---

## 3. Domain Bridge Nodes

### 3.1 Definition

A **Domain Bridge Node** is any BGP-X node that:
- Has active endpoints in two or more routing domains
- Advertises bridge capability in its node advertisement
- Actively forwards traffic between those domains

Domain bridge nodes are the routing fabric that makes inter-domain paths possible.

### 3.2 Bridge Types

| Bridge Type | From Domain | To Domain | Example |
|---|---|---|---|
| Internet Exit | clearnet | clearnet (different AS) | Standard relay/exit node |
| Mesh Gateway | clearnet | mesh:X | Community internet gateway |
| Mesh Bridge | mesh:X | mesh:Y | Island-to-island bridge |
| Satellite Gateway | clearnet | satellite:X | Satellite internet node |
| Satellite-Mesh | satellite:X | mesh:Y | Remote area access |
| Multi-Island | mesh:X | mesh:Y | Two-radio bridge node |

### 3.3 Domain Bridge Advertisement

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
      "bridge_transport": "wifi_mesh"
    }
  ]
}
```

### 3.4 Bridge Discovery

Domain bridge nodes are indexed in the unified DHT by their bridge pair:

```
DHT storage key = BLAKE3("bgpx-domain-bridge-v1" || from_domain_id_8B || to_domain_id_8B)

Note: from_domain and to_domain are ordered (lexicographically smaller first) so that
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

---

## 4. Mesh Island Records

### 4.1 Mesh Island Advertisement

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

### 4.2 Island DHT Key

```
island_key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8)

TTL: 24 hours
Re-publication: every 8 hours (more frequent than pool; topology changes quickly)
```

### 4.3 Island Self-Registration

When a mesh island has a bridge node with internet connectivity, that bridge node publishes the island advertisement to the unified DHT automatically. When no bridge node is available, the island operates in offline mode — island advertisement is only in the local mesh DHT.

---

## 5. New Onion Layer Hop Types

Cross-domain routing requires three new hop types beyond the existing 0x01–0x05:

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

### 5.1 DOMAIN_BRIDGE Hop (0x06) — Wire Format

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
4. Record `path_id → current_domain_endpoint` in the cross-domain path table
5. Select transport appropriate to target domain
6. Forward remaining onion payload to bridge node via target-domain transport

### 5.2 MESH_ENTRY Hop (0x07)

`next_hop` field:
```
Bytes 0-7:   Mesh island domain ID
Bytes 8-39:  First mesh relay node ID within island
```

Entering a mesh island: the bridge node forwards using the mesh transport (WiFi mesh MAC, LoRa addr) corresponding to the specified first relay node.

### 5.3 MESH_EXIT Hop (0x08)

`next_hop` field:
```
Bytes 0-7:   Destination domain ID (after exiting mesh)
Bytes 8-39:  Exit point node ID (bridge node with overlay or clearnet endpoint)
```

Exiting a mesh island: the final mesh relay forwards to the bridge node, which then uses clearnet or overlay transport for the next segment.

### 5.4 MESH_RELAY Hop (0x09)

Identical to RELAY (0x01) but signals to the recipient that this packet arrived via mesh transport and should be forwarded via mesh transport. Used when all hops within a mesh island should stay on radio transport.

---

## 6. Cross-Domain Onion Construction

### 6.1 Client-Side Construction

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

### 6.2 Bridge Node Onion Processing

When the bridge node decrypts its layer:
1. Finds `hop_type = 0x06` (DOMAIN_BRIDGE)
2. Extracts target domain ID and records path_id → clearnet predecessor
3. Records path_id → clearnet side in cross-domain path table
4. Selects mesh transport to reach next relay (using mesh transport address from DHT)
5. Forwards remaining onion payload via mesh transport

The remaining payload is ALREADY encrypted for the mesh relays — the bridge node does not need to re-encrypt. It simply delivers the inner layers via the appropriate transport.

### 6.3 Cross-Domain path_id Routing for Return Traffic

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

## 7. Domain Routing Policy

### 7.1 Any-to-Any Connectivity

BGP-X guarantees any-to-any connectivity across domains subject to:
- At least one domain bridge node existing between the required domain pair
- Sufficient active relay nodes in each domain segment
- Standard diversity constraints satisfied across all hops

If no bridge node exists between two specific domains, cross-domain connectivity is not possible for that pair. This is an operational gap, not a protocol limitation.

### 7.2 Domain Ordering Freedom

No domain ordering constraint exists at the protocol level. All of the following are valid path configurations:

```
clearnet → mesh → clearnet
mesh → clearnet → mesh
clearnet → mesh:A → mesh:B → clearnet
mesh:A → clearnet → mesh:B → clearnet → mesh:C
clearnet → satellite → mesh → clearnet
```

The client constructs any ordered sequence of domains. The protocol enforces no ordering.

### 7.3 Domain Isolation

Privacy isolation properties across domains:
- A node in domain A cannot see traffic in domain B that does not pass through it
- Domain bridge nodes see traffic entering and exiting the bridge, but not the full path in either domain
- path_id enables return routing across domains without revealing path composition
- The same adversary model applies cross-domain: entry/exit of the full path remain isolated

---

## 8. Unified DHT for All Domains

### 8.1 Single Logical DHT

BGP-X operates **one unified DHT**. All routing domains participate in the same key space.

Previous versions of this specification described two separate DHTs (internet and mesh) with gateway synchronization. This model is **replaced** by the unified DHT model:

- All nodes, regardless of routing domain, publish to and query the same DHT
- Domain bridge nodes (with clearnet endpoints) serve as the physical infrastructure for DHT storage accessible from the internet
- Mesh-only nodes access the unified DHT via their bridge nodes' DHT caches
- There is no "mesh DHT" — there is only the BGP-X DHT, accessible from all domains

### 8.2 Domain-Filtered DHT Queries

Clients can request domain-filtered results:

```
DHT_FIND_NODE request extension:
  domain_filter: optional DomainId
  If present: return only nodes with endpoints in this domain
  If absent: return all nodes regardless of domain
```

This allows clients to efficiently find relay nodes in a specific mesh island without downloading all DHT records.

### 8.3 Mesh-Only Node DHT Participation

Mesh-only nodes cannot directly reach internet DHT storage nodes. Their DHT participation:
1. Local mesh DHT: operates on mesh transport among mesh-island nodes
2. Bridge nodes sync local mesh records to internet DHT storage nodes
3. Mesh-only nodes query their bridge node's DHT cache for internet-hosted records
4. When mesh-only nodes have records to publish (their own advertisement), bridge nodes publish on their behalf

This is automatic — no configuration required. When a bridge node is present in the mesh island, DHT sync is continuous. When no bridge node is present, the island operates in local-mesh-only DHT mode.

### 8.4 Record Types in Unified DHT

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

## 9. Cross-Domain Session Establishment

### 9.1 Domain-Agnostic Handshake

The BGP-X handshake protocol is completely domain-agnostic. Whether a session is established over UDP, WiFi mesh, LoRa, or satellite:

- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE message format
- Same X25519 key exchange
- Same ChaCha20-Poly1305 for session key derivation
- Same HKDF-SHA256 key derivation with identical salts and info strings

The transport layer (UDP socket, WiFi mesh raw socket, LoRa serial port) is abstracted below the session layer. The handshake does not know or care which transport it runs on.

### 9.2 Cross-Domain Session Continuation

A BGP-X path crossing domain boundaries uses multiple sessions — one per hop as with single-domain paths. The session at a domain bridge node uses the transport appropriate to the domain where that node receives traffic.

The client's perspective:
- Handshake with clearnet relay 1 via UDP to its internet IP
- Handshake with clearnet relay 2 via UDP (relayed through relay 1's established session)
- Handshake with bridge node via UDP to its internet IP
- Handshake with mesh relay 1 via bridge node's established session (the bridge node forwards the handshake packet into the mesh using its radio)

The mesh relay never directly receives a UDP packet from the client. The bridge node delivers the handshake via mesh transport. From the mesh relay's perspective, a session is being established — it doesn't know the session request arrived via clearnet.

### 9.3 path_id in Cross-Domain Paths

path_id is generated once by the client and used throughout the entire cross-domain path. It is embedded in every onion layer including DOMAIN_BRIDGE layers.

At each domain transition:
- The bridge node records path_id in its cross-domain table for both directions
- Return traffic referencing that path_id is forwarded correctly across the domain boundary

path_id has the same privacy properties cross-domain as within a single domain:
- No relay in either domain can determine the full path from path_id alone
- path_id does not encode any information about the source, destination, or other hops

---

## 10. Domain Bridge Selection Algorithm

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

## 11. Path Failure Handling Across Domains

### 11.1 Bridge Node Failure

If a domain bridge node fails mid-path:
- path_id entries in both domains are orphaned (return traffic stops flowing)
- Client detects via KEEPALIVE timeout (90s default; shorter keepalive for mesh-bridge paths recommended)
- Client rebuilds path: select different bridge node for the failed domain transition
- In-flight data is retransmitted via ordered stream reliability layer

### 11.2 Segment-Level Rebuild Across Domains

BGP-X segment-level rebuild applies cross-domain:
- If only the mesh segment fails: rebuild only the mesh domain portion
- If only the clearnet segment fails: rebuild only the clearnet portion
- Bridge node failure triggers full rebuild of the affected bridge segment

### 11.3 Island Partitioning

If a mesh island loses all bridge nodes to other domains:
- Island continues to operate internally (intra-island paths unaffected)
- Clearnet-to-island traffic is impossible until a bridge node comes online
- Island-to-clearnet traffic is impossible
- Intra-island traffic: fully functional, no impact

This is expected behavior — mesh islands are designed to be operationally independent.

---

## 12. Privacy Properties of Cross-Domain Paths

### 12.1 Source-Destination Isolation

For a path: `clearnet-client → [relays] → bridge → [mesh relays] → mesh-service`

- clearnet relays: know client IP, do not know destination (mesh service address)
- bridge node: knows which clearnet relay forwarded to it (not client IP); knows which mesh relay received from it (not service address)
- mesh relays: know the bridge node (not client IP); know the next mesh relay or service (not origin)
- mesh service: knows the last mesh relay (not client IP)

The privacy model is identical to single-domain paths: split knowledge is maintained across domain boundaries.

### 12.2 Domain Bridge as a Hop

A domain bridge node is treated as a regular relay hop for privacy analysis purposes. It sees only its immediate predecessor in one domain and its immediate successor in the other. It does not see the full path in either domain.

### 12.3 Mesh Island Entry from Clearnet (No BGP-X Router Required)

A standard clearnet client with the BGP-X daemon (not a router — just a daemon on their device) can reach a mesh island service:

1. Client queries unified DHT for bridge nodes serving `clearnet ↔ mesh:target-island`
2. Client constructs path including a DOMAIN_BRIDGE hop to a bridge node
3. Bridge node delivers traffic into the mesh island via radio
4. Mesh service receives traffic with no knowledge of the client's clearnet IP
5. No BGP-X router, no mesh radio, no special hardware required on the client side

This is the core "any entry point can reach any domain" guarantee.

### 12.4 Timing Correlation Across Domains

A global passive adversary observing both clearnet traffic and mesh radio traffic may attempt timing correlation across domain boundaries. Mitigations:

- Cover traffic: same session_key as relay; applies within each domain independently
- KEEPALIVE randomization: applies in all domains (±5 seconds)
- Bridge node processing latency: adds variable delay at domain transitions
- Radio duty cycle variation (LoRa): naturally disrupts timing

Residual risk: acknowledged. Same fundamental limitation as single-domain timing correlation.

---

## 13. Compliance Requirements

### MUST

- Support DOMAIN_BRIDGE (0x06) hop type in forwarding pipeline
- Maintain cross-domain path_id routing table at bridge nodes
- Publish routing_domains and bridges in node advertisement when serving multiple domains
- Register bridge pairs in unified DHT when bridge capable
- Support domain-filtered DHT queries (domain_filter extension)
- Support N-hop unlimited construction (no protocol-level hop cap)
- Route return traffic across domain boundaries via cross-domain path_id table
- Propagate mesh island advertisements to unified DHT when bridge node is online
- Process MESH_ISLAND_ADVERTISE (0x1B) and DOMAIN_ADVERTISE (0x1A) message types

### MUST NOT

- Enforce a maximum path hop count at the protocol level
- Require clients to use a specific routing domain as entry
- Prevent paths from re-entering a previously traversed domain
- Log domain transition metadata (from/to domain) per-path or per-session

### SHOULD

- Support MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09) hop types
- Pre-discover domain bridge nodes for likely domain transitions (cache bridge records)
- Operate in offline (mesh-only) mode when no bridge to other domains is available
- Provide keepalive timeout extension for high-latency domain segments (LoRa, satellite)
