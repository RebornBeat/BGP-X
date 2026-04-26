# BGP-X Protocol Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

---

## 1. Introduction

This document is the authoritative specification for the BGP-X wire protocol. It defines:

- The message types exchanged between BGP-X nodes
- The structure and encoding of all protocol messages
- The state machines governing node behavior
- The rules for session establishment, maintenance, and teardown
- The behavior required of compliant implementations

All BGP-X implementations must conform to this specification.

---

## 2. Terminology

| Term | Definition |
|---|---|
| Node | Any participant in the BGP-X overlay network |
| Client | An initiating party that constructs and sends overlay packets |
| Entry node | The first relay hop; knows client transport address, not destination |
| Relay node | A middle hop; knows only previous and next hop |
| Exit node / Gateway | The final hop for clearnet traffic; knows destination, not client |
| Domain bridge node | A node with endpoints in 2+ routing domains |
| Routing domain | A named, addressable network segment with its own transport characteristics |
| Mesh island | A named collection of BGP-X nodes communicating via radio transport |
| Path | An ordered sequence of nodes spanning one or more routing domains |
| Segment | A portion of a path within one routing domain or pool |
| Stream | A logical bidirectional channel multiplexed over a path |
| Session | The cryptographic context for communication between client and one node |
| Onion packet | A layered-encrypted packet progressively decrypted at each hop |
| NodeID | BLAKE3(Ed25519_public_key) — 32 bytes |
| ServiceID | Ed25519 public key of a BGP-X native service — 32 bytes |
| path_id | 8-byte CSPRNG-generated identifier per path; enables return traffic routing across domain boundaries |
| Pool | A named, signed collection of nodes grouped by trust or operational intent |
| Pool ID | BLAKE3 hash identifying a pool, 32 bytes |
| Curator | Entity that signs pool advertisements and membership records |
| Domain ID | 8-byte identifier: type (4 bytes BE) + instance hash (4 bytes BE) |
| Bridge pair | Two routing domains connected by a domain bridge node |
| DHT | Unified Distributed Hash Table for all routing domains |
| PT | Pluggable Transport — traffic obfuscation layer below BGP-X |

---

## 3. Protocol Overview

### 3.1 Underlay (Transport)

BGP-X runs over UDP/IP for internet deployments. For mesh deployments: WiFi 802.11s, LoRa, Bluetooth BLE, or Ethernet point-to-point.

The protocol is transport-agnostic. All BGP-X messages have identical format regardless of transport.

Default BGP-X port (UDP): **7474/UDP**

Implementations MUST support IPv4. SHOULD support IPv6. Dual-stack RECOMMENDED.

For mesh transports, addressing uses NodeID rather than IP addresses. The mesh transport layer maps NodeIDs to transport-specific addresses (MAC, LoRa device address, etc.).

### 3.2 N-Hop Unlimited Policy

BGP-X imposes **no protocol-level maximum on path hop count** at any level:

- No maximum on total path hops
- No maximum on pool segment count
- No maximum on routing domain traversal count
- No maximum on mesh relay hops within an island
- No maximum on domain bridge transitions per path

The common header contains no hop counter field. No protocol mechanism decrements or enforces a hop maximum. Only the physical minimum (3 hops) and practical performance constraints apply.

Implementations MUST NOT enforce a maximum hop count at the protocol level.

### 3.3 Routing Domain Model

BGP-X treats three network classes as equal first-class citizens:

1. **Clearnet** — BGP-routed public internet
2. **BGP-X Overlay** — the onion-encrypted layer spanning all transports
3. **Mesh islands** — named radio-transport communities

Packets may traverse any combination of routing domains in any order, any number of times. Domain bridge nodes enable cross-domain routing. See `/protocol/routing_domains.md` for the authoritative domain specification.

A clearnet client with a BGP-X daemon and no special hardware can reach a mesh island service. A mesh island client with no ISP can reach clearnet via a gateway. The protocol enforces no entry point restriction and no domain ordering.

### 3.4 Unified DHT

BGP-X operates one unified Kademlia-style DHT spanning all routing domains. All nodes participate in the same key space regardless of transport or domain. There is no separate mesh DHT. Domain bridge nodes serve as the physical DHT storage infrastructure accessible from the internet for mesh-only nodes.

---

## 4. Message Types

