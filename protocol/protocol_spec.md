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
| Entry node | The first relay hop; knows client IP, not destination |
| Relay node | A middle hop; knows only previous and next hop |
| Exit node / Gateway | The final hop for clearnet traffic; knows destination, not client IP |
| Path | An ordered sequence of nodes from entry to exit |
| Segment | A portion of a path from one pool; paths may have multiple segments |
| Stream | A logical bidirectional channel multiplexed over a path |
| Session | The cryptographic context for communication between a client and one node |
| Onion packet | A layered-encrypted packet progressively decrypted at each hop |
| NodeID | BLAKE3(Ed25519_public_key) — 32 bytes |
| ServiceID | Ed25519 public key of a BGP-X native service — 32 bytes |
| path_id | 8-byte CSPRNG-generated identifier per path; enables return traffic routing |
| Pool | A named, signed collection of nodes grouped by trust or operational intent |
| Pool ID | BLAKE3 hash identifying a pool, 32 bytes |
| Curator | Entity that signs pool advertisements and membership records |
| Advertisement | A signed record published by a node to the DHT |
| DHT | Distributed Hash Table used for node discovery |
| PT | Pluggable Transport — traffic obfuscation layer below BGP-X |

---

## 3. Protocol Overview

### 3.1 Underlay (Transport)

BGP-X runs over UDP/IP for internet deployments. For mesh deployments, BGP-X runs over WiFi 802.11s, LoRa, Bluetooth BLE, or Ethernet point-to-point.

The protocol is transport-agnostic. All BGP-X messages have the same format regardless of transport.

Default BGP-X port (UDP): **7474/UDP**

Implementations MUST support IPv4. Implementations SHOULD support IPv6. Dual-stack operation is RECOMMENDED.

For mesh transports, addressing uses NodeID rather than IP addresses. The mesh transport layer maps NodeIDs to transport-specific addresses.

### 3.2 Overlay (BGP-X Protocol)

The BGP-X overlay protocol operates within transport payloads. It defines:

- Packet types and their wire formats
- Onion encryption structure (including path_id field)
- Session establishment and teardown
- Stream multiplexing
- Control plane messages (node advertisement, DHT operations, pool operations)
- Mesh transport operations

---

## 4. Message Types

BGP-X defines the following message types identified by a 1-byte type field in the outer header.

| Type ID | Name | Direction | Description |
|---|---|---|---|
| 0x01 | RELAY | Any → Any | Forward an onion-encrypted payload |
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
| 0x11 | COVER | Any → Any | Cover traffic (indistinguishable from RELAY) |
| 0x12 | DHT_POOL_ADVERTISE | Curator → DHT | Publish pool advertisement |
| 0x13 | DHT_POOL_GET | Any → DHT | Retrieve a pool advertisement |
| 0x14 | DHT_POOL_GET_RESP | DHT → Any | Response to DHT_POOL_GET |
| 0x15 | NODE_WITHDRAW | Node → DHT | Signal node withdrawal from network |
| 0x16 | PATH_QUALITY_REPORT | Node → Client | Report path quality (encrypted for client) |
| 0x17 | PT_HANDSHAKE | Any → Any | Pluggable transport handshake |
| 0x18 | MESH_BEACON | Node → Broadcast | Mesh network presence announcement |
| 0x19 | MESH_FRAGMENT | Any → Any | Fragment of a larger BGP-X packet (low-MTU transports) |
| 0x1A | POOL_KEY_ROTATION | Curator → DHT | Pool curator key rotation record |

---

## 5. Common Header

Every BGP-X message begins with a common outer header:

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

## 6. RELAY Message

The RELAY message is the primary data-forwarding message.

```
Common Header (Msg Type = 0x01)
├───────────────────────────────────────────────────────────────┤
│                     Onion Payload                             │
│          (ChaCha20-Poly1305 encrypted, this hop only)        │
└───────────────────────────────────────────────────────────────┘
```

### Relay Node Behavior

Upon receiving a RELAY message, a node MUST:

