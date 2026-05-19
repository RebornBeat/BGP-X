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

All BGP-X implementations must conform to this specification. Where this document is ambiguous, implementors should err toward the more privacy-preserving interpretation and open an issue for clarification.

---

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

| Term | Definition |
|---|---|
| Node | Any participant in the BGP-X overlay network |
| Client | An initiating party that constructs and sends overlay packets |
| Entry node | The first relay hop; knows client transport address, not destination |
| Relay node | A middle hop; knows only previous and next hop |
| Exit node / Gateway | The final hop for clearnet traffic; knows destination, not client |
| Domain bridge node | A node with endpoints in 2+ routing domains; enables cross-domain paths |
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
| Clearnet | All BGP-routed internet — including satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) |
| Satellite internet | Commercial satellite providers (Starlink, Iridium, etc.) — provides BGP-routed IP connectivity; clearnet domain (0x00000001), NOT a separate domain |
| bgpx-satellite (0x00000005) | RESERVED for future BGP-X-native satellite network; NOT active in current deployments |
| Jurisdiction | OPTIONAL node-declared legal jurisdiction for geo plausibility scoring |
| Geo plausibility | OPTIONAL reputation signal based on RTT measurements; applies only if jurisdiction declared |

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

BGP-X defines the following message types identified by a 1-byte type field in the outer header.

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