| Type ID | Name | Direction | Description |
|---|---|---|---|
| 0x01 | RELAY | Any → Any | Forward an onion-encrypted payload within current domain |
| 0x02 | HANDSHAKE_INIT | Client → Node | Initiate session key exchange |
| 0x03 | HANDSHAKE_RESP | Node → Client | Respond to key exchange |
| 0x04 | HANDSHAKE_DONE | Client → Node | Complete key exchange |
| 0x05 | DATA | Within session | Encrypted application data |
| 0x06 | STREAM_OPEN | Within session | Open a new multiplexed stream |
| 0x07 | STREAM_CLOSE | Within session | Close a multiplexed stream |
| 0x08 | STREAM_DATA | Within session | Data on a specific stream |
| 0x09 | KEEPALIVE | Any → Any | Maintain session liveness |
| 0x0A | NODE_ADVERTISE | Node → DHT | Publish node advertisement |
| 0x0B | DHT_FIND_NODE | Any → DHT | Locate nodes near a target ID |
| 0x0C | DHT_FIND_NODE_RESP | DHT → Any | Response to DHT_FIND_NODE |
| 0x0D | DHT_GET | Any → DHT | Retrieve a DHT record |
| 0x0E | DHT_GET_RESP | DHT → Any | Response to DHT_GET |
| 0x0F | DHT_PUT | Any → DHT | Store a DHT record |
| 0x10 | ERROR | Any → Any | Signal a protocol error |
| 0x11 | COVER | Any → Any | Cover traffic (externally indistinguishable from RELAY) |
| 0x12 | DHT_POOL_ADVERTISE | Curator → DHT | Publish pool advertisement |
| 0x13 | DHT_POOL_GET | Any → DHT | Retrieve a pool advertisement |
| 0x14 | DHT_POOL_GET_RESP | DHT → Any | Response to DHT_POOL_GET |
| 0x15 | NODE_WITHDRAW | Node → DHT | Signal node withdrawal from network |
| 0x16 | PATH_QUALITY_REPORT | Node → Client | Report path quality with domain ID (encrypted for client) |
| 0x17 | PT_HANDSHAKE | Any → Any | Pluggable transport handshake |
| 0x18 | MESH_BEACON | Node → Broadcast | Mesh network presence announcement |
| 0x19 | MESH_FRAGMENT | Any → Any | Fragment of a larger BGP-X packet (low-MTU transports) |
| 0x1A | DOMAIN_ADVERTISE | Node → DHT | Announce routing domain endpoints and bridge capability |
| 0x1B | MESH_ISLAND_ADVERTISE | Island gateway → DHT | Announce mesh island existence, bridges, and services |
| 0x1C | POOL_KEY_ROTATION | Curator → DHT | Pool curator key rotation record |

---

## 5. Common Header

Every BGP-X message begins with the 32-byte common outer header. See `/protocol/packet_format.md` Section 3 for the complete bit layout.

### 5.1 Bidirectional Sequence Numbers

Each session maintains TWO independent sequence number spaces:

- **Outbound**: sequence numbers for packets sent by this node (carried in common header)
- **Inbound**: sequence numbers for packets received (tracked in sliding window replay detection)

This separation prevents nonce reuse even though the same session key is used for both directions. Domain-agnostic — identical mechanism in all routing domains.

---

## 6. RELAY Message (0x01)

The RELAY message is the primary data-forwarding message within a routing domain. It carries onion-encrypted payloads. Cover traffic (0x11) uses the same session_key as RELAY and is externally indistinguishable.

### Relay Node Behavior

Upon receiving a RELAY message:

1. Look up session key for the Session ID (INBOUND sequence space)
2. If no session: drop silently, log internally
3. Check inbound sequence window for replay
4. Decrypt using session_key and sequence number as nonce input
5. If authentication tag mismatch: drop silently
6. Parse 57-byte onion layer header
7. Validate path_id (not all-zeros)
8. Store `path_id → source_addr` in path table (TTL = session idle timeout)
9. Dispatch based on hop_type (0x01–0x09)

### Relay Node Behavior at DOMAIN_BRIDGE Hop (0x06)

When hop_type = 0x06:

1. Parse target domain ID (bytes 0-7 of next_hop)
2. Parse bridge node ID (bytes 8-39 of next_hop)
3. Store `path_id → source_addr` in current domain path table
4. Store `path_id → {source_domain, source_addr, target_domain}` in cross-domain path table
5. Select transport appropriate to target domain
6. Forward remaining onion payload via target-domain transport to bridge node
7. No re-encryption: the remaining payload is already encrypted for subsequent hops

### Cover Traffic (0x11)

COVER packets use session_key identically to RELAY. The COVER payload is random bytes formatted to resemble valid onion structure. If the onion parse produces uninterpretable content after decryption, the node drops silently — this is expected and correct. External observers cannot distinguish COVER from RELAY in any routing domain.

---

