# BGP-X Packet Format Specification

**Version**: 0.1.0-draft

This document specifies the complete wire encoding of all BGP-X packet types, with focus on the onion packet structure that is core to the data plane.

---

## 1. Encoding Conventions

- All multi-byte integers are big-endian (network byte order)
- All byte sequences are unsigned unless stated otherwise
- Variable-length fields are preceded by a length prefix unless length is determined by Payload Length field
- All lengths are in bytes unless stated otherwise
- Diagrams show bits from most-significant (left) to least-significant (right)

---

## 2. Transport Framing

BGP-X packets are transport datagrams. For UDP/IP transport, each UDP datagram contains exactly one BGP-X message. For mesh transports with small MTU, MESH_FRAGMENT (0x19) handles fragmentation at the transport layer.

```
[UDP Datagram]                  [LoRa Frame]                  [BLE Frame]
  [IP Header]                    [LoRa Header]                  [BLE Header]
  [UDP Header]                   [MESH_FRAGMENT Header]         [MESH_FRAGMENT Header]
  [BGP-X Common Header]          [Fragment Payload]             [Fragment Payload]
  [BGP-X Message Payload]
```

There is no BGP-X framing layer above the transport. Each transport datagram is one self-contained BGP-X message (or one fragment of a message, for low-MTU transports).

---

## 3. Common Header

```
Byte offset:
  0        1        2        3
  ┌────────┬────────┬────────────────┐
  │Version │MsgType │   Reserved     │
  └────────┴────────┴────────────────┘
  4        5        ...             19
  ┌─────────────────────────────────┐
  │          Session ID             │
  │           (16 bytes)            │
  └─────────────────────────────────┘
  20       21       ...            27
  ┌─────────────────────────────────┐
  │     Outbound Sequence Number    │
  │           (8 bytes)             │
  └─────────────────────────────────┘
  28       29       30       31
  ┌─────────────────────────────────┐
  │        Payload Length           │
  │           (4 bytes)             │
  └─────────────────────────────────┘
  32 ...
  ┌─────────────────────────────────┐
  │           Payload               │
  │          (variable)             │
  └─────────────────────────────────┘
```

Total header size: **32 bytes**

Maximum total packet size: **1280 bytes** (path MTU target)

Maximum payload size: **1248 bytes** (1280 - 32 header)

| Field | Size | Description |
|---|---|---|
| Version | 1 byte | Protocol version. Current: 0x01 |
| Msg Type | 1 byte | Message type identifier (see Section 4) |
| Reserved | 2 bytes | MUST be 0x0000 |
| Session ID | 16 bytes | Randomly generated per session. 0x00...00 for pre-session messages |
| Outbound Sequence Number | 8 bytes | Monotonically increasing for OUTBOUND direction only. Separate from inbound sequence space. |
| Payload Length | 4 bytes | Length of the payload in bytes, big-endian |
| Payload | Variable | Message-type-specific content |

---

## 4. N-Hop Unlimited Policy

The BGP-X packet format imposes **no hop count field** in the common header and **no protocol-level hop count limit**. A packet may have any number of onion layers. No counter decrements at each hop. No protocol mechanism drops a packet after N hops.

The only bounds on path length are:
- The packet must fit within 1280 bytes (handled by padding and size classes)
- Path construction feasibility
- Latency budget (application-configurable; no protocol default maximum)

Implementations MUST NOT impose a maximum hop count at the protocol level.

---

## 5. Message Type Table

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

## 6. Onion Layer Hop Types

The hop_type field in the onion layer determines how a node should forward the packet:

| Type ID | Name | Description |
|---|---|---|
| 0x01 | RELAY | Forward to next BGP-X node within current domain |
| 0x02 | EXIT_TCP | Exit to clearnet IPv4 destination |
| 0x03 | EXIT_UDP | Exit to clearnet IPv6 destination |
| 0x04 | EXIT_RESOLVE | Exit to clearnet domain name (resolve at exit) |
| 0x05 | DELIVERY | Deliver to BGP-X native service |
| 0x06 | DOMAIN_BRIDGE | Transition to another routing domain |
| 0x07 | MESH_ENTRY | Enter a mesh island from another domain |
| 0x08 | MESH_EXIT | Exit a mesh island to another domain |
| 0x09 | MESH_RELAY | Relay within a mesh island (signals mesh-transport forwarding) |

---

## 7. Onion Layer Structure

### 7.1 Overview

For a path of N hops, the client constructs N nested encryption layers. The outermost layer is addressed to the entry node. Each hop decrypts its layer to reveal the path_id, next-hop address, and remaining ciphertext for subsequent hops.

