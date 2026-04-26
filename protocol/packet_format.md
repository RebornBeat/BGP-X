# BGP-X Packet Format Specification

**Version**: 0.1.0-draft

---

## 1. Encoding Conventions

- All multi-byte integers are big-endian (network byte order)
- All byte sequences are unsigned unless stated otherwise
- Variable-length fields are preceded by a length prefix unless length is determined by Payload Length field
- All lengths are in bytes unless stated otherwise

---

## 2. Transport Framing

BGP-X packets are transport datagrams. For UDP/IP transport, each UDP datagram contains exactly one BGP-X message. For mesh transports with small MTU, MESH_FRAGMENT (0x19) handles fragmentation at the transport layer.

```
[UDP Datagram]                  [LoRa Frame]
  [IP Header]                    [LoRa Header]
  [UDP Header]                   [MESH_FRAGMENT Header]
  [BGP-X Common Header]          [Fragment Payload]
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

Total header size: **32 bytes**. Maximum total packet size: **1280 bytes**. Maximum payload: **1248 bytes**.

| Field | Size | Description |
|---|---|---|
| Version | 1 byte | Protocol version. Current: 0x01 |
| Msg Type | 1 byte | Message type identifier |
| Reserved | 2 bytes | MUST be 0x0000 |
| Session ID | 16 bytes | Random per session. 0x00...00 for pre-session messages |
| Outbound Sequence Number | 8 bytes | Monotonically increasing per outbound direction |
| Payload Length | 4 bytes | Length of payload in bytes, big-endian |

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

| Type ID | Name | Description |
|---|---|---|
| 0x01 | RELAY | Forward an onion-encrypted payload within current domain |
| 0x02 | HANDSHAKE_INIT | Initiate session key exchange |
| 0x03 | HANDSHAKE_RESP | Respond to key exchange |
| 0x04 | HANDSHAKE_DONE | Complete key exchange |
| 0x05 | DATA | Encrypted application data |
| 0x06 | STREAM_OPEN | Open a new multiplexed stream |
| 0x07 | STREAM_CLOSE | Close a multiplexed stream |
| 0x08 | STREAM_DATA | Data on a specific stream |
| 0x09 | KEEPALIVE | Maintain session liveness |
| 0x0A | NODE_ADVERTISE | Publish node advertisement to DHT |
| 0x0B | DHT_FIND_NODE | Locate nodes near a target ID |
| 0x0C | DHT_FIND_NODE_RESP | Response to DHT_FIND_NODE |
| 0x0D | DHT_GET | Retrieve a DHT record |
| 0x0E | DHT_GET_RESP | Response to DHT_GET |
| 0x0F | DHT_PUT | Store a DHT record |
| 0x10 | ERROR | Signal a protocol error |
| 0x11 | COVER | Cover traffic (externally indistinguishable from RELAY) |
| 0x12 | DHT_POOL_ADVERTISE | Publish pool advertisement |
| 0x13 | DHT_POOL_GET | Retrieve a pool advertisement |
| 0x14 | DHT_POOL_GET_RESP | Response to DHT_POOL_GET |
| 0x15 | NODE_WITHDRAW | Signal node withdrawal |
| 0x16 | PATH_QUALITY_REPORT | Report path quality with domain ID (encrypted for client) |
| 0x17 | PT_HANDSHAKE | Pluggable transport handshake |
| 0x18 | MESH_BEACON | Mesh network presence announcement |
| 0x19 | MESH_FRAGMENT | Fragment of larger BGP-X packet (low-MTU transports) |
| 0x1A | DOMAIN_ADVERTISE | Node announces routing domain endpoints and bridge capability |
| 0x1B | MESH_ISLAND_ADVERTISE | Mesh island announces existence, bridges, and services |
| 0x1C | POOL_KEY_ROTATION | Pool curator key rotation record |

---

## 6. Onion Layer Hop Types

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

**Total layer header: 57 bytes.**

---

## 8. Next Hop Field Encoding

The Next Hop field is always **40 bytes**. Encoding depends on hop_type:

### hop_type 0x01 — RELAY

| Bytes | Content |
|---|---|
| 0-31 | NodeID of next relay (32 bytes) |
| 32-35 | IPv4 address of next relay |
| 36-37 | UDP port |
| 38-39 | Reserved (0x0000) |

### hop_type 0x02 — EXIT_TCP (clearnet IPv4)

| Bytes | Content |
|---|---|
| 0-3 | Destination IPv4 |
| 4-5 | Destination port |
| 6-39 | Zero-padded |

### hop_type 0x03 — EXIT_UDP (clearnet IPv6)

| Bytes | Content |
|---|---|
| 0-15 | Destination IPv6 |
| 16-17 | Destination port |
| 18-39 | Zero-padded |

### hop_type 0x04 — EXIT_RESOLVE (domain name)

| Bytes | Content |
|---|---|
| 0 | Domain name length (1-38) |
| 1-N | Domain name (ASCII) |
| N+1-37 | Zero-padded |
| 38-39 | Destination port |

### hop_type 0x05 — DELIVERY (BGP-X native service)

| Bytes | Content |
|---|---|
| 0-31 | ServiceID (Ed25519 public key, 32 bytes) |
| 32-39 | Zero-padded |

### hop_type 0x06 — DOMAIN_BRIDGE

| Bytes | Content |
|---|---|
| 0-7 | Target domain ID (8 bytes: type uint32 BE + instance uint32 BE) |
| 8-39 | Bridge node NodeID (32 bytes) |

Domain ID encoding:
```
Bytes 0-3: Domain type (uint32 BE):
           0x00000001 = clearnet
           0x00000002 = bgpx-overlay
           0x00000003 = mesh
           0x00000004 = lora-regional
           0x00000005 = satellite