## 7. HANDSHAKE Messages — Domain-Agnostic

The BGP-X handshake protocol is completely domain-agnostic. Whether the session is established over UDP/IP, WiFi mesh, LoRa, Bluetooth, or satellite:

- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE message format
- Same X25519 key exchange
- Same ChaCha20-Poly1305 encryption
- Same HKDF-SHA256 key derivation with identical salts and info strings
- Same HANDSHAKE_DONE path_id delivery

See `/protocol/handshake.md` for the complete handshake specification.

### Critical: HANDSHAKE_INIT Two-Part Format

HANDSHAKE_INIT MUST have a cleartext portion (client ephemeral public key, bytes 32-63) and an encrypted portion (bytes 64+). The node needs the client's ephemeral public key to compute dh1 before it can decrypt anything. This is not a security problem — the public key is not secret.

### Cross-Domain Session Establishment

For paths crossing domain boundaries, the client establishes sessions with all hops including bridge nodes. Sessions with post-bridge hops are delivered via the bridge node's established session — the bridge node relays handshake packets into the mesh island via its radio. The mesh relay never directly receives a UDP packet from the client.

---

## 8. DOMAIN_ADVERTISE Message (0x1A)

Nodes with bridge capability publish their domain bridge records to the unified DHT.

See `/protocol/packet_format.md` Section 10 for wire format.

DHT storage key:
```
key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id, to_domain_id) || max(from_domain_id, to_domain_id))
TTL: 24 hours. Re-publication: every 8 hours.
```

Symmetric ordering: A→B and B→A use the same key. Bridge discovery is bidirectional.

Storage nodes MUST verify DOMAIN_ADVERTISE signatures before storing. The advertisement is signed by the node's private key (same Ed25519 keypair as node advertisement).

---

## 9. MESH_ISLAND_ADVERTISE Message (0x1B)

Mesh island gateway nodes publish island advertisements to the unified DHT when internet-connected.

See `/protocol/packet_format.md` Section 11 for wire format.

DHT storage key:
```
key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8)
TTL: 24 hours. Re-publication: every 8 hours.
```

When no bridge node with internet is available, the island advertisement exists only in the local mesh DHT. When a bridge node reconnects, the island advertisement is published to the unified internet DHT automatically.

---

## 10. DHT_FIND_NODE with Domain Filter

DHT_FIND_NODE is extended to support optional domain filtering:

```
Standard payload:
  Target NodeID (32 bytes)

Extended payload:
  Target NodeID (32 bytes)
  Domain filter flag (1 byte): 0x00 = no filter, 0x01 = filter active
  Domain ID (8 bytes): only present if flag = 0x01
```

When domain filter is active: storage nodes MUST return only contacts with advertised endpoints in the specified routing domain.

---

## 11. DATA and STREAM Messages

Stream multiplexing, DATA message flags, STREAM_OPEN/CLOSE formats, stream ID parity rules — all domain-agnostic by design. Unchanged from single-domain specification.

**Stream ID parity (MANDATORY)**:
- Client-initiated streams: ODD IDs (1, 3, 5, ...)
- Service-initiated streams: EVEN IDs (2, 4, 6, ...)
- Violation: sender receives RST; error logged internally

**Cross-domain stream continuity**: a stream maintains its stream_id across all domain boundaries in a cross-domain path. The stream is not re-identified at domain transitions.

### DATA Message Flags

| Bit | Name | Description |
|---|---|---|
| 0 | FIN | Last data segment on this stream |
| 1 | RST | Reset stream immediately |
| 2 | SYN | First data segment on new stream |
| 8 | CONGESTION_MILD | Node queue above 75% capacity |
| 9 | CONGESTION_SEVERE | Node queue above 90% capacity |
| 10 | CONGESTION_CLEAR | Congestion resolved |
| 11 | ECH_NEGOTIATED | ECH was used for this stream |
| 12 | ECH_AVAILABLE | Destination supports ECH |
| 3-7, 13-15 | Reserved | MUST be 0 |

### STREAM_OPEN Flags

| Bit | Name | Description |
|---|---|---|
| 0 | ECH_REQUIRED | Fail stream if ECH not available at exit |
| 1 | ECH_PREFERRED | Use ECH when available (default: true) |
| 2-15 | Reserved | |

### STREAM_CLOSE Reason Codes

| Code | Meaning |
|---|---|
| 0x0000 | Normal close |
| 0x0001 | Remote error |
| 0x0002 | Destination unreachable |
| 0x0003 | Exit policy denied |
| 0x0004 | Timeout |
| 0x0005 | Protocol error |
| 0x0006 | ECH required but not available |
| 0x0007 | ECH negotiation failed |
| 0x0008 | Domain bridge unavailable |
| 0x0009 | Mesh island unreachable |