```
Client constructs for 4-hop path:
┌─────────────────────────────────────────────────────────────────┐
│  Encrypted for Entry (hop 1)                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Encrypted for Relay 1 (hop 2)                            │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Encrypted for Relay 2 (hop 3)                      │  │  │
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │
│  │  │  │  Encrypted for Exit (hop 4)                   │  │  │  │
│  │  │  │  [hop_type][next_hop][path_id][stream_id]     │  │  │  │
│  │  │  │  [flags][reserved][payload]                   │  │  │  │
│  │  │  └───────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Single Onion Layer Format (After Decryption)

After decryption at a relay node, the onion layer header contains:

```
Byte  0:      hop_type     (1 byte)
Bytes 1-40:   next_hop     (40 bytes) — encoding depends on hop_type (see Section 8)
Bytes 41-48:  path_id      (8 bytes) — return traffic routing identifier
Bytes 49-52:  stream_id    (4 bytes)
Bytes 53-54:  flags        (2 bytes)
Bytes 55-56:  reserved     (2 bytes, MUST be 0x0000)
Bytes 57+:    remaining_ciphertext (for subsequent hops)
```

**Total layer header: 57 bytes**

### 7.3 path_id Field (Bytes 41-48)

The path_id is 8 bytes of cryptographically random data generated by the client per path instance.

Properties:
- Generated using CSPRNG
- Unique per path instance (regenerated on each path build)
- Identical value in EVERY onion layer of the same path including DOMAIN_BRIDGE layers
- Never reused within a session
- Not linked to any client identifier
- Never logged by any BGP-X component
- Used by domain bridge nodes to maintain cross-domain path tables for return routing

All-zeros path_id (0x0000000000000000) is INVALID and MUST be rejected — drop silently, log internally "Invalid path_id: all zeros".

### 7.4 path_id Generation

```
path_id = CSPRNG.generate_bytes(8)

Constraints:
  - MUST be 8 bytes
  - MUST be cryptographically random
  - MUST NOT be all zeros
  - MUST be regenerated for each new path
  - MUST NOT be reused within a session
  - MUST be included identically in all layers of the same path
```

### 7.5 Flags Field (Bytes 53-54)

General flags used across all hop types:

| Bit | Name | Description |
|---|---|---|
| 0 | CONGESTION_MILD | Node queue above 75% capacity (set by relay, encrypted in layer) |
| 1 | CONGESTION_SEVERE | Node queue above 90% capacity |
| 2 | CONGESTION_CLEAR | Congestion resolved |
| 3-15 | Reserved | MUST be 0 |

For DOMAIN_BRIDGE hop type (0x06), additional flags apply:

| Bit | Name | Description |
|---|---|---|
| 0 | ENTER_DOMAIN | This hop enters the target domain |
| 1 | EXIT_DOMAIN | This hop exits the source domain |
| 2 | BIDIRECTIONAL | Bridge maintains path_id in both domains |
| 3 | OFFLINE_CAPABLE | Target domain may be temporarily unreachable; enable store-and-forward |
| 4-15 | Reserved | MUST be 0 |

---

## 8. Next Hop Field Encoding

The Next Hop field is always **40 bytes**. Encoding depends on hop_type.

### 8.1 hop_type 0x01 — RELAY

| Bytes | Content |
|---|---|
| 0-31 | NodeID of next relay (32 bytes, BLAKE3 hash) |
| 32-35 | IPv4 address of next relay |
| 36-37 | UDP port |
| 38-39 | Reserved (0x0000) |

### 8.2 hop_type 0x02 — EXIT_TCP (clearnet IPv4)

| Bytes | Content |
|---|---|
| 0-3 | Destination IPv4 |
| 4-5 | Destination port |
| 6-39 | Zero-padded |

### 8.3 hop_type 0x03 — EXIT_UDP (clearnet IPv6)

| Bytes | Content |
|---|---|
| 0-15 | Destination IPv6 |
| 16-17 | Destination port |
| 18-39 | Zero-padded |

### 8.4 hop_type 0x04 — EXIT_RESOLVE (domain name)

| Bytes | Content |
|---|---|
| 0 | Domain name length (1-38) |
| 1-N | Domain name (ASCII) |
| N+1-37 | Zero-padded |
| 38-39 | Destination port |

### 8.5 hop_type 0x05 — DELIVERY (BGP-X native service)

| Bytes | Content |
|---|---|
| 0-31 | ServiceID (Ed25519 public key, 32 bytes) |
| 32-39 | Zero-padded |

### 8.6 hop_type 0x06 — DOMAIN_BRIDGE

| Bytes | Content |
|---|---|
| 0-7 | Target domain ID (8 bytes) |
| 8-39 | Bridge node NodeID (32 bytes) |

**Domain ID encoding:**
```
Bytes 0-3: Domain type (uint32 BE):
           0x00000001 = clearnet
           0x00000002 = bgpx-overlay
           0x00000003 = mesh
           0x00000004 = lora-regional
           0x00000005 = RESERVED (future BGP-X-native satellite — NOT ACTIVE)