Bytes 4-7: Instance hash (uint32 BE):
           0x00000000 for singletons (clearnet, overlay)
           BLAKE3(island_id_utf8)[0:4] for named domains (mesh, lora, satellite)
```

Flags for DOMAIN_BRIDGE:
```
Bit 0: ENTER_DOMAIN — this hop enters the target domain
Bit 1: EXIT_DOMAIN  — this hop exits the source domain
Bit 2: BIDIRECTIONAL — bridge maintains path_id in both domains
Bit 3: OFFLINE_CAPABLE — target domain may be temporarily unreachable; enable store-and-forward
Bits 4-15: Reserved
```

### hop_type 0x07 — MESH_ENTRY

| Bytes | Content |
|---|---|
| 0-7 | Mesh island domain ID (type 0x00000003 + instance) |
| 8-39 | First mesh relay NodeID within island |

### hop_type 0x08 — MESH_EXIT

| Bytes | Content |
|---|---|
| 0-7 | Destination domain ID (domain to enter after exiting mesh) |
| 8-39 | Exit point NodeID (bridge node with next-domain endpoint) |

### hop_type 0x09 — MESH_RELAY

| Bytes | Content |
|---|---|
| 0-31 | NodeID of next mesh relay |
| 32-35 | LoRa address hint (4 bytes; 0x00000000 for WiFi mesh auto-resolve) |
| 36-39 | Reserved |

---

## 9. path_id Field

The path_id (bytes 41-48 of the onion layer header) is:

- 8 bytes of cryptographically random data generated by the client per path
- Unique per path instance (regenerated on each path build)
- Identical value in EVERY onion layer of the same path including DOMAIN_BRIDGE layers
- Never all-zeros (0x0000000000000000 is invalid — drop silently)
- Not linked to any client identifier
- Never logged by any BGP-X component
- Used by domain bridge nodes to maintain cross-domain path tables for return routing

All-zeros path_id: node drops silently, logs internally "Invalid path_id: all zeros".

---

## 10. DOMAIN_ADVERTISE Message (0x1A)

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
      "available": true
    }
  ],
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature>"
}
```