---

## 12. KEEPALIVE Message (0x09)

```
BGP-X Common Header (Msg Type = 0x09)
(No payload)
```

Nodes MUST send KEEPALIVE at 25-second intervals on idle sessions, randomized within ±5 seconds. This randomization is MANDATORY — not optional — to prevent KEEPALIVE timing fingerprinting.

Session dead if no KEEPALIVE or DATA received for 90 seconds.

For high-latency domain segments (LoRa, satellite GEO): KEEPALIVE interval and session dead timeout SHOULD be extended (configurable up to 5 minutes).

---

## 13. NODE_WITHDRAW Message (0x15)

Wire format: see `/protocol/packet_format.md` Section 16.

On receiving NODE_WITHDRAW, storage nodes MUST:
1. Verify signature using the NodeID's public key
2. If valid: immediately remove advertisement from DHT storage
3. Propagate to 5 nearest DHT peers (propagation limit prevents infinite gossip)

On receiving NODE_WITHDRAW, routing nodes MUST:
1. Remove node from DHT routing table k-buckets
2. Remove node from path-eligible node database

For domain bridge nodes: after NODE_WITHDRAW, also remove from bridge pair records.

---

## 14. PATH_QUALITY_REPORT Message (0x16)

See `/protocol/packet_format.md` Section 12 for wire format. Extended to include domain ID.

Intermediate relays and bridge nodes forward PATH_QUALITY_REPORT opaquely via path_id routing — no decryption at any intermediate node, including across domain boundaries.

---

## 15. MESH_BEACON Message (0x18)

See `/protocol/packet_format.md` Section 17 for wire format. Extended to include bridge capability flag and served domain IDs.

Broadcast every 30 seconds on all active mesh transports. ±5 seconds jitter.

Receiving nodes MUST verify signature before adding sender to DHT routing table.

---

## 16. Node Advertisement Record

The node advertisement now uses `routing_domains` and `bridges` fields. Legacy `endpoints` and `mesh_endpoints` remain valid for backward compatibility.

```json
{
  "version": 1,
  "node_id": "<32-byte BLAKE3 hash, hex>",
  "public_key": "<32-byte Ed25519 public key, base64url>",
  "roles": ["relay", "entry", "discovery"],

  "routing_domains": [
    {
      "domain_type": "clearnet",
      "domain_id": "0x00000001-00000000",
      "endpoints": [
        { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
        { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
      ]
    }
  ],

  "bridge_capable": false,
  "bridges": [],

  "exit_policy": null,
  "exit_policy_version": null,
  "bandwidth_mbps": 100,
  "latency_ms": 12,
  "uptime_pct": 99.1,
  "region": "EU",
  "country": "DE",
  "asn": 12345,
  "operator_id": "<base64url Ed25519 public key>",
  "pool_memberships": ["pool-id-hex-1"],
  "island_memberships": [],
  "pt_supported": ["obfs4"],
  "ech_capable": false,
  "is_gateway": false,
  "gateway_for_mesh": null,
  "provides_clearnet_exit": false,
  "link_quality_profiles": {
    "lora": { "latency_class": 2, "bandwidth_class": 1 }
  },
  "withdrawn": false,
  "withdrawn_at": null,
  "protocol_versions_min": 1,
  "protocol_versions_max": 1,
  "extensions": {
    "cover_traffic": true,
    "multiplexing": true,
    "path_quality_reporting": true,
    "pluggable_transport": true,
    "ech_support": false,
    "geo_plausibility": true,
    "pool_support": true,
    "mesh_transport": false,
    "advertisement_withdrawal": true,
    "session_rehandshake": true,
    "domain_bridge": false,
    "cross_domain_routing": true,
    "mesh_island_routing": false
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-04-25T12:00:00Z",
  "signature": "<64-byte Ed25519 signature, base64url>"
}
```

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

## 17. Replay Protection

Two independent 64-sequence-number sliding windows per session (inbound and outbound). Initial sequence numbers are random (CSPRNG, uint64). Window size: 64 positions. Domain-agnostic — identical mechanism in all routing domains.

```
if seq > inbound_max:
    advance inbound window, mark seq as received → ACCEPT
elif inbound_max - seq >= 64:
    → REJECT (too old)
elif inbound_window.bit_set(inbound_max - seq):
    → REJECT (replay detected)
else:
    inbound_window.set_bit(inbound_max - seq) → ACCEPT
```

---