Bytes 4-7: Instance hash (uint32 BE):
           0x00000000 for singletons (clearnet, overlay)
           BLAKE3(instance_string_utf8)[0:4] for named domains (mesh, lora-regional)
```

**Important**: Any DOMAIN_BRIDGE hop with domain type 0x00000005 MUST be rejected as unverifiable.

### 8.7 hop_type 0x07 — MESH_ENTRY

| Bytes | Content |
|---|---|
| 0-7 | Mesh island domain ID (type 0x00000003 + instance hash) |
| 8-39 | First mesh relay NodeID within island |

### 8.8 hop_type 0x08 — MESH_EXIT

| Bytes | Content |
|---|---|
| 0-7 | Destination domain ID (domain to enter after exiting mesh) |
| 8-39 | Exit point NodeID (bridge node with next-domain endpoint) |

### 8.9 hop_type 0x09 — MESH_RELAY

| Bytes | Content |
|---|---|
| 0-31 | NodeID of next mesh relay |
| 32-35 | LoRa address hint (4 bytes; 0x00000000 for WiFi mesh auto-resolve) |
| 36-39 | Reserved |

---

## 9. Encryption of Each Layer

Each layer is encrypted with ChaCha20-Poly1305 using:

```
Key:    session_key[hop_i]                    (32 bytes)
Nonce:  [0x00000000] || sequence_number_BE8   (12 bytes)
AAD:    BGP-X common header (32 bytes)
PT:     Layer Header (57 bytes) || Remaining Ciphertext
CT:     Encrypted plaintext || 16-byte Poly1305 tag
```

The authentication tag is appended to the ciphertext. Decryption MUST verify the tag before any plaintext processing.

**COVER packets (message type 0x11)** use the **same session_key as RELAY packets**. There is no separate cover_key. Both use ChaCha20-Poly1305 encryption and are externally indistinguishable.

---

## 10. Onion Construction Algorithm

```
function construct_onion(payload, path, session_keys, path_id, sequence):

    N = len(path)

    # Start with innermost layer (exit hop)
    current = construct_layer(
        hop_type = exit_type(path[N-1]),
        next_hop = destination,
        path_id  = path_id,          # same path_id in ALL layers
        stream_id = stream_id,
        payload  = payload,
        key      = session_keys[N-1],
        aad      = construct_header(sequence),
        sequence = sequence
    )
    current = pad_to_size_class(current)

    # Wrap each layer from exit-1 toward entry
    for i in range(N-2, -1, -1):
        current = construct_layer(
            hop_type  = hop_type_for_position(i),
            next_hop  = path[i + 1],
            path_id   = path_id,       # same path_id in ALL layers
            stream_id = stream_id,
            payload   = current,
            key       = session_keys[i],
            aad       = construct_header(sequence),
            sequence  = sequence
        )
        current = pad_to_size_class(current)

    return current  # Outermost layer, sent to entry
```

DOMAIN_BRIDGE hops are constructed identically to RELAY hops at the crypto level. The only differences are the hop_type value (0x06) and next_hop encoding (domain_id + bridge_node_id).

---

## 11. Onion Unwrapping Algorithm (at each relay)

```
function unwrap_onion(packet, session_key, sequence_number):

    # Replay check on inbound sequence space
    if not check_inbound_sequence(sequence_number):
        return DROP("Replay detected")

    # Decrypt
    plaintext = ChaCha20-Poly1305-Decrypt(
        key      = session_key,
        nonce    = 0x00000000 || sequence_number_BE8,
        aad      = packet.common_header,
        ciphertext = packet.payload
    )

    if plaintext is None:
        return DROP("Auth failure")

    # Parse layer header
    hop_type   = plaintext[0]
    next_hop   = plaintext[1:41]
    path_id    = plaintext[41:49]
    stream_id  = plaintext[49:53]
    flags      = plaintext[53:55]
    reserved   = plaintext[55:57]
    remaining  = plaintext[57:]

    # Validate path_id
    if path_id == bytes(8):  # All zeros
        return DROP("Invalid path_id")

    # Validate reserved
    if reserved != b'\x00\x00':
        return DROP("Reserved field non-zero")

    # Store path_id → source for return routing
    path_table[path_id] = (packet.source_addr, current_time())

    return (hop_type, next_hop, path_id, stream_id, remaining)
```

---

## 12. HANDSHAKE_INIT Wire Format

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x02)
Bytes 32-63:   Client Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
               Node needs this to compute dh1 before decrypting anything.
               Not a security problem: public key is not secret.
Bytes 64-239:  Encrypted Payload (176 bytes, ChaCha20-Poly1305)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

**AAD for HANDSHAKE_INIT encryption**:
```
AAD = BGP-X Common Header (32 bytes) || Client Ephemeral Public Key (32 bytes)
    = 64 bytes total
