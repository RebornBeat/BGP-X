# BGP-X Protocol Test Vectors

**Version**: 0.1.0-draft

This directory will contain machine-readable test vectors for all BGP-X protocol operations, published during the reference implementation phase.

---

## Compliance Requirement

All BGP-X implementations MUST pass all published test vectors before being considered specification-compliant.

---

## Test Vector Categories

### Cryptographic Primitives

- X25519 key exchange: 10 vectors (RFC 7748)
- Ed25519 sign + verify: 10 vectors (RFC 8032)
- ChaCha20-Poly1305: 10 vectors (RFC 8439)
- BLAKE3: 10 vectors (BLAKE3 specification)
- HKDF-SHA256: 10 vectors (RFC 5869)
- All primitives MUST produce identical results in clearnet, mesh, and satellite domain contexts (domain-agnostic verification)

### BGP-X Session Key Derivation

- Session key derivation (dh1 || dh2 → session_key): 5 vectors
- Keepalive key derivation (separate from session_key): 5 vectors
- Init key derivation (from dh1 only): 5 vectors
- Response key derivation (from dh1 XOR dh2): 5 vectors
- Confirmation hash (HANDSHAKE_RESP): 5 vectors
- Confirmation hash (HANDSHAKE_DONE): 5 vectors

### Onion Layer Operations (Single Domain)

- Onion construction (3-hop): 3 vectors — verify correct path_id placement at bytes 41-48
- Onion construction (4-hop): 5 vectors
- Onion construction (5-hop): 3 vectors
- Onion layer decryption per hop: vectors matching construction vectors
- path_id extraction: 10 vectors
- Size class padding and unpadding: 5 vectors per size class (Small/Medium/Large/Maximum)

### Cross-Domain Onion Layer Operations (NEW)

- DOMAIN_BRIDGE onion layer construction (clearnet→mesh): 5 vectors
  - Verify domain_id encoding at bytes 0-7 of next_hop
  - Verify bridge_node_id at bytes 8-39 of next_hop
  - Verify path_id unchanged across domain transition
- DOMAIN_BRIDGE onion layer construction (mesh→clearnet): 5 vectors
- DOMAIN_BRIDGE onion layer construction (mesh→mesh): 3 vectors
- DOMAIN_BRIDGE onion layer construction (clearnet→satellite): 2 vectors
- DOMAIN_BRIDGE onion unwrapping at bridge node: vectors matching construction
  - No re-encryption: inner layers pass through unchanged
  - Cross-domain path_id table entry created correctly
- MESH_ENTRY (0x07) layer encoding and decoding: 3 vectors
- MESH_EXIT (0x08) layer encoding and decoding: 3 vectors
- MESH_RELAY (0x09) layer encoding and decoding: 3 vectors
- Three-segment cross-domain path (2 domain transitions): 3 vectors
  - Verify path_id identical in all 9 onion layers
  - Verify DOMAIN_BRIDGE layers at correct positions
- N-hop unlimited paths (10 hops): 3 vectors — must construct and parse correctly
- N-hop unlimited paths (15 hops): 3 vectors
- N-hop unlimited paths (20 hops): 3 vectors

### Handshake Messages (Domain-Agnostic Verification)

- HANDSHAKE_INIT construction (two-part format, exactly 256 bytes): 3 vectors
  - Verify bytes 32-63 are cleartext (client ephemeral public key)
  - Verify bytes 64-239 are encrypted with init_key derived from dh1
  - Verify bytes 240-255 are Poly1305 tag
- HANDSHAKE_RESP construction (exactly 256 bytes): 3 vectors
  - Verify bytes 32-63 are cleartext (node ephemeral public key)
  - Verify encrypted payload includes confirmation hash
- HANDSHAKE_DONE construction (exactly 128 bytes): 3 vectors
  - Verify path_id at correct position in encrypted payload
- Full handshake round-trip: 3 vectors
- Re-handshake (24hr key rotation on existing session): 2 vectors

### Node Advertisement

- Advertisement signing (including routing_domains): 5 vectors
- Advertisement verification: 5 vectors
- Advertisement with bridge_capable=true and bridges array: 3 vectors
- Advertisement with mesh routing_domains entry: 3 vectors
- Withdrawal message signing and verification: 3 vectors

### Domain Bridge and Island Records (NEW)

- DOMAIN_ADVERTISE record construction and signature: 3 vectors
  - Verify symmetric DHT key computation (min/max domain_id ordering)
  - Verify bridge pair encoding
- DOMAIN_ADVERTISE record verification: 3 vectors
- MESH_ISLAND_ADVERTISE record construction: 3 vectors
  - Verify island_id → domain_id hash computation
  - Verify DHT key computation
- MESH_ISLAND_ADVERTISE record verification: 3 vectors
- Domain ID wire format (type + instance): 5 vectors
  - clearnet: 0x00000001-00000000
  - mesh:lima-district-1: 0x00000003-BLAKE3("lima-district-1")[0:4]
  - satellite:starlink-leo: 0x00000005-BLAKE3("starlink-leo")[0:4]

### Pool Operations

- Pool advertisement signing: 5 vectors
- Pool membership verification: 5 vectors
- Domain-scoped pool domain_scope field: 3 vectors
  - Verify scope="mesh:island-1" restricts to mesh nodes
- Domain-bridge pool bridges field: 3 vectors

### Pool Curator Key Rotation

- Standard rotation verification (valid dual-signatures): 3 vectors
- Emergency rotation (compromise_confirmed, 2hr window): 2 vectors
- Expired rotation record: 2 vectors
- Invalid old key signature: 2 vectors
- Invalid new key signature: 2 vectors

### ECH Operations