## 18. MTU and Fragmentation

Maximum BGP-X packet size: **1280 bytes**. BGP-X MUST NOT fragment packets above the transport layer. MESH_FRAGMENT (0x19) handles fragmentation at the transport layer for low-MTU transports.

| Transport | MTU Available |
|---|---|
| UDP/IP | 1280 bytes |
| WiFi 802.11s | 1280 bytes |
| LoRa SF7 | ~200 bytes usable |
| LoRa SF12 | ~50 bytes usable |
| Bluetooth BLE | 100-200 bytes |
| Ethernet P2P | 1500 bytes (BGP-X still targets 1280) |

---

## 19. Protocol State Machine

### Node State Machine

```
INITIAL → BOOTSTRAP → ACTIVE → DRAIN → STOP
```

On DRAIN entry: publish NODE_WITHDRAW; mark all bridge pairs as unavailable in DOMAIN_ADVERTISE and re-publish; wait up to 30 seconds; stop accepting new handshakes; drain existing sessions.

### Session State Machine

```
NO SESSION → HANDSHAKING → ESTABLISHED → CLOSED
```

---

## 20. Extension Flags

| Bit | Extension | v0.1 |
|---|---|---|
| 0 | Cover traffic support | Yes |
| 1 | Pluggable transport support | Yes |
| 2 | Stream multiplexing | Yes |
| 3 | Path quality reporting (with domain ID) | Yes |
| 4 | ECH support (exit nodes only) | Yes |
| 5 | Geographic plausibility scoring | Yes |
| 6 | Pool support (including domain-scoped pools) | Yes |
| 7 | Mesh transport support | Yes |
| 8 | Advertisement withdrawal | Yes |
| 9 | Session re-handshake (24hr) | Yes |
| 10 | Domain bridge support | Yes |
| 11 | Cross-domain path construction | Yes |
| 12 | Mesh island routing | Yes |
| 13-31 | Reserved | No |

Bit 10: node can act as a domain bridge (handles DOMAIN_BRIDGE, MESH_ENTRY, MESH_EXIT, MESH_RELAY hop types; has valid bridge pair DHT records).

Bit 11: node's path selection supports domain_segments construction.

Bit 12: node participates in mesh island DHT registration and island advertisement propagation.

---

## 21. Compliance Requirements

### MUST

- Implement all message types in Section 4
- Implement common header with bidirectional sequence numbers
- Implement onion encryption with path_id field in all layers
- Implement two-part HANDSHAKE_INIT (cleartext ephemeral key + encrypted payload)
- Implement replay protection (separate inbound/outbound windows)
- Respect MTU limits
- Implement state machines
- Verify all node advertisement signatures before using or storing
- Enforce geographic and ASN diversity in path selection
- Verify DHT_PUT advertisement signatures before storing
- Implement path_id extraction and path table management
- Implement NODE_WITHDRAW propagation
- Implement POOL_KEY_ROTATION verification
- Implement KEEPALIVE randomization (±5 seconds, MANDATORY)
- Implement two independent sequence number spaces
- For bridge-capable nodes: implement DOMAIN_BRIDGE (0x06) forwarding
- For bridge-capable nodes: maintain cross-domain path_id routing table
- For bridge-capable nodes: publish DOMAIN_ADVERTISE records
- Support domain-filtered DHT_FIND_NODE
- Enforce N-hop unlimited: MUST NOT impose a maximum hop count at the protocol level
- Participate in one unified DHT (MUST NOT maintain separate per-domain DHTs)
- For exit nodes: implement DoH DNS with DNSSEC validation and ECS stripping
- For exit nodes: implement ECH when ech_capable=true
- NEVER log: client IPs at relay/exit, destinations at relay, path_id values, path composition, session IDs, traffic volume per path, pool query history, cross-domain traversal details

### MUST NOT

- Log prohibited items (see above)
- Allow disabling signature verification
- Allow disabling replay protection
- Attempt to decrypt onion layers beyond this node's own layer
- Re-encrypt onion payload when transitioning between routing domains at bridge nodes
- Reuse (session_key, nonce) pairs
- Use a separate cover_key (COVER uses session_key)
- Enforce a maximum hop count at the protocol level
- Maintain separate per-routing-domain DHTs

### SHOULD

- Support IPv6
- Support cover traffic
- Support pluggable transport
- Implement reputation system
- Implement geographic plausibility scoring with domain-specific thresholds
- Support pool-based path construction including domain-scoped pools
- Support mesh transport on appropriate hardware
- Implement session re-handshake for sessions exceeding 24 hours
- Support DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE message types
- Support cross-domain path construction