```

Both cleartext portions are authenticated even though the first 32 bytes are not encrypted.

Encrypted payload content (160 bytes plaintext):
```
Session ID (16 bytes, CSPRNG)
Timestamp  (8 bytes, uint64 Unix epoch)
Protocol Version (2 bytes)
Extension Flags  (4 bytes)
Padding to 160 bytes
```

**Domain-agnostic**: This wire format is identical regardless of routing domain or transport type. A handshake over UDP/IP, WiFi mesh, LoRa, Bluetooth, or satellite uses the same message bytes.

---

## 13. HANDSHAKE_RESP Wire Format

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x03)
Bytes 32-63:   Node Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
Bytes 64-239:  Encrypted Payload (176 bytes)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

Encrypted payload content:
```
Session ID Echo     (16 bytes, MUST match HANDSHAKE_INIT session_id)
Timestamp           (8 bytes)
Selected Version    (2 bytes)
Accepted Extensions (4 bytes)
Confirmation Hash   (32 bytes)
  BLAKE3("bgpx-resp-confirm" || session_key || session_id)
Padding
```

**Domain-agnostic**: Same format for all domains.

---

## 14. HANDSHAKE_DONE Wire Format

**MUST be exactly 128 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x04)
Bytes 32-111:  Encrypted Payload (80 bytes, using session_key)
Bytes 112-127: Poly1305 Authentication Tag (16 bytes)
```

Encrypted payload:
```
Confirmation Hash  (32 bytes)
  BLAKE3("bgpx-done-confirm" || session_key || session_id)
path_id            (8 bytes) — informs node of its path_id for return routing
Padding to 64 bytes plaintext
```

**For domain bridge nodes**: The path_id in HANDSHAKE_DONE establishes both:
1. The single-domain path table entry (`path_id → predecessor_addr`)
2. The cross-domain path table entry (`path_id → {predecessor_domain, predecessor_addr, target_domain}`)

**Domain-agnostic**: Same format for all domains.

---

## 15. PATH_QUALITY_REPORT Wire Format

The PATH_QUALITY_REPORT payload is **20 bytes** (extended from 16 to include 8-byte domain ID).

```
BGP-X Common Header (32 bytes, Msg Type = 0x16)

ENCRYPTED PAYLOAD (20 bytes plaintext → 20 bytes ciphertext + 16 bytes tag = 36 bytes total):
  Bytes 0-7:   Domain ID of reporting node (8 bytes)
               type (4 bytes BE) + instance (4 bytes BE)
  Byte 8:      hop_latency_bucket
               0x00 = excellent (domain-calibrated)
               0x01 = good
               0x02 = acceptable
               0x03 = degraded
  Byte 9:      congestion_flag
               0x00 = none, 0x01 = mild, 0x02 = severe
  Bytes 10-19: padding (random, for size normalization)

Total PATH_QUALITY_REPORT packet: 32 + 36 = 68 bytes
```

Encryption: `ChaCha20-Poly1305(key=session_key[hop_i], nonce=report_seq, aad=path_id||hop_position_byte, pt=20 bytes)`

**Intermediate relays and bridge nodes forward PATH_QUALITY_REPORT opaquely** via path_id routing — no decryption at any intermediate node, including across domain boundaries. Only the exit node and the client can generate and consume PATH_QUALITY_REPORT content. This prevents path composition leakage.

---

## 16. NODE_WITHDRAW Wire Format

```
BGP-X Common Header (32 bytes, Msg Type = 0x15, Session ID = 0x00...00)
Bytes 32-63:   NodeID (32 bytes)
Bytes 64-71:   Withdraw Timestamp (8 bytes, uint64 Unix epoch)
Bytes 72-135:  Ed25519 Signature (64 bytes)
               Signs: BLAKE3("bgpx-withdrawal-v1" || node_id || timestamp_BE8)

Total payload: 104 bytes
```

On receiving NODE_WITHDRAW, storage nodes MUST:
1. Verify the signature using the NodeID's public key (fetched from cached advertisement)
2. If valid: immediately remove the node advertisement from DHT storage
3. Propagate the withdrawal to 5 nearest DHT peers (propagation limit prevents infinite gossip)

On receiving NODE_WITHDRAW, routing nodes MUST:
1. Remove node from DHT routing table k-buckets
2. Remove node from path-eligible node database

For domain bridge nodes: after NODE_WITHDRAW, also remove from bridge pair records.

---

## 17. MESH_BEACON Wire Format

Broadcast by mesh nodes to announce their presence and enable bootstrap without internet connectivity.

