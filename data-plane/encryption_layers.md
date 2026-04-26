# BGP-X Encryption Layers Specification

**Version**: 0.1.0-draft

---

## 1. Cross-Domain Cryptographic Properties

### Domain-Agnostic Crypto

BGP-X uses identical cryptographic operations regardless of which routing domain a session is in:

- Same ChaCha20-Poly1305 for all onion layers in all domains
- Same X25519 key exchange in all domains
- Same HKDF-SHA256 key derivation in all domains (identical salts and info strings)
- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE format in all domains
- Same nonce construction in all domains

There is NO "mesh crypto" or "clearnet crypto" — one cryptographic suite serves all domains.

### No Re-Encryption at Domain Boundaries

When a packet crosses a domain boundary (DOMAIN_BRIDGE hop), the remaining onion payload is forwarded WITHOUT re-encryption by the bridge node. The bridge node decrypts only its own layer, then forwards the still-encrypted inner layers via the target domain's transport.

This is correct and secure:
- Inner layers are already encrypted for subsequent hop nodes in the target domain
- Bridge node cannot decrypt inner layers (it doesn't have those session keys)
- Clearnet observers see an encrypted UDP datagram (cannot see inner layers)
- Mesh radio observers see an encrypted radio frame (same protection)

### COVER Traffic Across Domains

COVER packets (0x11) use session_key identically in all routing domains. Externally indistinguishable from RELAY in clearnet, mesh, and satellite domains.

---

## 2. Cryptographic Primitive (Unchanged)

**ChaCha20-Poly1305** (RFC 8439). Key: 32 bytes. Nonce: 12 bytes. Tag: 16 bytes. No separate cover_key.

---

## 3. Key Hierarchy (Unchanged, Domain-Agnostic)

```
session_key     = HKDF-SHA256(IKM=dh1||dh2, salt=BLAKE3("bgpx-session-key-v1"), info=session_id||node_id, L=32)
keepalive_key   = HKDF-SHA256(IKM=dh1||dh2, salt=BLAKE3("bgpx-keepalive-key-v1"), info=session_id||node_id, L=32)
init_key        = HKDF-SHA256(IKM=dh1, salt=BLAKE3("bgpx-init-key-v1"), info="handshake-init", L=32)
resp_key        = HKDF-SHA256(IKM=dh1 XOR dh2, salt=BLAKE3("bgpx-resp-key-v1"), info=session_id||"handshake-resp", L=32)
```

No cover_key. COVER uses session_key. Applies in all routing domains.

---

## 4. Nonce Construction (Unchanged, Domain-Agnostic)

```
nonce = 0x00000000 || sequence_number_big_endian_8_bytes  (12 bytes)
```

Two independent sequence number spaces per session (inbound/outbound). No (key, nonce) pair is ever reused. Domain-agnostic — same mechanism in clearnet, mesh, and satellite sessions.

---

## 5. Onion Construction (Cross-Domain)

For a cross-domain path, the client constructs onion layers for ALL hops including domain bridge hops. The construction algorithm is unchanged — one layer per hop, innermost to outermost.

DOMAIN_BRIDGE hops are constructed identically to RELAY hops at the crypto level. The only differences are the `hop_type` value (0x06) and the `next_hop` encoding (domain_id + bridge_node_id).

```python
# Example: clearnet relay(1) → relay(2) → bridge → mesh relay(1) → mesh relay(2) → service
# 5 layers

layer_5 = construct_layer(
    hop_type = DELIVERY,
    next_hop = encode_service(service_id),
    path_id  = path_id,
    key      = session_keys[4],
    payload  = application_data,
    sequence = seq
)

layer_4 = construct_layer(
    hop_type = MESH_RELAY,
    next_hop = encode_mesh_node(mesh_relay_2),
    path_id  = path_id,
    key      = session_keys[3],
    payload  = pad_to_size_class(layer_5),
    sequence = seq
)

layer_3 = construct_layer(
    hop_type = DOMAIN_BRIDGE,
    next_hop = encode_domain_bridge(mesh_island_domain_id, bridge_node_id),
    path_id  = path_id,
    key      = session_keys[2],   # bridge node's session key
    payload  = pad_to_size_class(layer_4),
    sequence = seq
)

layer_2 = construct_layer(
    hop_type = RELAY,
    next_hop = encode_clearnet_node(relay_2),
    path_id  = path_id,
    key      = session_keys[1],
    payload  = pad_to_size_class(layer_3),
    sequence = seq
)

outermost = construct_layer(
    hop_type = RELAY,
    next_hop = encode_clearnet_node(relay_1),
    path_id  = path_id,
    key      = session_keys[0],
    payload  = pad_to_size_class(layer_2),
    sequence = seq
)

# Send outermost to relay_1 via clearnet UDP
```

The bridge node's session key was established via the clearnet UDP endpoint. The bridge node decrypts layer_3, reads the DOMAIN_BRIDGE instruction, and forwards layer_4 (already encrypted for mesh hops) via its mesh radio. It does NOT re-encrypt layer_4.

---

## 6. Return Path Architecture (Cross-Domain)

**Cross-domain return**:
```
Mesh service encrypts response with K_service (client's session key for that hop)
         │
Mesh relay 2 → path_id lookup → M1_mesh_addr → forward blob (opaque, no decrypt)
         │
Mesh relay 1 → path_id lookup → B1_mesh_addr → forward blob (opaque, no decrypt)
         │
Bridge node B1 → cross-domain path_id lookup → R1_clearnet_addr
  → forward blob via clearnet UDP (opaque, no decrypt)
         │
Clearnet relay 1 → path_id lookup → E1_addr (opaque)
         │
Entry E1 → path_id lookup → client_addr (opaque)
         │
Client decrypts with K_service → receives response
```

Only the service and the client can decrypt the response. Bridge nodes and all intermediate relays are opaque forwarders for return traffic in both directions (forward and return), both within and across domains.

---

## 7. Session Re-Handshake (Unchanged, Domain-Agnostic)

24-hour re-handshake generates fresh keys. Same procedure for clearnet and mesh sessions. Re-handshake for a mesh session follows the same relay chain as initial establishment.

---

## 8. ECH at Exit Nodes (Unchanged)

ECH is a TLS-layer mechanism at clearnet exit nodes. Applies when the exit is a clearnet exit regardless of what domains the path traversed before reaching the exit. If the final destination is a mesh island service, ECH is not applicable.

---

## 9. Padding and Size Normalization (Unchanged, Domain-Agnostic)

```
Size classes: 256, 512, 1024, 1280 bytes (total UDP payload)
COVER MUST use same size class distribution as RELAY on same session
Applies in all routing domains
```

For mesh transports with MTU below 256 bytes: MESH_FRAGMENT handles fragmentation at transport layer below the BGP-X crypto layer. Padding and size classes are defined for the pre-fragmentation packet.

---

## 10. Zeroization Ordering (MANDATORY, Unchanged)

```
After dh2 computed:    client_ephemeral_priv → ZEROIZE
After session_key:     dh1, dh2 → ZEROIZE
After handshake done:  init_key, resp_key, node_ephemeral_priv → ZEROIZE
Session closed:        session_key, keepalive_key → ZEROIZE
```

Applies identically in all routing domains.

---

## 11. Implementation Requirements

### MUST

- Use ChaCha20-Poly1305 from an audited library (NOT home-grown)
- Verify Poly1305 authentication tag before accessing any decrypted plaintext (constant-time)
- Use session_key for COVER packets (NOT a separate cover_key)
- Maintain two independent sequence number spaces per session
- Zeroize key material in the mandatory order
- Use `mlock()` or equivalent to prevent session keys from swapping to disk
- Support ECH at exit nodes when ech_capable=true
- Use identical crypto suite for all routing domains (no domain-specific crypto)
- NOT re-encrypt onion payload when transitioning between routing domains at bridge nodes
- NOT decrypt inner layers at domain bridge nodes (only the bridge node's own layer)

### MUST NOT

- Implement ChaCha20, Poly1305, X25519, Ed25519, BLAKE3, or HKDF from scratch
- Use ECB, CBC, or any non-AEAD cipher
- Truncate authentication tags below 16 bytes
- Skip authentication tag verification
- Proceed with plaintext from failed decryption
- Reuse (key, nonce) pairs
- Derive a separate cover_key
- Use domain-specific key derivation
- Re-encrypt at domain boundaries