DHT storage key:
```
key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id_8B, to_domain_id_8B) || max(from_domain_id_8B, to_domain_id_8B))
```

The min/max ordering makes the key symmetric: querying A→B and B→A produces the same key.

TTL: 24 hours. Re-publication: every 8 hours.

---

## 11. MESH_ISLAND_ADVERTISE Message (0x1B)

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
      "node_id": "hex...",
      "bridges": ["clearnet", "mesh:lima-district-1"],
      "clearnet_endpoint": "203.0.113.1:7474"
    }
  ],
  "services": [
    { "service_id": "hex...", "name": "community-forum", "advertise": true }
  ],
  "dht_coverage": "full",
  "offline_capable": true,
  "signed_at": "2026-04-24T12:00:00Z",
  "curator_public_key": "...",
  "signature": "..."
}
```

DHT storage key:
```
key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8_bytes)
TTL: 24 hours. Re-publication: every 8 hours.
```

---

## 12. PATH_QUALITY_REPORT Wire Format (Updated)

The PATH_QUALITY_REPORT payload is now **20 bytes** (extended from 16 to include 8-byte domain ID).

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

Intermediate relays forward PATH_QUALITY_REPORT opaquely via path_id routing — no decryption, including across domain boundaries at bridge nodes.

---

## 13. HANDSHAKE_INIT Wire Format

**Domain-agnostic — identical regardless of routing domain or transport type.**

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x02)
Bytes 32-63:   Client Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
               Node needs this to compute dh1 before decrypting anything.
               Not a security problem: public key is not secret.
Bytes 64-239:  Encrypted Payload (176 bytes, ChaCha20-Poly1305)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

AAD: bytes 0-63 (header + cleartext pubkey). Both authenticated even though first not encrypted.

Encrypted payload content (160 bytes plaintext):
```
Session ID (16 bytes, CSPRNG)
Timestamp  (8 bytes, uint64 Unix epoch)
Protocol Version (2 bytes)
Extension Flags  (4 bytes)
Padding to 160 bytes
```

---

## 14. HANDSHAKE_RESP Wire Format

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x03)
Bytes 32-63:   Node Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
Bytes 64-239:  Encrypted Payload (176 bytes)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

---

## 15. HANDSHAKE_DONE Wire Format

**MUST be exactly 128 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x04)
Bytes 32-111:  Encrypted Payload (80 bytes, using session_key)
Bytes 112-127: Poly1305 Authentication Tag (16 bytes)
```

Encrypted payload:
```
Confirmation Hash  (32 bytes) = BLAKE3("bgpx-done-confirm" || session_key || session_id)
path_id            (8 bytes)  — informs node of its path_id for return routing
Padding to 64 bytes plaintext
```

For domain bridge nodes: path_id in HANDSHAKE_DONE establishes both the single-domain path table entry AND the cross-domain path table entry.

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

---

## 17. MESH_BEACON Wire Format

```
BGP-X Common Header (32 bytes, Msg Type = 0x18, Session ID = 0x00...00)
Bytes 32-63:   NodeID (32 bytes)
Bytes 64-95:   Ed25519 Public Key (32 bytes)
Byte  96:      Transports Supported (1 byte bitmask)
               Bit 0: UDP/IP  Bit 1: WiFi mesh  Bit 2: LoRa  Bit 3: BLE  Bit 4: Ethernet P2P
Bytes 97-98:   DHT Routing Table Size (2 bytes, uint16)
Bytes 99-106:  Timestamp (8 bytes, uint64 Unix epoch)
Byte  107:     Bridge capable flag (1 byte: 0x01 = bridge capable, 0x00 = not)
Byte  108:     Served domain count (1 byte, 0-3)
Bytes 109-132: Served domain IDs (3 × 8 bytes = 24 bytes; zero-padded if fewer)
Bytes 133-196: Ed25519 Signature (64 bytes)
               Signs: NodeID || PublicKey || TransportsMask || RoutingTableSize ||
                      Timestamp || BridgeCapable || ServedDomainCount || ServedDomainIDs
Total payload: 197 bytes
```