```
BGP-X Common Header (Msg Type = 0x18, Session ID = 0x00...00)
Bytes 32-63:   NodeID (32 bytes)
Bytes 64-95:   Ed25519 Public Key (32 bytes)
Byte  96:      Transports Supported (1 byte bitmask)
               Bit 0: UDP/IP  Bit 1: WiFi mesh  Bit 2: LoRa
               Bit 3: BLE     Bit 4: Ethernet P2P
Bytes 97-98:   DHT Routing Table Size (2 bytes, uint16)
Byte  99:      bridge_capable (1 byte): 0x00 = no, 0x01 = yes
Byte  100:     served_domains_count (1 byte)
Bytes 101-124: served_domains (served_domains_count × 8 bytes)
               Each domain ID: type (4B) + instance hash (4B)
               Zero-padded if fewer than 3 domains
Bytes 125-132: Timestamp (8 bytes, uint64 Unix epoch)
Bytes 133-196: Ed25519 Signature (64 bytes)
               Signs: NodeID || PublicKey || TransportsMask ||
                      RoutingTableSize || bridge_capable ||
                      served_domains_count || served_domains ||
                      Timestamp

Total payload: 197 bytes
```

MESH_BEACON is broadcast on all active mesh transports every 30 seconds, with ±5 seconds jitter.

Receiving nodes MUST verify the signature before adding the sender to their DHT routing table.

The `bridge_capable` and `served_domains` fields allow mesh nodes to discover cross-domain bridge nodes without internet DHT access.

---

## 18. MESH_FRAGMENT Wire Format

For mesh transports with small MTU (particularly LoRa), BGP-X packets that exceed the transport MTU are fragmented at the transport layer.

```
BGP-X Common Header (Msg Type = 0x19, Session ID = 0x00...00)
Bytes 32-35:   Original Packet ID (4 bytes, random per original packet)
Byte  36:      Fragment Number (1 byte, 0-indexed)
Byte  37:      Total Fragments (1 byte, 1-16)
Bytes 38+:     Fragment Payload (variable)

Fragment overhead: 6 bytes beyond common header
```

- Fragmentation occurs BEFORE BGP-X header construction for the original packet
- Maximum 16 fragments per packet
- Fragment timeout: 10 seconds — if not all fragments received, discard and log
- Reassembly produces a complete BGP-X packet for normal processing
- packet_id is 4 bytes random per original packet (unique for reassembly tracking)

**MTU limits by transport:**

| Transport | MTU Available | Notes |
|---|---|---|
| UDP/IP | 1280 bytes | BGP-X target MTU |
| WiFi 802.11s | 1280 bytes | Same as UDP/IP |
| LoRa SF7 | ~200 bytes usable | Requires MESH_FRAGMENT |
| LoRa SF12 | ~50 bytes usable | Requires MESH_FRAGMENT |
| Bluetooth BLE | 100-200 bytes | Requires MESH_FRAGMENT |
| Ethernet P2P | 1500 bytes | BGP-X target MTU still applies |

---

## 19. DOMAIN_ADVERTISE Message (0x1A)

Nodes with bridge capability publish domain bridge records to the unified DHT.

```
BGP-X Common Header (32 bytes, Msg Type = 0x1A, Session ID = 0x00...00)
Record length (4 bytes, uint32 BE)
Canonical JSON of domain bridge advertisement (UTF-8, max 4096 bytes)
```

Domain bridge advertisement JSON:
```json
{
  "version": 1,
  "node_id": "<NodeID hex>",
  "bridge_pairs": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "bridge_latency_ms": 25,
      "transport": "wifi_mesh",
      "available": true,
      "geo_plausibility_exempt": false
    }
  ],
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature>"
}
```