1. Verify the outer header (Version, Reserved fields)
2. Look up the session key for the Session ID using the INBOUND sequence space
3. If no session: drop silently, log internally
4. Attempt decryption using session key and Outbound Sequence Number as nonce input
5. If authentication tag mismatch: drop silently, log internally
6. Extract path_id (bytes 1-8 of decrypted layer header — see Section 7)
7. Record: `path_id → source_address` in-memory path table (TTL = session idle timeout)
8. Extract next-hop address from decrypted layer header
9. Forward remaining encrypted payload to next hop (with new outer header using node's outbound session to next hop)

Relay nodes MUST NOT:
- Log the source address of RELAY messages
- Log path_id values
- Log payload content
- Attempt to decrypt more than their own layer
- Modify the payload before forwarding

### Cover Traffic Behavior

COVER packets (0x11) use the same session_key as RELAY packets. They MUST be processed identically to RELAY packets from a forwarding perspective. If the onion layer parse fails (random payload), the node drops silently — this is expected and correct. External observers cannot distinguish COVER from RELAY.

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

---

## 8. HANDSHAKE Messages

See `/protocol/handshake.md` for the full handshake specification.

### Critical Structure: HANDSHAKE_INIT Two-Part Format

HANDSHAKE_INIT MUST be structured as two parts:

**Part 1 (Cleartext, 32 bytes)**: Client Ephemeral Public Key (X25519, 32 bytes)

This MUST be cleartext because the receiving node needs the client's ephemeral public key to compute `dh1 = X25519(node_static_priv, client_ephemeral_pub)` before it can derive any key material to decrypt anything.

This is NOT a security problem: the ephemeral public key is not secret (only the private key is secret, which never leaves the client).

**Part 2 (Encrypted, variable)**: All other HANDSHAKE_INIT content, encrypted using `init_key` derived from `dh1`.

Full wire format: see `/protocol/packet_format.md`.

---

## 9. DATA Message

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
| 3-7, 11-15 | Reserved | MUST be 0 |

Congestion flags (bits 8-10) are set by relay nodes and encrypted within the onion layer. They are NOT visible to external observers. They reach the client via the return path.

---

## 10. STREAM Messages

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

---

## 11. KEEPALIVE Message

```
Common Header (Msg Type = 0x09)
(No payload)
```

Nodes MUST send KEEPALIVE messages at 25-second intervals on idle sessions, randomized within ±5 seconds. This randomization is MANDATORY (not optional) to prevent KEEPALIVE timing fingerprinting.

Nodes MUST treat a session as dead if no KEEPALIVE or DATA message is received for 90 seconds and MUST tear down the session.

For sessions over satellite links, the keepalive timeout may be extended (configurable up to 5 minutes) to accommodate high-latency paths.

---

## 12. DHT Messages

See existing Section 11 from original spec for DHT message formats (DHT_FIND_NODE, DHT_FIND_NODE_RESP, DHT_PUT, DHT_GET, DHT_GET_RESP). These are unchanged.

### DHT Storage Authentication Requirement

Storage nodes MUST verify advertisement signatures before accepting a DHT_PUT:

1. Parse the advertisement from the DHT_PUT payload
2. Verify the signature using the advertisement's public_key field
3. Verify node_id = BLAKE3(public_key)
4. Verify timestamps are valid
5. Reject and drop the DHT_PUT silently if any verification fails

Implementations that accept unsigned or invalid records into DHT storage are non-compliant.

---

## 13. NODE_WITHDRAW Message (0x15)

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
1. Verify the signature using the NodeID's public key (fetched from their cached advertisement)
2. If valid: immediately remove the node advertisement from DHT storage
3. Propagate the withdrawal to 5 nearest DHT peers

On receiving NODE_WITHDRAW, routing nodes MUST:
1. Remove the node from their DHT routing table k-buckets
2. Remove the node from their path-eligible node database

Nodes MUST publish NODE_WITHDRAW before graceful shutdown.

---

## 14. PATH_QUALITY_REPORT Message (0x16)

Carries a path quality report from a relay node to the client. This message travels on the return path, encrypted for the client only.

```
Common Header (Msg Type = 0x16)
├───────────────────────────────────────────────────────────────┤
│              Encrypted Report Payload (16 bytes, fixed)       │
│              Encrypted using client's session key for         │
│              this hop                                         │
│                                                               │
│  Decrypted content:                                           │
│    Byte 0: hop_latency_bucket                                 │
│      0x00 = <50ms, 0x01 = 50-150ms, 0x02 = 150-300ms,       │
│      0x03 = >300ms                                            │
│    Byte 1: congestion_flag                                    │
│      0x00 = none, 0x01 = mild, 0x02 = severe                 │
│    Bytes 2-15: padding (random, for size normalization)       │
└───────────────────────────────────────────────────────────────┘
```

PATH_QUALITY_REPORT is always exactly 16 bytes of payload (after encryption tag). It is sent by relay nodes that support the path_quality_reporting extension (extension flag bit 3).

Intermediate relays forward PATH_QUALITY_REPORT opaquely via path_id routing. They do NOT decrypt it.

Only the exit node and the client can generate and consume PATH_QUALITY_REPORT content. This prevents path composition leakage.

---

## 15. MESH_BEACON Message (0x18)

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
│              Timestamp (8 bytes, uint64, Unix epoch)          │
├───────────────────────────────────────────────────────────────┤
│              Signature (64 bytes)                             │
│   Signs: NodeID || PublicKey || TransportsMask ||             │
│           RoutingTableSize || Timestamp                       │
└───────────────────────────────────────────────────────────────┘
```

MESH_BEACON is broadcast on all active mesh transports every 30 seconds.

Receiving nodes MUST verify the signature before adding the sender to their DHT routing table.

---

## 16. MESH_FRAGMENT Message (0x19)

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

## 17. POOL_KEY_ROTATION Message (0x1A)

See `/protocol/pool_curator_key_rotation.md` for full specification. Wire format summary:

```
Common Header (Msg Type = 0x1A, Session ID = 0x00...00)
├───────────────────────────────────────────────────────────────┤
│              Record Length (4 bytes)                          │
├───────────────────────────────────────────────────────────────┤
│              Canonical JSON of rotation record (variable)     │
│              (max 4096 bytes)                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## 18. Node Advertisement Record

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

---

## 19. Replay Protection

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

## 20. MTU and Fragmentation

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

## 21. Protocol State Machine

### 21.1 Node State Machine

```
INITIAL → BOOTSTRAP → ACTIVE → DRAIN → STOP
```

On DRAIN entry: publish NODE_WITHDRAW; wait up to 30 seconds for propagation; then stop accepting new handshakes and drain existing sessions.

### 21.2 Session State Machine (per session)

```
NO SESSION → HANDSHAKING → ESTABLISHED → CLOSED
```

---

## 22. Extension Flags

Extension flags are negotiated during handshake (HANDSHAKE_INIT extensions field, HANDSHAKE_RESP accepted extensions):

| Bit | Extension | v1 |
|---|---|---|
| 0 | Cover traffic support | Yes |
| 1 | Pluggable transport support | Yes |
| 2 | Stream multiplexing | Yes |
| 3 | Path quality reporting | Yes |
| 4 | ECH support (exit nodes) | Yes |
| 5 | Geographic plausibility scoring | Yes |
| 6 | Pool support | Yes |
| 7 | Mesh transport support | Yes |
| 8 | Advertisement withdrawal | Yes |
| 9 | Session re-handshake | Yes |
| 10-31 | Reserved | — |

---

## 23. Compliance Requirements

### MUST

- Implement all message types in Section 4
- Implement common header (Section 5) with bidirectional sequence numbers
- Implement onion encryption scheme from `/protocol/packet_format.md` with path_id field
- Implement handshake from `/protocol/handshake.md` with two-part HANDSHAKE_INIT
- Implement replay protection (Section 19) with separate inbound/outbound windows
- Respect MTU limits (Section 20)
- Implement state machines (Section 21)
- Verify all node advertisement signatures before using or storing
- Enforce geographic and ASN diversity in path selection
- Verify DHT_PUT advertisement signatures before storing (Section 12)
- Implement path_id extraction and path table management
- Implement NODE_WITHDRAW propagation
- Implement POOL_KEY_ROTATION verification
- Implement KEEPALIVE randomization (±5 seconds, Section 11)
- Implement two independent sequence number spaces (Section 5.1)
- Log PROHIBITED items: NEVER log client IPs at relay/exit, NEVER log destinations at relay nodes, NEVER log path_id, NEVER log path composition, NEVER log session IDs, NEVER log traffic volume per path
- For exit nodes: implement DoH DNS resolution with DNSSEC validation and ECS stripping
- For exit nodes: implement ECH when ech_capable=true

### MUST NOT

- Log prohibited items (see above)
- Allow disabling signature verification
- Allow disabling replay protection
- Attempt to decrypt onion layers beyond this node's own layer
- Log relay packet source addresses
- Reuse (session_key, nonce) pairs
- Use a separate cover_key (COVER uses session_key)

### SHOULD

- Support IPv6
- Support cover traffic generation
- Support pluggable transport
- Implement reputation system
- Implement geographic plausibility scoring
- Support pool-based path construction
- Support mesh transport on appropriate hardware
- Implement session re-handshake for sessions exceeding 24 hours