---

## 18. MESH_FRAGMENT Wire Format

```
BGP-X Common Header (32 bytes, Msg Type = 0x19, Session ID = 0x00...00)
Bytes 32-35:   Original Packet ID (4 bytes, random per original packet)
Byte  36:      Fragment Number (0-indexed)
Byte  37:      Total Fragments (1-16)
Bytes 38+:     Fragment Payload (variable)
Fragment overhead: 6 bytes beyond common header
```

MTU limits by transport:
- LoRa SF7: ~200 bytes usable payload per fragment
- LoRa SF12: ~50 bytes usable
- Bluetooth BLE: ~100-200 bytes

---

## 19. Cross-Domain Onion Construction Algorithm

```python
function construct_cross_domain_onion(all_hops, session_keys, payload, path_id, sequence):
    # all_hops: ordered list of (hop_type, next_hop_40_bytes) tuples
    # Client constructs one onion layer per hop regardless of domain
    # DOMAIN_BRIDGE hops are constructed identically to RELAY hops from the crypto perspective

    N = len(all_hops)

    # Start with innermost layer (last hop)
    current = construct_layer(
        hop_type  = all_hops[N-1].type,
        next_hop  = all_hops[N-1].encoding,
        path_id   = path_id,     # same path_id in ALL layers
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
            path_id   = path_id,
            payload   = current,
            key       = session_keys[i],
            sequence  = sequence
        )
        current = pad_to_size_class(current)

    return current  # Outermost layer, sent to first hop
```

DOMAIN_BRIDGE hops are constructed identically to RELAY hops at the crypto level. The only differences are the hop_type value (0x06) and next_hop encoding (domain_id + bridge_node_id).

---

## 20. Size Class Padding

All RELAY, DOMAIN_BRIDGE, MESH_RELAY, and COVER packets SHOULD be padded:

| Size Class | Total UDP Payload |
|---|---|
| Small | 256 bytes |
| Medium | 512 bytes |
| Large | 1024 bytes |
| Maximum | 1280 bytes |

COVER packets MUST use the same size class distribution as RELAY packets on the same session in all routing domains. Padding structure: `[content][random_padding][padding_length: uint16]`. Last 2 bytes = padding_length for receiver to strip.

---

## 21. Bidirectional Sequence Numbers

Each session maintains TWO independent 64-bit sequence number spaces:

- **Outbound**: sequence numbers for packets sent by this node (in common header)
- **Inbound**: sequence numbers for packets received from the peer (tracked in sliding window)

This separation prevents nonce reuse even with identical session key used in both directions. Domain-agnostic — same mechanism applies in clearnet, mesh, and satellite sessions.

---

## 22. Test Vectors

Test vectors will be published in `/protocol/test_vectors/` during reference implementation. Cross-domain-specific vectors required:

1. DOMAIN_BRIDGE onion layer construction: 5 vectors (clearnet→mesh, mesh→clearnet, mesh→mesh, clearnet→satellite, three-segment path)
2. DOMAIN_BRIDGE onion unwrapping at bridge node: 5 matching vectors
3. MESH_ENTRY/MESH_EXIT/MESH_RELAY layer encoding: 3 vectors each
4. Cross-domain path_id inclusion in all layers: 5 vectors
5. PATH_QUALITY_REPORT with domain ID: 3 vectors
6. DOMAIN_ADVERTISE sign and verify: 3 vectors
7. MESH_ISLAND_ADVERTISE sign and verify: 3 vectors
8. N-hop unlimited (10, 15, 20 hop paths): 3 vectors each
9. Negative — invalid domain_id type: 2 vectors (must drop silently)
10. Negative — DOMAIN_BRIDGE with all-zeros domain_id: 2 vectors (must reject)