**DHT storage key:**
```
key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id, to_domain_id) || max(from_domain_id, to_domain_id))
TTL: 24 hours. Re-publication: every 8 hours.

The min/max ordering makes the key symmetric: querying A→B and B→A produces the same key.

**Storage node verification**:
- MUST verify the signature using the node's Ed25519 public key
- MUST verify the node has valid endpoints in BOTH domains it claims to bridge
- MUST verify bridge_latency_ms is within plausible range
- MUST reject and drop silently if any verification fails

---

## 20. MESH_ISLAND_ADVERTISE Message (0x1B)

Mesh island gateway nodes publish island advertisements to the unified DHT when internet-connected.

```
BGP-X Common Header (32 bytes, Msg Type = 0x1B, Session ID = 0x00...00)
Record length (4 bytes, uint32 BE)
Canonical JSON of mesh island advertisement (UTF-8, max 8192 bytes)
```

Mesh island advertisement JSON:
```json
{
  "version": 1,
  "island_id": "lima-district-1",
  "domain_id": "0x00000003-a1b2c3d4",
  "node_count": 12,
  "active_relay_count": 8,
  "bridge_nodes": [
    {
      "node_id": "<NodeID hex>",
      "bridges": ["clearnet", "mesh:lima-district-1"],
      "clearnet_endpoint": "203.0.113.1:7474"
    }
  ],
  "services": [
    { "service_id": "<ServiceID hex>", "name": "community-forum", "advertise": true }
  ],
  "dht_coverage": "full",
  "offline_capable": true,
  "signed_at": "2026-04-24T12:00:00Z",
  "curator_public_key": "<base64url Ed25519 public key>",
  "signature": "<base64url Ed25519 signature>"
}
```

**DHT storage key:**
```
key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8_bytes)
TTL: 24 hours. Re-publication: every 8 hours.
```

When no bridge node with internet is available, the island advertisement exists only in the local mesh DHT. When a bridge node reconnects, the island advertisement is published to the unified internet DHT automatically.

---

## 21. POOL_KEY_ROTATION Message (0x1C)

Pool curator key rotation records are stored in the unified DHT.

```
BGP-X Common Header (32 bytes, Msg Type = 0x1C, Session ID = 0x00...00)
Record length (4 bytes, uint32 BE)
Canonical JSON of rotation record (UTF-8, max 4096 bytes)
```

Rotation record JSON:
```json
{
  "version": 1,
  "pool_id": "<32-byte pool ID hex>",
  "rotation_reason": "scheduled",
  "old_curator_public_key": "<base64url>",
  "new_curator_public_key": "<base64url>",
  "new_pool_advertisement_signature": "<base64url>",
  "old_key_signature": "<base64url>",
  "new_key_signature": "<base64url>",
  "effective_at": "2026-04-24T12:00:00Z",
  "accept_until": "2026-04-25T12:00:00Z",
  "grace_period_hours": 24,
  "signed_at": "2026-04-24T11:00:00Z"
}
```

**DHT storage key:**
```
key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || rotation_timestamp_BE8)
TTL: accept_until + 7 days (retained for audit)
```

See `/protocol/pool_curator_key_rotation.md` for full specification.

---

## 22. Size Class Padding

All RELAY, DOMAIN_BRIDGE, MESH_RELAY, and COVER packets SHOULD be padded to normalized size classes to resist traffic analysis.

| Size Class | Total UDP Payload | Use Case |
|---|---|---|
| Small | 256 bytes | Low-bandwidth paths, LoRa |
| Medium | 512 bytes | Standard paths |
| Large | 1024 bytes | High-throughput paths |
| Maximum | 1248 bytes | Maximum payload efficiency |

**Padding structure:**
```
[content][random_padding][padding_length: uint16 BE]
```

The last 2 bytes contain the padding length, allowing the receiver to strip padding efficiently.

**COVER packet requirements**:
- MUST use the same size class distribution as RELAY packets on the same session
- MUST use session_key (same as RELAY) — no separate cover_key
- Externally indistinguishable from RELAY in all routing domains

**Domain-agnostic padding**: Padding rules apply identically in all routing domains — clearnet, mesh, and satellite-class clearnet use the same size classes.

---

## 23. Cross-Domain Onion Construction Algorithm

The client constructs one onion layer per hop regardless of domain. DOMAIN_BRIDGE hops are constructed identically to RELAY hops from the crypto perspective.

```python
function construct_cross_domain_onion(all_hops, session_keys, payload, path_id, sequence):
    # all_hops: ordered list of (hop_type, next_hop_40_bytes) tuples
    # Client constructs one onion layer per hop regardless of domain
    # DOMAIN_BRIDGE hops are constructed identically to RELAY hops from crypto perspective

    N = len(all_hops)

    # Start with innermost layer (last hop)
    current = construct_layer(
        hop_type  = all_hops[N-1].type,
        next_hop  = all_hops[N-1].encoding,
        path_id   = path_id,     # same path_id in ALL layers
        stream_id = stream_id,
        payload   = payload,
        key       = session_keys[N-1],
        sequence  = sequence
    )
    current = pad_to_size_class(current)

    # Wrap from N-2 toward 0 (outermost)
    for i in range(N-2, -1, -1):
        current = construct_layer(
            hop_type  = all_hops[i].type,
            next_hop  = all_hops[i].encoding,
            path_id   = path_id,     # same path_id in ALL layers
            stream_id = stream_id,
            payload   = current,
            key       = session_keys[i],
            sequence  = sequence
        )
        current = pad_to_size_class(current)

    return current  # Outermost layer, sent to first hop