Every BGP-X message begins with the 32-byte common outer header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────┬───────────────┬───────────────────────────────┤
│    Version    │   Msg Type    │          Reserved             │
├───────────────┴───────────────┴───────────────────────────────┤
│                        Session ID                             │
│                        (16 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│                    Outbound Sequence Number                    │
│                         (8 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│                     Payload Length                            │
│                        (4 bytes)                              │
├───────────────────────────────────────────────────────────────┤
│                         Payload                               │
│                      (variable)                               │
└───────────────────────────────────────────────────────────────┘
```

| Field | Size | Description |
|---|---|---|
| Version | 1 byte | Protocol version. Current: 0x01 |
| Msg Type | 1 byte | Message type identifier (see Section 4) |
| Reserved | 2 bytes | MUST be 0x0000 |
| Session ID | 16 bytes | Randomly generated per session. 0x00...00 for pre-session messages |
| Outbound Sequence Number | 8 bytes | Monotonically increasing for OUTBOUND direction only. Separate from inbound sequence space. |
| Payload Length | 4 bytes | Length of the payload in bytes, big-endian |
| Payload | Variable | Message-type-specific content |

Total header size: **32 bytes**

All multi-byte integer fields are big-endian (network byte order).

### 5.1 Bidirectional Sequence Numbers

Each session maintains TWO independent sequence number spaces:

- **Outbound**: sequence numbers for packets sent by this node to the next hop
- **Inbound**: sequence numbers for packets received from the previous hop

The common header carries the outbound sequence number. The replay detection window tracks the inbound sequence space separately.

This separation prevents nonce reuse even though the same session key is used for both directions.

---

## 6. RELAY Message (0x01)

The RELAY message is the primary data-forwarding message within a routing domain. It carries onion-encrypted payloads. Cover traffic (0x11) uses the same session_key as RELAY and is externally indistinguishable.

```
Common Header (Msg Type = 0x01)
├───────────────────────────────────────────────────────────────┤
│                     Onion Payload                             │
│          (ChaCha20-Poly1305 encrypted, this hop only)        │
└───────────────────────────────────────────────────────────────┘
```

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

**Important**: COVER packets use the **same session_key as RELAY packets**. There is no separate cover_key. Both use ChaCha20-Poly1305 encryption, and are externally indistinguishable.

---

## 7. Onion Layer Structure

The onion layer is the decrypted content of a RELAY message payload. After decryption at a relay node, the layer contains:

```
Byte offset:
  0        Hop Type (1 byte)
  1-40     Next Hop (40 bytes)
  41-48    path_id (8 bytes) — return traffic routing identifier
  49-52    Stream ID (4 bytes)
  53-54    Flags (2 bytes)
  55-56    Reserved (2 bytes, MUST be 0x0000)
  57+      Remaining Ciphertext (for subsequent hops)
```

Total layer header: **57 bytes**

### path_id Field

The path_id (bytes 41-48) is:
- Generated by the client using CSPRNG (8 bytes, 64 bits)
- Unique per path instance
- Included in EVERY onion layer of the same path
- The same value at every hop of the same path
- Never reused within a session
- Not linked to any client identity
- Not logged by any relay node

When a relay decrypts its layer and extracts path_id, it stores `path_id → source_addr` in an in-memory table. This mapping enables return traffic routing without requiring the relay to know the full path structure.

### Hop Type Values

| Value | Meaning |
|---|---|
| 0x01 | Relay — forward to another BGP-X node |
| 0x02 | Exit — forward to clearnet destination (IPv4) |
| 0x03 | Exit — forward to clearnet destination (IPv6) |
| 0x04 | Exit — forward to domain name (resolve at exit) |
| 0x05 | Delivery — this is the final hop (BGP-X native service) |
| 0x06 | Domain Bridge — transition to another routing domain |
| 0x07 | Mesh Entry — enter a mesh island from overlay |
| 0x08 | Mesh Exit — leave a mesh island to overlay |
| 0x09 | Mesh Relay — relay within a mesh island |

---

## 8. HANDSHAKE Messages — Domain-Agnostic

The BGP-X handshake protocol is completely domain-agnostic. Whether the session is established over UDP/IP, WiFi mesh, LoRa, Bluetooth, or satellite:

- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE message format
- Same X25519 key exchange
- Same ChaCha20-Poly1305 encryption
- Same HKDF-SHA256 key derivation with identical salts and info strings
- Same HANDSHAKE_DONE path_id delivery

See `/protocol/handshake.md` for the complete handshake specification.

### Critical: HANDSHAKE_INIT Two-Part Format

HANDSHAKE_INIT MUST have a cleartext portion and an encrypted portion:

**Part 1 (Cleartext, 32 bytes)**: Client Ephemeral Public Key (X25519, 32 bytes)

This MUST be cleartext because the receiving node needs the client's ephemeral public key to compute `dh1 = X25519(node_static_priv, client_ephemeral_pub)` before it can derive any key material to decrypt anything.

This is NOT a security problem — the ephemeral public key is not secret (only the private key is secret, which never leaves the client).

**Part 2 (Encrypted, variable)**: All other HANDSHAKE_INIT content, encrypted using `init_key` derived from `dh1`.

Full wire format: see `/protocol/packet_format.md`.

### Cross-Domain Session Establishment

For paths crossing domain boundaries, the client establishes sessions with all hops including bridge nodes. Sessions with post-bridge hops are delivered via the bridge node's established session — the bridge node relays handshake packets into the mesh island via its radio. The mesh relay never directly receives a UDP packet from the client.

---

## 9. DOMAIN_ADVERTISE Message (0x1A)

Nodes with bridge capability publish their domain bridge records to the unified DHT.

See `/protocol/packet_format.md` Section 10 for wire format.

DHT storage key:
```
key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id, to_domain_id) || max(from_domain_id, to_domain_id))
TTL: 24 hours. Re-publication: every 8 hours.
```

Symmetric ordering: A→B and B→A use the same key. Bridge discovery is bidirectional.

Storage nodes MUST verify DOMAIN_ADVERTISE signatures before storing. The advertisement is signed by the node's private key (same Ed25519 keypair as node advertisement).

**Important**: A bridge node must have verified endpoints in BOTH domains it claims to bridge. Storage nodes should verify this by checking the node advertisement's `routing_domains` field.

---

## 10. MESH_ISLAND_ADVERTISE Message (0x1B)

Mesh island gateway nodes publish island advertisements to the unified DHT when internet-connected.

See `/protocol/packet_format.md` Section 11 for wire format.

DHT storage key:
```
key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8)
TTL: 24 hours. Re-publication: every 8 hours.
```

When no bridge node with internet is available, the island advertisement exists only in the local mesh DHT. When a bridge node reconnects, the island advertisement is published to the unified internet DHT automatically.

---

## 11. DHT_FIND_NODE with Domain Filter

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

### DHT Storage Authentication Requirement

Storage nodes MUST verify advertisement signatures before accepting a DHT_PUT:

1. Parse the advertisement from the DHT_PUT payload
2. Verify the signature using the advertisement's public_key field
3. Verify node_id = BLAKE3(public_key)
4. Verify timestamps are valid
5. Reject and drop the DHT_PUT silently if any verification fails

Implementations that accept unsigned or invalid records into DHT storage are non-compliant.

---

## 12. DATA Message

```
Common Header (Msg Type = 0x05)
├───────────────────────────────────────────────────────────────┤
│                      Stream ID (4 bytes)                      │
├───────────────────────────────────────────────────────────────┤
│                    Flags (2 bytes)                            │
├───────────────────────────────────────────────────────────────┤
│               Encrypted Application Data                      │
└───────────────────────────────────────────────────────────────┘
```

### Data Message Flags

| Bit | Name | Description |
|---|---|---|
| 0 | FIN | Last data segment on this stream |
| 1 | RST | Reset stream immediately |
| 2 | SYN | First data segment on a new stream |
| 8 | CONGESTION_MILD | Node queue above 75% capacity |
| 9 | CONGESTION_SEVERE | Node queue above 90% capacity |
| 10 | CONGESTION_CLEAR | Congestion resolved (queue below 50%) |
| 11 | ECH_NEGOTIATED | ECH was used for this stream |
| 12 | ECH_AVAILABLE | Destination supports ECH |
| 3-7, 13-15 | Reserved | MUST be 0 |

Congestion flags (bits 8-10) are set by relay nodes and encrypted within the onion layer. They are NOT visible to external observers. They reach the client via the return path.

---

## 13. STREAM Messages

### STREAM_OPEN (0x06)

```
Common Header (Msg Type = 0x06)
├───────────────────────────────────────────────────────────────┤
│                     Stream ID (4 bytes)                       │
│  Bit 31: 0 = client-initiated (odd ID), 1 = service-init     │
├───────────────────────────────────────────────────────────────┤
│                  Stream Type (1 byte)                         │
│  0x01 = ORDERED, 0x02 = DATAGRAM                             │
├───────────────────────────────────────────────────────────────┤
│                  Stream Flags (2 bytes)                       │
│  Bit 0: ECH_REQUIRED (fail if ECH not available)             │
│  Bit 1: ECH_PREFERRED (use ECH if available)                 │
├───────────────────────────────────────────────────────────────┤
│                  Destination (variable, encrypted)            │
│  Address type (1B): 0x04=IPv4, 0x06=IPv6, 0x0A=domain,      │
│                     0xBB=BGP-X ServiceID                      │
│  Address (variable)                                           │
│  Port (2B)                                                    │
└───────────────────────────────────────────────────────────────┘
```

**Stream ID Assignment Rules (MANDATORY)**:
- Client-initiated streams: ODD IDs (1, 3, 5, ...). High bit (bit 31) = 0.
- Service-initiated streams: EVEN IDs (2, 4, 6, ...). High bit (bit 31) = 1.
- Violation: sender receives RST; error logged internally.

### STREAM_CLOSE (0x07)

```
Common Header (Msg Type = 0x07)
├───────────────────────────────────────────────────────────────┤
│                     Stream ID (4 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│                  Close Reason (2 bytes)                       │
└───────────────────────────────────────────────────────────────┘
```

Close Reasons:

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

### Cross-Domain Stream Continuity

A stream maintains its stream_id across all domain boundaries in a cross-domain path. The stream is not re-identified at domain transitions.

---

## 14. KEEPALIVE Message (0x09)

```
BGP-X Common Header (Msg Type = 0x09)
(No payload)
```

Nodes MUST send KEEPALIVE at 25-second intervals on idle sessions, randomized within ±5 seconds. This randomization is MANDATORY — not optional — to prevent KEEPALIVE timing fingerprinting.

Session dead if no KEEPALIVE or DATA received for 90 seconds.

For high-latency domain segments (LoRa, satellite GEO): KEEPALIVE interval and session dead timeout SHOULD be extended (configurable up to 5 minutes).

---

## 15. NODE_WITHDRAW Message (0x15)

Signals that a node is voluntarily withdrawing from the network.

```
Common Header (Msg Type = 0x15, Session ID = 0x00...00)
├───────────────────────────────────────────────────────────────┤
│                   NodeID (32 bytes)                           │
├───────────────────────────────────────────────────────────────┤
│                   Withdraw Timestamp (8 bytes, uint64, Unix)  │
├───────────────────────────────────────────────────────────────┤
│                   Signature (64 bytes, Ed25519)               │
│   Signs: BLAKE3("bgpx-withdrawal-v1" || node_id || timestamp) │
└───────────────────────────────────────────────────────────────┘
```

On receiving NODE_WITHDRAW, storage nodes MUST:
1. Verify the signature using the NodeID's public key (fetched from cached advertisement)
2. If valid: immediately remove the node advertisement from DHT storage
3. Propagate the withdrawal to 5 nearest DHT peers (propagation limit prevents infinite gossip)

On receiving NODE_WITHDRAW, routing nodes MUST:
1. Remove node from DHT routing table k-buckets
2. Remove node from path-eligible node database

For domain bridge nodes: after NODE_WITHDRAW, also remove from bridge pair records.

Nodes MUST publish NODE_WITHDRAW before graceful shutdown.

---

## 16. PATH_QUALITY_REPORT Message (0x16)

Carries a path quality report from a relay node to the client. This message travels on the return path, encrypted for the client only. Extended to include domain identification.

```
Common Header (Msg Type = 0x16)
├───────────────────────────────────────────────────────────────┤
│              Encrypted Report Payload (20 bytes, fixed)       │
│              Encrypted using client's session key for         │
│              this hop                                         │
│                                                               │
│  Decrypted content:                                           │
│    Bytes 0-1: domain_id_type (uint16 BE)                     │
│      0x0001 = clearnet                                        │
│      0x0003 = mesh                                            │
│      0x0004 = lora-regional                                   │
│    Byte 2: hop_latency_bucket                                 │
│      0x00 = <50ms, 0x01 = 50-150ms, 0x02 = 150-300ms,       │
│      0x03 = >300ms                                            │
│    Byte 3: congestion_flag                                    │
│      0x00 = none, 0x01 = mild, 0x02 = severe                 │
│    Bytes 4-15: padding (random, for size normalization)       │
└───────────────────────────────────────────────────────────────┘
```

PATH_QUALITY_REPORT is always exactly 20 bytes of payload (after encryption tag). It is sent by relay nodes that support the path_quality_reporting extension (extension flag bit 3).

Intermediate relays and bridge nodes forward PATH_QUALITY_REPORT opaquely via path_id routing — no decryption at any intermediate node, including across domain boundaries. Only the exit node and the client can generate and consume PATH_QUALITY_REPORT content. This prevents path composition leakage.

---

## 17. MESH_BEACON Message (0x18)

Broadcast by mesh nodes to announce their presence and enable bootstrap without internet connectivity.

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

MESH_BEACON is broadcast on all active mesh transports every 30 seconds, with ±5 seconds jitter.

Receiving nodes MUST verify the signature before adding the sender to their DHT routing table.

The `bridge_capable` and `served_domains` fields allow mesh nodes to discover cross-domain bridge nodes without internet DHT access.

---

## 18. MESH_FRAGMENT Message (0x19)

For mesh transports with small MTU (particularly LoRa), BGP-X packets that exceed the transport MTU are fragmented at the transport layer.

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

- Fragmentation occurs BEFORE BGP-X header construction for the original packet
- Maximum 16 fragments per packet
- Fragment timeout: 10 seconds — if not all fragments received, discard and log
- Reassembly produces a complete BGP-X packet for normal processing
- packet_id is 4 bytes random per original packet (unique for reassembly tracking)
- MTU limits by transport: LoRa SF7 ~200 bytes usable, LoRa SF12 ~50 bytes usable

---

## 19. POOL_KEY_ROTATION Message (0x1C)

See `/protocol/pool_curator_key_rotation.md` for full specification. Wire format summary:

```
Common Header (Msg Type = 0x1C, Session ID = 0x00...00)
├───────────────────────────────────────────────────────────────┤
│              Record Length (4 bytes)                          │
├───────────────────────────────────────────────────────────────┤
│              Canonical JSON of rotation record (variable)     │
│              (max 4096 bytes)                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## 20. Node Advertisement Record

Node advertisements are JSON documents stored in DHT and retrieved by clients during path construction.

### Complete Structure

```json
{
  "version": 1,
  "node_id": "<32-byte BLAKE3 hash, hex>",
  "public_key": "<32-byte Ed25519 public key, base64url>",
  "roles": ["relay", "entry"],
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 }
  ],
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
      "bandwidth_khz": 125
    }
  ],
  "exit_policy": null,
  "exit_policy_version": null,
  "bandwidth_mbps": 100,
  "latency_ms": 12,
  "uptime_pct": 99.1,
  "region": "EU",
  "country": "DE",
  "asn": 12345,
  "operator_id": "<base64url Ed25519 public key>",
  "pool_memberships": ["pool-id-1", "pool-id-2"],
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
    "mesh_transport": true,
    "advertisement_withdrawal": true,
    "session_rehandshake": true
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-04-25T12:00:00Z",
  "signature": "<base64url Ed25519 signature>"
}
```

### Node Advertisement using `routing_domains` structure

The node advertisement now supports `routing_domains` and `bridges` fields. Legacy `endpoints` and `mesh_endpoints` remain valid for backward compatibility.

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

### Domain Bridge Node Advertisement Example

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

### New Fields (beyond original spec)

| Field | Type | Description |
|---|---|---|
| mesh_endpoints | array | Transport-specific mesh addresses (WiFi mesh MAC, LoRa addr) |
| exit_policy_version | uint16 | Incremented on any exit policy change; clients re-fetch on version change |
| pool_memberships | array of strings | Pool IDs this node is currently a member of |
| pt_supported | array of strings | Pluggable transports supported (e.g., ["obfs4"]) |
| ech_capable | boolean | Whether this exit node supports ECH (requires exit role) |
| is_gateway | boolean | Whether this node bridges mesh to clearnet |
| gateway_for_mesh | array of strings | Mesh IDs this gateway serves |
| provides_clearnet_exit | boolean | Separate from is_gateway; confirms clearnet exit capability |
| link_quality_profiles | object | Per-transport latency_class and bandwidth_class |
| withdrawn | boolean | If true, node is being withdrawn; triggers propagation |
| withdrawn_at | string | ISO 8601 timestamp of withdrawal |

### Exit Policy Object (Updated)

```json
{
  "version": 1,
  "allow_protocols": ["tcp", "udp"],
  "allow_ports": [80, 443, 8080, 8443],
  "deny_destinations": ["10.0.0.0/8", "192.168.0.0/16", "172.16.0.0/12"],
  "logging_policy": "none",
  "operator_contact": "operator@example.com",
  "jurisdiction": "DE",
  "dns_mode": "doh",
  "dns_resolver": "https://dns.quad9.net/dns-query",
  "dnssec_validation": true,
  "ecs_handling": "strip",
  "ech_capable": true,
  "policy_version": 1,
  "policy_previous_version": 0,
  "policy_signed_at": "2026-04-24T12:00:00Z",
  "policy_signature": "<Ed25519 signature by operator_id key>"
}
```

New exit policy fields:

| Field | Description |
|---|---|
| dns_mode | "doh", "dot", "system", "custom" |
| dns_resolver | URL or IP of DNS resolver |
| dnssec_validation | Whether DNSSEC is validated |
| ecs_handling | "strip" (mandatory), "pass", "anonymize" |
| ech_capable | Whether exit supports ECH |
| policy_version | Monotonically increasing version number |
| policy_previous_version | Previous version for transition tracking |

### Signature Verification

Receivers MUST:

1. Recompute node_id = BLAKE3(public_key) and verify it matches the node_id field
2. Verify the signature over the canonical JSON using the public_key field
3. Verify that signed_at is not in the future (allowing 60 seconds of clock skew)
4. Verify that expires_at has not passed
5. Reject the advertisement if any verification step fails

---

## 21. Replay Protection

Each session maintains TWO independent 64-sequence-number sliding windows:

- **Outbound window**: tracks sequence numbers this node sends (prevents outbound replay)
- **Inbound window**: tracks sequence numbers this node receives (prevents inbound replay)

Initial sequence numbers are random (CSPRNG, uint64). Window size: 64 positions.

Replay detection is applied to the INBOUND window for received packets:

```
if seq > inbound_max:
    advance inbound window, mark seq as received
    return ACCEPT
elif inbound_max - seq >= 64:
    return REJECT("Too old")
elif inbound_window.bit_set(inbound_max - seq):
    return REJECT("Replay detected")
else:
    inbound_window.set_bit(inbound_max - seq)
    return ACCEPT
```

---

## 22. MTU and Fragmentation

BGP-X MUST NOT fragment packets at the BGP-X protocol layer (above transport).

Maximum BGP-X packet size: **1280 bytes** (targeting minimum IPv6 MTU for UDP transport)

BGP-X overhead: approximately 120 bytes (header + encryption)

Maximum application payload per packet: **1160 bytes**

For mesh transports with smaller MTU, the MESH_FRAGMENT mechanism (message type 0x19) handles fragmentation at the transport layer, below the BGP-X packet level.

Oversized packets received at any node MUST be dropped silently. The sender MUST ensure compliance with the MTU before transmission.

| Transport | MTU Available | Notes |
|---|---|---|
| UDP/IP | 1280 bytes | BGP-X target MTU |
| WiFi 802.11s | 1280 bytes | Same as UDP/IP |
| LoRa SF7 | ~200 bytes usable | Requires MESH_FRAGMENT |
| LoRa SF12 | ~50 bytes usable | Requires MESH_FRAGMENT |
| Bluetooth BLE | 100-200 bytes | Requires MESH_FRAGMENT |
| Ethernet P2P | 1500 bytes | BGP-X target MTU still applies |

---

## 23. Protocol State Machine

### 23.1 Node State Machine

```
INITIAL → BOOTSTRAP → ACTIVE → DRAIN → STOP
```

On DRAIN entry: publish NODE_WITHDRAW; mark all bridge pairs as unavailable in DOMAIN_ADVERTISE and re-publish; wait up to 30 seconds for propagation; then stop accepting new handshakes and drain existing sessions.

### 23.2 Session State Machine (per session)

```
NO SESSION → HANDSHAKING → ESTABLISHED → CLOSED
```

---

## 24. Extension Flags

Extension flags are negotiated during handshake (HANDSHAKE_INIT extensions field, HANDSHAKE_RESP accepted extensions):

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

## 25. HTTP/2 over BGP-X Streams

BGP-X native services (.bgpx) use HTTP/2 over BGP-X streams.

HTTP/2 is selected over HTTP/3 because:

- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers — this is standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 26. Compliance Requirements

### MUST

- Implement all message types in Section 4
- Implement common header (Section 5) with bidirectional sequence numbers
- Implement onion encryption scheme from `/protocol/packet_format.md` with path_id field in all layers
- Implement handshake from `/protocol/handshake.md` with two-part HANDSHAKE_INIT
- Implement replay protection (Section 21) with separate inbound/outbound windows
- Respect MTU limits (Section 22)
- Implement state machines (Section 23)
- Verify all node advertisement signatures before using or storing
- Enforce geographic and ASN diversity in path selection
- Verify DHT_PUT advertisement signatures before storing (Section 11)
- Implement path_id extraction and path table management
- Implement NODE_WITHDRAW propagation
- Implement POOL_KEY_ROTATION verification
- Implement KEEPALIVE randomization (±5 seconds, MANDATORY)
- Implement two independent sequence number spaces (Section 5.1)
- For bridge-capable nodes: implement DOMAIN_BRIDGE (0x06) forwarding
- For bridge-capable nodes: maintain cross-domain path_id routing table
- For bridge-capable nodes: publish DOMAIN_ADVERTISE records
- Support domain-filtered DHT_FIND_NODE
- Enforce N-hop unlimited: MUST NOT impose a maximum hop count at the protocol level
- Participate in one unified DHT (MUST NOT maintain separate per-domain DHTs)
- For exit nodes: implement DoH DNS resolution with DNSSEC validation and ECS stripping
- For exit nodes: implement ECH when ech_capable=true
- NEVER log prohibited items (see below)

### MUST NOT

- Log prohibited items:
  - Client IPs at relay/exit nodes
  - Destinations at relay nodes
  - path_id values
  - Path composition
  - Session IDs
  - Traffic volume per path
  - Pool query history
  - Cross-domain traversal details per session
  - Mesh island identifiers associated with specific sessions
- Allow disabling of signature verification
- Allow disabling of replay protection
- Attempt to decrypt onion layers beyond this node's own layer
- Re-encrypt onion payload when transitioning between routing domains at bridge nodes
- Reuse (session_key, nonce) pairs
- Use a separate cover_key (COVER uses session_key)
- Enforce a maximum hop count at the protocol level
- Maintain separate per-routing-domain DHTs
- Accept unsigned or invalid records into DHT storage

### SHOULD

- Support IPv6
- Support cover traffic generation
- Support pluggable transport
- Implement reputation system
- Implement geographic plausibility scoring with domain-specific thresholds
- Support pool-based path construction including domain-scoped pools
- Support mesh transport on appropriate hardware
- Implement session re-handshake for sessions exceeding 24 hours
- Support DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE message types
- Support cross-domain path construction
- Provide extension bit 10 (domain_bridge) when configured with 2+ routing domains
- Maintain bridge latency measurements per bridge pair
- Verify domain bridge advertisements against node advertisement routing_domains

---

## 27. Test Vectors

Test vectors for BGP-X protocol verification are documented in `/protocol/test_vectors/README.md`.

Required test vector categories:

- Common header serialization/deserialization
- Onion layer construction and parsing (all hop types 0x01-0x09)
- HANDSHAKE_INIT two-part format
- HANDSHAKE_RESP encryption
- HANDSHAKE_DONE with path_id
- Node advertisement signature verification
- Pool advertisement signature verification
- DOMAIN_ADVERTISE signature verification
- MESH_ISLAND_ADVERTISE signature verification
- PATH_QUALITY_REPORT with domain ID
- NODE_WITHDRAW signature verification
- POOL_KEY_ROTATION dual-signature verification
- MESH_FRAGMENT reassembly
- MESH_BEACON verification
- Replay window edge cases
- Negative test vectors (invalid signatures, malformed layers, path_id = 0x00...00)

---

## Appendix A: Cross-Domain Routing Summary

| Source Domain | Destination Domain | Bridge Node Requirement |
|---|---|---|
| Clearnet | Clearnet | No bridge needed |
| Clearnet | Mesh island | Domain bridge node with clearnet + mesh endpoints |
| Mesh island | Clearnet | Domain bridge node with mesh + clearnet endpoints (gateway) |
| Mesh island | Mesh island (same) | No bridge needed (intra-island) |
| Mesh island | Mesh island (different) | Domain bridge node OR internet relay pool |

Cross-domain paths use the same onion encryption throughout. Domain bridge nodes decrypt only the DOMAIN_BRIDGE layer and forward remaining encrypted payload to the next domain. No re-encryption occurs at domain boundaries.