- ECH_REQUIRED stream flag encoding in STREAM_OPEN: 3 vectors
- ECH_NEGOTIATED result flag encoding in DATA: 3 vectors
- ECH stream open with flags bit patterns: 5 vectors

### PATH_QUALITY_REPORT (Updated to 20 bytes)

- PQR report construction with domain_id field (20 bytes plaintext): 3 vectors
  - Verify domain_id at bytes 0-7
  - Verify hop_latency_bucket at byte 8
  - Verify congestion_flag at byte 9
- PQR encryption and decryption (per-hop session key): 3 vectors
- PQR domain-calibrated latency bucket boundaries: 5 vectors per domain type
  (clearnet, wifi_mesh, lora, satellite)
- PQR opaque propagation across domain boundary: 3 vectors
  - Verify intermediate relays cannot decrypt
  - Verify bridge nodes forward without decryption

### Geographic Plausibility Scoring

- NA→EU RTT score computation: 3 vectors (within range, slightly above, far above)
- Same-region RTT score: 2 vectors
- WiFi mesh latency score (domain-specific threshold): 3 vectors
- LoRa latency score (high latency expected, not penalized): 3 vectors
- Satellite exempt (returns 0.5): 2 vectors
- Insufficient measurements returns 0.5: 2 vectors

### MESH_BEACON (Extended Format)

- MESH_BEACON construction with bridge_capable=true and served domains: 3 vectors
- MESH_BEACON construction with bridge_capable=false: 3 vectors
- MESH_BEACON signature verification: 5 vectors
- MESH_BEACON with 3 served domain IDs: 2 vectors

### MESH_FRAGMENT

- Fragmentation of 1000-byte BGP-X packet for LoRa SF7 (200B MTU): 5 vectors
- Fragmentation for BLE (100B MTU): 3 vectors
- Reassembly including out-of-order fragments: 5 vectors
- Fragment timeout (incomplete reassembly): 2 vectors

### Cross-Domain Return Path (NEW)

- Return traffic routing at bridge node: 3 vectors
  - Verify cross-domain path_id lookup resolves correct (domain, source_addr)
  - Verify opaque forwarding (no decryption of return blob at bridge)
- Return traffic routing across multiple bridge transitions: 2 vectors
- Stale path_id (expired entry) → silent drop: 2 vectors

### Negative Test Cases (MUST REJECT)

**Replay detection**:
- Sequence number replay (same sequence twice): 5 vectors (must reject second)
- Sequence number too old (beyond 64-position window): 3 vectors

**Authentication failure**:
- Corrupt Poly1305 authentication tag (single bit flip): 5 vectors
- Wrong session key: 3 vectors

**Invalid advertisements**:
- Expired node advertisement (expires_at in past): 3 vectors
- Mismatched NodeID (BLAKE3(pubkey) ≠ claimed node_id): 3 vectors
- Invalid signature: 5 vectors
- Signed_at in future (>60s clock skew): 2 vectors
- bridge_capable=true but no bridges array: 2 vectors
- ECH_capable=true but no exit role: 2 vectors

**Invalid protocol structures**:
- Invalid path_id (all zeros 0x0000000000000000): 3 vectors (must drop silently)
- Stream ID parity violation (client odd / service even): 4 vectors (must RST)
- Oversized packets (>1280 bytes total): 3 vectors (must drop)
- Onion layer too short (<57 bytes after decryption): 3 vectors
- Unknown hop_type (0x0A and above): 3 vectors (must drop silently)
- DOMAIN_BRIDGE with unknown domain_type: 3 vectors
- DOMAIN_BRIDGE with all-zeros domain_id: 2 vectors

**Pool curator key rotation failures**:
- Expired rotation record (accept_until passed): 3 vectors
- Old key does not match cached: 2 vectors
- New key signature invalid: 2 vectors
- Old key signature invalid: 2 vectors

**Cross-domain failures**:
- DOMAIN_BRIDGE hop at non-bridge-capable node: 2 vectors (must drop silently)
- MESH_RELAY at non-mesh node: 2 vectors (must drop silently)
- Fragment reassembly with total_fragments=0: 2 vectors

---

## File Format

Test vectors will be published as JSON files in this directory:

```
test_vectors/
├── crypto_primitives.json
├── session_key_derivation.json
├── onion_construction_single_domain.json
├── onion_construction_cross_domain.json
├── handshake.json
├── node_advertisement.json
├── domain_bridge_records.json
├── mesh_island_records.json
├── pool_operations.json
├── pool_key_rotation.json
├── ech_flags.json
├── path_quality_reporting.json
├── geo_plausibility.json
├── mesh_beacon.json
├── mesh_fragment.json
├── cross_domain_return_path.json
└── negative_cases.json
```

Each test case object format:

```json
{
  "test_id": "domain_bridge_clearnet_to_mesh_01",
  "description": "DOMAIN_BRIDGE onion layer construction: clearnet relay to mesh:lima-district-1",
  "domain_agnostic_note": "Crypto operations identical to RELAY hop; only hop_type and next_hop encoding differ",
  "input": {
    "hop_type": "0x06",
    "target_domain_id": "0x00000003a1b2c3d4",
    "bridge_node_id": "...",
    "path_id": "...",
    "stream_id": "...",
    "payload": "...",
    "session_key": "...",
    "sequence_number": "..."
  },
  "expected_output": {
    "onion_layer_bytes": "...",
    "decrypted_at_bridge": {
      "hop_type": "0x06",
      "target_domain_id": "0x00000003a1b2c3d4",
      "bridge_node_id": "...",
      "path_id": "...",
      "remaining_ciphertext": "..."
    }
  }
}
```

---

## Publication Timeline

Test vectors will be generated from the reference implementation after it is complete and independently audited. All implementations MUST pass published test vectors before being considered compliant. Community-contributed test vectors for edge cases are welcome via the standard contribution process.