```

**DOMAIN_BRIDGE hop construction** is identical to RELAY hop construction at the crypto level. The only differences are:
- hop_type value = 0x06
- next_hop encoding = domain_id (8 bytes) + bridge_node_id (32 bytes)

---

## 24. Bidirectional Sequence Numbers

Each session maintains TWO independent 64-bit sequence number spaces:

- **Outbound**: Sequence numbers for packets sent by this node to the next hop
- **Inbound**: Sequence numbers for packets received from the previous hop

**Common header carries outbound sequence number.**

**Replay window tracks inbound sequence space separately.**

This separation prevents nonce reuse even though the same session key is used for both directions.

### 24.1 Outbound Sequence Numbers

- Monotonically increasing uint64
- Stored in common header bytes 20-27
- Each outbound packet increments the counter
- Used as nonce input for encryption

### 24.2 Inbound Sequence Numbers

- Tracked in 64-position sliding window per session
- Not included in any packet
- Used for replay detection on received packets
- Separate from outbound space

### 24.3 Sliding Window Implementation

```
Initial: inbound_base = 0, inbound_window = 0 (64-bit bitmap)

On packet receipt with sequence S:
  if S > inbound_base + 63:
    # New sequence beyond window
    shift = S - (inbound_base + 63)
    inbound_window >>= shift
    inbound_base += shift
    mark bit (S - inbound_base) as received
    return ACCEPT
  elif S < inbound_base:
    # Too old
    return REJECT("Sequence too old")
  elif bit (S - inbound_base) already set:
    return REJECT("Replay detected")
  else:
    mark bit (S - inbound_base) as received
    return ACCEPT
```

---

## 25. HTTP/2 over BGP-X Streams

BGP-X native services (.bgpx) use HTTP/2 over BGP-X streams.

**Why HTTP/2, not HTTP/3:**
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

**HTTP/3 at exit nodes**: When an exit node connects to an HTTP/3 clearnet server, it uses standard HTTP/3 over TLS over the exit's clearnet connection — not over BGP-X streams.

**LoRa path optimization**: For LoRa paths, HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## 26. Geographic Plausibility — OPTIONAL Feature

Geographic plausibility scoring is an OPTIONAL reputation signal. It is NOT required for BGP-X operation.

### 26.1 When Geo Plausibility Applies

- IF a node declares a jurisdiction in its advertisement: geo plausibility scoring applies
- IF a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

Nodes are NOT required to declare a jurisdiction. Declaring jurisdiction is an opt-in privacy/convenience tradeoff.

### 26.2 Exemptions

The following node types are EXEMPT from geo plausibility scoring (always return neutral score 0.5):

- Satellite-class clearnet nodes (`latency_class = satellite-*`)
- Mesh nodes without declared jurisdiction
- Nodes in domains without internet RTT calibration (pure mesh islands without clearnet bridge)

### 26.3 Behavior in Path Construction

A node with poor geo plausibility score:

- Receives lower selection probability (scoring penalty)
- Can still be selected in a path (not hard excluded)
- Persistent implausibility may lead to reputation penalties through normal reputation system mechanisms

Geo plausibility is a reputation signal, not a mandatory filtering criterion.

---

## 27. Satellite Internet Clarification

**Commercial satellite internet services are clearnet domain (0x00000001).**

This includes:
- Starlink (LEO, 20-40ms RTT)
- Iridium Certus (LEO, 150-300ms RTT)
- Inmarsat (GEO, 600ms+ RTT)
- HughesNet (GEO, 600ms+ RTT)
- Viasat (GEO, 600ms+ RTT)
- OneWeb (LEO, 30-50ms RTT)
- Kuiper (LEO, 20-40ms RTT)

From BGP-X's perspective, a satellite-connected node is a clearnet node with high latency_class annotation. The physical medium (fiber vs. satellite radio) is invisible to the BGP-X protocol layer.

**Domain type 0x00000005 (bgpx-satellite)** is RESERVED for future BGP-X-native satellite infrastructure where:
- Satellites themselves run BGP-X relay software
- Inter-satellite links carry BGP-X packets directly
- No ground station is required for BGP-X traffic

This is NOT currently active. No commercial satellite service provides this capability. Any node advertising domain type 0x00000005 in current deployments MUST be rejected as unverifiable.

---

## 28. Test Vectors

Test vectors for BGP-X packet format verification will be published in `/protocol/test_vectors/` during reference implementation.

Required test vector categories:

### 28.1 Core Onion Layer Vectors

1. **Onion layer construction (3-hop path)**: 5 vectors with varying payloads
2. **Onion layer construction (10-hop path)**: 3 vectors
3. **Onion layer construction (15-hop path)**: 3 vectors (N-hop unlimited verification)
4. **Onion layer construction (20-hop path)**: 3 vectors (N-hop unlimited verification)

### 28.2 Hop Type Vectors

5. **RELAY (0x01)**: 5 vectors with varying next_hop encodings
6. **EXIT_TCP (0x02)**: 3 vectors
7. **EXIT_UDP (0x03)**: 3 vectors
8. **EXIT_RESOLVE (0x04)**: 3 vectors with varying domain names
9. **DELIVERY (0x05)**: 3 vectors
10. **DOMAIN_BRIDGE (0x06)**: 5 vectors (clearnet→mesh, mesh→clearnet, mesh→mesh, clearnet→satellite-reserved, three-segment path)
11. **MESH_ENTRY (0x07)**: 3 vectors
12. **MESH_EXIT (0x08)**: 3 vectors
13. **MESH_RELAY (0x09)**: 3 vectors

### 28.3 Cross-Domain Specific Vectors

14. **DOMAIN_BRIDGE onion layer construction**: 5 vectors
15. **DOMAIN_BRIDGE onion unwrapping at bridge node**: 5 matching vectors
16. **Cross-domain path_id inclusion in all layers**: 5 vectors (same path_id at every hop)

### 28.4 Path Quality Reporting

17. **PATH_QUALITY_REPORT with domain ID (clearnet)**: 3 vectors
18. **PATH_QUALITY_REPORT with domain ID (mesh)**: 3 vectors
19. **PATH_QUALITY_REPORT with domain ID (lora-regional)**: 3 vectors

### 28.5 Advertisement Messages

20. **DOMAIN_ADVERTISE sign and verify**: 3 vectors
21. **MESH_ISLAND_ADVERTISE sign and verify**: 3 vectors
22. **MESH_BEACON verification**: 3 vectors

### 28.6 Handshake Vectors

23. **HANDSHAKE_INIT two-part format**: 5 vectors
24. **HANDSHAKE_RESP encryption**: 5 vectors
25. **HANDSHAKE_DONE with path_id**: 5 vectors

### 28.7 Withdrawal and Rotation

26. **NODE_WITHDRAW signature verification**: 3 vectors
27. **POOL_KEY_ROTATION dual-signature verification**: 3 vectors

### 28.8 Mesh Fragmentation

28. **MESH_FRAGMENT reassembly (2 fragments)**: 3 vectors
29. **MESH_FRAGMENT reassembly (5 fragments)**: 3 vectors
30. **MESH_FRAGMENT reassembly (10 fragments)**: 3 vectors

### 28.9 Negative Test Vectors

31. **Invalid domain_id type (0x00000005 - reserved)**: 2 vectors (must reject)
32. **DOMAIN_BRIDGE with all-zeros domain_id**: 2 vectors (must reject)
33. **path_id = 0x00...00 (all zeros)**: 2 vectors (must drop silently)
34. **Malformed onion layer (too short)**: 2 vectors (must drop silently)
35. **DOMAIN_BRIDGE hop at non-bridge node**: 2 vectors (must drop silently)
36. **Invalid DOMAIN_ADVERTISE signature**: 2 vectors (must reject)
37. **Invalid MESH_ISLAND_ADVERTISE signature**: 2 vectors (must reject)
38. **Replay detection edge cases**: 3 vectors (boundary of sliding window)

### 28.10 Cover Traffic Vectors

39. **COVER packet indistinguishability**: 3 vectors (COVER and RELAY from same session; verify identical ciphertext size distribution)
40. **COVER with same session_key as RELAY**: 3 vectors

---

## 29. Compliance Requirements

### 29.1 MUST

- Implement common header (Section 3) with bidirectional sequence numbers
- Implement onion encryption scheme with path_id field in all layers
- Implement DOMAIN_BRIDGE (0x06) forwarding for bridge-capable nodes
- Maintain cross-domain path_id routing table for bridge-capable nodes
- Implement NODE_WITHDRAW propagation
- Implement POOL_KEY_ROTATION verification
- Implement MESH_FRAGMENT for low-MTU transport support
- Support domain-filtered DHT_FIND_NODE
- Enforce N-hop unlimited: MUST NOT impose a maximum hop count at the protocol level
- Participate in one unified DHT (MUST NOT maintain separate per-domain DHTs)
- Use session_key for COVER (no separate cover_key)
- Never log path_id, path composition, or client IPs at relay nodes

### 29.2 MUST NOT

- Accept DOMAIN_BRIDGE hops with domain type 0x00000005 (reserved)
- Accept path_id = 0x00...00 (all zeros)
- Re-encrypt onion payload when transitioning between routing domains at bridge nodes
- Use a separate cover_key
- Enforce a maximum hop count at the protocol level
- Maintain separate per-routing-domain DHTs
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

### 29.3 SHOULD

- Implement geographic plausibility scoring with domain-specific thresholds
- Support DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE message types
- Support cross-domain path construction
- Maintain bridge latency measurements per bridge pair
- Verify domain bridge advertisements against node advertisement routing_domains
- Pad packets to size classes (Section 22)
