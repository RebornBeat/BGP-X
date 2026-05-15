# BGP-X Encryption Layers Specification

**Version**: 0.1.0-draft

This document specifies the complete encryption scheme for BGP-X — onion layering, cryptographic primitives, key derivation, nonce construction, and the return path architecture across all routing domains.

---

## 1. Overview

BGP-X uses **layered symmetric encryption** (onion encryption) to ensure no relay node can decrypt more than its own layer. Each layer uses a key shared only between the client and that specific hop.

### 1.1 Requirements

- **Unlinkability**: A relay node cannot correlate incoming and outgoing packets by ciphertext content.
- **Unforgeability**: A relay node cannot inject or modify packets without detection.
- **Forward secrecy**: Compromise of long-term keys does not expose past sessions.
- **Efficiency**: Fast enough for wire-speed operation.
- **Domain-agnostic**: Encryption properties are identical in all routing domains (clearnet, mesh, satellite).

### 1.2 Cross-Domain Cryptographic Properties

BGP-X uses identical cryptographic operations regardless of which routing domain a session is in:

- Same ChaCha20-Poly1305 for all onion layers in all domains.
- Same X25519 key exchange in all domains.
- Same HKDF-SHA256 key derivation in all domains (identical salts and info strings).
- Same HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE format in all domains.
- Same nonce construction in all domains.

There is **NO "mesh crypto" or "clearnet crypto"** — one cryptographic suite serves all domains.

### 1.3 No Re-Encryption at Domain Boundaries

When a packet crosses a domain boundary (DOMAIN_BRIDGE hop), the remaining onion payload is forwarded **WITHOUT re-encryption** by the bridge node. The bridge node decrypts only its own layer, then forwards the still-encrypted inner layers via the target domain's transport.

This is correct and secure:
- Inner layers are already encrypted for subsequent hop nodes in the target domain.
- Bridge node cannot decrypt inner layers (it doesn't have those session keys).
- Clearnet observers see an encrypted UDP datagram (cannot see inner layers).
- Mesh radio observers see an encrypted radio frame (same protection).

---

## 2. Cryptographic Primitive: ChaCha20-Poly1305

All onion layers use **ChaCha20-Poly1305** (RFC 8439) as the AEAD cipher.

### 2.1 Why ChaCha20-Poly1305

| Property | Detail |
|---|---|
| Performance without AES-NI | Faster than AES-GCM on hardware without AES hardware acceleration (common in embedded/router hardware). |
| Performance with AES-NI | Slightly slower than AES-GCM on x86 with AES-NI, but acceptably so. |
| Side-channel resistance | ChaCha20 is designed to be constant-time; no S-box lookups that could leak via cache timing. |
| Key size | 256-bit key (full security margin). |
| Nonce size | 96-bit (12-byte) nonce. |
| Tag size | 128-bit (16-byte) Poly1305 authentication tag. |
| Authentication | Poly1305 provides full AEAD — ciphertext integrity is verified before decryption. |

### 2.2 Parameters

- **Key**: 32 bytes (256 bits), unique per (session, hop).
- **Nonce**: 12 bytes, constructed from sequence number (see Section 4).
- **Tag**: 16 bytes, appended to ciphertext.
- **AAD**: 32-byte BGP-X common header.

---

## 3. Key Hierarchy

BGP-X uses a hierarchy of keys derived from the handshake:

```
Handshake Key Material (from X25519 ECDH)
    │
    ├── session_key (32 bytes)
    │       └── Used for: RELAY, DATA, KEEPALIVE, COVER (ALL traffic on this session)
    │
    ├── keepalive_key (32 bytes)
    │       └── Used for: KEEPALIVE message authentication (separate from RELAY/DATA)
    │
    ├── init_key (32 bytes)
    │       └── Used for: HANDSHAKE_INIT encryption (derived from dh1 only)
    │
    └── resp_key (32 bytes)
            └── Used for: HANDSHAKE_RESP encryption (derived from dh1 XOR dh2)
```

**Critical**: There is **NO separate cover_key**. COVER packets use `session_key` identically to RELAY packets. This makes COVER packets externally indistinguishable from RELAY packets in all routing domains.

---

## 4. Nonce Construction

ChaCha20-Poly1305 nonces are 12 bytes, constructed from the packet sequence number:

```
nonce = 0x00000000 || sequence_number_big_endian_8_bytes
```

First 4 bytes always zero. Last 8 bytes are the 64-bit **outbound** sequence number in big-endian format.

### 4.1 Bidirectional Sequence Numbers

Each session maintains **TWO independent sequence number spaces**:

- **Outbound**: Sequence numbers for packets sent by this node to the next hop.
- **Inbound**: Sequence numbers for packets received from the previous hop.

The common header carries the outbound sequence number. The replay detection window tracks the inbound sequence space separately.

This separation prevents nonce reuse: even if identical data is sent both directions, the different sequence spaces produce different nonces.

### 4.2 Nonce Uniqueness Guarantee

- Session keys are derived from ephemeral key material (unique per session).
- Outbound sequence numbers within a session are strictly monotonically increasing.
- No two packets in the same session will have the same sequence number.
- No two packets with the same sequence number can use the same key (different sessions = different keys).
- Nonce uniqueness is guaranteed across all BGP-X packets for a given key.

### 4.3 Nonce Reuse Prevention

BGP-X prevents nonce reuse by:

- Using the sequence number as a nonce input — sequence numbers are strictly increasing.
- Session keys are never reused across sessions (fresh handshake = fresh keys).
- On crash recovery, sessions are NOT resumed (new handshake required) to avoid sequence number ambiguity.
- The session table tracks the maximum outbound sequence number.
- Two independent spaces prevent cross-direction nonce collision.

---

## 5. Key Derivation

All keys are derived from handshake ECDH output via HKDF-SHA256:

### 5.1 Input Key Material

```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)

IKM = dh1 || dh2
```

### 5.2 Session Key

```
session_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

### 5.3 Keepalive Key

```
keepalive_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-keepalive-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

### 5.4 Handshake Keys

```
init_key = HKDF-SHA256(
    IKM  = dh1,
    salt = BLAKE3("bgpx-init-key-v1"),
    info = "handshake-init",
    L    = 32
)

resp_key = HKDF-SHA256(
    IKM  = dh1 XOR dh2,
    salt = BLAKE3("bgpx-resp-key-v1"),
    info = session_id || "handshake-resp",
    L    = 32
)
```

**Note**: `init_key` is derived from `dh1` only because the node's ephemeral public key is not yet known when the client sends HANDSHAKE_INIT. `resp_key` uses both DH results.

---

## 6. Onion Layer Construction (Client Side)

The client constructs the full onion packet before transmitting. This is a layered encryption process, working from innermost (exit) to outermost (entry).

### 6.1 Input

```
path         = [entry_node, relay_1, ..., relay_n, exit_node]
session_keys = [key_entry, key_relay_1, ..., key_relay_n, key_exit]
payload      = application data (plaintext)
destination  = clearnet address or BGP-X ServiceID
path_id      = 8-byte CSPRNG identifier for this path
sequence     = current outbound sequence number
stream_id    = 4-byte stream identifier
```

### 6.2 Construction Algorithm (Single Domain)

```python
function construct_onion(path, session_keys, payload, destination, path_id, sequence, stream_id):

    N = len(path)

    # Build innermost layer (exit hop)
    inner_plaintext = build_layer_header(
        hop_type  = exit_hop_type(destination),
        next_hop  = encode_destination(destination),
        path_id   = path_id,
        stream_id = stream_id,
        flags     = 0x0000
    ) || payload

    inner_plaintext = pad_to_size_class(inner_plaintext)

    inner_header = make_common_header(0x01, sequence)

    inner_ct = ChaCha20-Poly1305-Encrypt(
        key   = session_keys[N-1],
        nonce = 0x00000000 || sequence_BE8,
        aad   = inner_header,
        pt    = inner_plaintext
    )

    current_ct = inner_ct

    # Wrap each layer from exit-1 toward entry
    for i in range(N-2, -1, -1):
        layer_plaintext = build_layer_header(
            hop_type  = 0x01,              # RELAY
            next_hop  = encode_bgpx_node(path[i+1]),
            path_id   = path_id,           # Same path_id in all layers
            stream_id = stream_id,
            flags     = 0x0000
        ) || current_ct

        layer_plaintext = pad_to_size_class(layer_plaintext)
        layer_header = make_common_header(0x01, sequence)

        layer_ct = ChaCha20-Poly1305-Encrypt(
            key   = session_keys[i],
            nonce = 0x00000000 || sequence_BE8,
            aad   = layer_header,
            pt    = layer_plaintext
        )

        current_ct = layer_ct

    return current_ct  # Outermost layer → send to entry node
```

### 6.3 Cross-Domain Onion Construction

For a cross-domain path, the client constructs onion layers for ALL hops including domain bridge hops. The construction algorithm is unchanged — one layer per hop, innermost to outermost.

DOMAIN_BRIDGE hops are constructed identically to RELAY hops at the crypto level. The only differences are the `hop_type` value (0x06) and the `next_hop` encoding (domain_id + bridge_node_id).

**Example: Clearnet → Clearnet Relays → Domain Bridge → Mesh Relays → Service**

```python
# 5 layers: clearnet relay(1) → relay(2) → bridge → mesh relay(1) → mesh relay(2) → service

# Layer 5 (innermost, Service)
layer_5 = construct_layer(
    hop_type = DELIVERY,
    next_hop = encode_service(service_id),
    path_id  = path_id,
    key      = session_keys[4],
    payload  = application_data,
    sequence = seq
)

# Layer 4 (Mesh Relay 2)
layer_4 = construct_layer(
    hop_type = MESH_RELAY,
    next_hop = encode_mesh_node(mesh_relay_2),
    path_id  = path_id,
    key      = session_keys[3],
    payload  = pad_to_size_class(layer_5),
    sequence = seq
)

# Layer 3 (Domain Bridge)
layer_3 = construct_layer(
    hop_type = DOMAIN_BRIDGE,
    next_hop = encode_domain_bridge(mesh_island_domain_id, bridge_node_id),
    path_id  = path_id,
    key      = session_keys[2],   # Bridge node's session key
    payload  = pad_to_size_class(layer_4),
    sequence = seq
)

# Layer 2 (Clearnet Relay 2)
layer_2 = construct_layer(
    hop_type = RELAY,
    next_hop = encode_clearnet_node(relay_2),
    path_id  = path_id,
    key      = session_keys[1],
    payload  = pad_to_size_class(layer_3),
    sequence = seq
)

# Layer 1 (Outermost, Clearnet Relay 1)
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

The bridge node's session key was established via the clearnet UDP endpoint. The bridge node decrypts layer_3, reads the DOMAIN_BRIDGE instruction, and forwards layer_4 (already encrypted for mesh hops) via its mesh radio. It does **NOT** re-encrypt layer_4.

### 6.4 Size Class Padding

Before encrypting each layer:

| Size Class | Total UDP Payload |
|---|---|
| Small | 256 bytes |
| Medium | 512 bytes |
| Large | 1024 bytes |
| Maximum | 1280 bytes |

Padding is added inside innermost encryption layer (before onion wrapping) — encrypted and indistinguishable from payload.

Padding structure:
```
[original_content][random_padding_bytes][padding_length: uint16]
```

**COVER packets MUST use the same size class distribution as RELAY packets on the same session.**

For mesh transports with MTU below 256 bytes: MESH_FRAGMENT handles fragmentation at transport layer below the BGP-X crypto layer. Padding and size classes are defined for the pre-fragmentation packet.

---

## 7. Onion Layer Decryption (Relay Node Side)

```python
function decrypt_onion_layer(session, header, ciphertext):

    # Replay check (inbound sequence space)
    if not check_inbound_sequence(session, header.sequence_number):
        return DROP

    # Construct nonce
    nonce = 0x00000000 || header.sequence_number_BE8

    # Decrypt and verify
    plaintext = ChaCha20-Poly1305-Decrypt(
        key        = session.session_key,
        nonce      = nonce,
        aad        = header.serialize(),   # 32 bytes
        ciphertext = ciphertext
    )

    if plaintext is None:
        return DROP  # Authentication failure

    # Strip padding
    padding_length = uint16_from_bytes(plaintext[-2:])
    if padding_length > len(plaintext) - 2:
        return DROP  # Invalid padding
    plaintext = plaintext[:len(plaintext) - 2 - padding_length]

    # Parse layer header (57 bytes)
    if len(plaintext) < 57:
        return DROP  # Too short

    hop_type  = plaintext[0]
    next_hop  = plaintext[1:41]
    path_id   = plaintext[41:49]
    stream_id = plaintext[49:53]
    flags     = plaintext[53:55]
    remaining = plaintext[57:]

    if path_id == bytes(8):  # All zeros
        return DROP  # Invalid path_id

    return (hop_type, next_hop, path_id, stream_id, remaining)
```

### 7.1 DOMAIN_BRIDGE Hop Handling

When `hop_type = 0x06`:

1. Parse target domain ID (bytes 0-7 of `next_hop`).
2. Parse bridge node ID (bytes 8-39 of `next_hop`).
3. Store `path_id → source_addr` in current domain path table.
4. Store `path_id → {source_domain, source_addr, target_domain}` in cross-domain path table.
5. Select transport appropriate to target domain.
6. Forward remaining onion payload via target-domain transport to bridge node.
7. **No re-encryption**: the remaining payload is already encrypted for subsequent hops.

---

## 8. Cover Traffic Encryption

COVER packets (0x11) use `session_key` identically to RELAY packets.

COVER payload: random bytes formatted to mimic valid onion layer structure (same size classes as RELAY).

At the recipient relay node:
1. Decrypt using `session_key` (same as RELAY).
2. Attempt to parse layer header.
3. If parse fails or produces uninterpretable result: drop silently.
4. This is **EXPECTED behavior** for COVER — the point is external indistinguishability.

From an external observer's perspective: COVER is completely indistinguishable from RELAY. Same size class, same session key, same header format, same ChaCha20-Poly1305 ciphertext.

---

## 9. Return Path Architecture

Return traffic uses a different architecture than outbound.

### 9.1 Single-Domain Return Path

```
Exit node receives clearnet response
         │
         ▼
Exit node encrypts response using client's session key K_exit
(Client and exit share K_exit from handshake)
         │
         ▼
Exit sends encrypted blob to last relay (using path_id → predecessor lookup)
         │
         ▼
Relay N receives blob → looks up path_id → forwards to Relay N-1 (OPAQUE, no decryption)
         │
         ▼
...
         │
         ▼
Relay 1 receives blob → looks up path_id → forwards to Entry (OPAQUE)
         │
         ▼
Entry receives blob → looks up path_id → forwards to Client (OPAQUE)
         │
         ▼
Client decrypts using K_exit → receives clearnet response
```

### 9.2 Cross-Domain Return Path

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

### 9.3 Why Intermediate Relays Don't Decrypt Return Traffic

- Intermediate relays do **NOT** have the client's session key for the exit node (`K_exit`).
- Only the exit node and the client share `K_exit`.
- Intermediate relays see an opaque encrypted blob.
- They forward it using `path_id → predecessor` mapping **without decryption**.

### 9.4 Return Traffic Sequence Numbers

Return traffic uses the **INBOUND** sequence space at each relay:
- Exit sends return blob with its own outbound sequence number.
- Each relay maintains `path_id → (predecessor, inbound_session)` mapping.
- Relay forwards using the `inbound_session`'s outbound sequence number.
- The client receives with its own inbound sequence tracking.

---

## 10. ECH at Exit Nodes

When an exit node connects to a clearnet TLS destination:

1. **DNS Resolution**: Exit node queries DNS (DoH preferred) for HTTPS record of destination domain.

2. **ECH Check**: If HTTPS record contains `ECHConfig`:
   - Use ECH in TLS ClientHello.
   - Inner ClientHello contains real SNI (encrypted to server's ECH public key).
   - Outer ClientHello contains generic SNI.
   - Network observers and exit node itself cannot read the real SNI from ClientHello.

3. **No ECH Support**:
   - If stream flags contain `ECH_REQUIRED`: fail with `ERR_ECH_REQUIRED_NOT_SUPPORTED`.
   - If `ECH_PREFERRED` only: proceed with standard TLS (SNI visible to exit node).
   - One retry without ECH is allowed if ECH negotiation fails and `require_ech=false`.

4. **ECH Config Cache**:
   - TTL: 1 hour.
   - Key: destination domain.
   - After cache miss: fetch via DoH before establishing TLS.

ECH result returned to client via stream DATA flags:
- Bit 11: `ECH_NEGOTIATED` (1 = ECH was used).
- Bit 12: `ECH_AVAILABLE` (1 = destination supports ECH).

**Cross-Domain Note**: ECH applies only when the final hop is a clearnet exit. If the final destination is a mesh island service, ECH is not applicable.

---

## 11. Session Re-Handshake Key Update

When a session exceeds 24 hours:

1. Client generates new ephemeral X25519 keypair for this hop.
2. Client sends HANDSHAKE_INIT on existing session (`session_id` unchanged).
3. Node processes as re-handshake intent (non-zero `session_id`).
4. New `dh1`, `dh2` computed with new ephemeral keys.
5. New `session_key` derived.
6. In-flight streams: complete with old `session_key`.
7. New streams: use new `session_key`.
8. After all streams migrated: old `session_key` zeroized.

Implementations **MUST** support re-handshake on established sessions.

---

## 12. Mesh Transport Encryption

Onion encryption is **UNCHANGED** for mesh transports. The same ChaCha20-Poly1305 algorithm, the same key derivation, the same nonce construction.

The transport layer (below BGP-X crypto) handles radio-specific concerns:
- Fragmentation (`MESH_FRAGMENT` for LoRa).
- Transport addressing (NodeID → MAC/LoRa addr mapping).
- Duty cycle compliance (LoRa EU 1%).

From the BGP-X encryption layer's perspective, mesh and internet are identical.

---

## 13. Implementation Requirements

### 13.1 MUST

- Use ChaCha20-Poly1305 from an audited library (NOT home-grown).
- Verify Poly1305 authentication tag before accessing any decrypted plaintext (constant-time).
- Use `session_key` for COVER packets (NOT a separate `cover_key`).
- Maintain two independent sequence number spaces per session (inbound/outbound).
- Zeroize key material in the mandatory order:
  1. `client_ephemeral_priv` → after `dh2` computed.
  2. `node_ephemeral_priv` → after session established (node side).
  3. `dh1` → after `session_key` derived.
  4. `dh2` → after `session_key` derived.
  5. `init_key` → after handshake complete.
  6. `resp_key` → after handshake complete.
  7. `session_key` → when session closed.
  8. `keepalive_key` → when session closed.
- Use `mlock()` (Linux) or equivalent to prevent session keys from swapping to disk.
- Support ECH at exit nodes when `ech_capable=true`.
- Use identical crypto suite for all routing domains (no domain-specific crypto).
- **NOT** re-encrypt onion payload when transitioning between routing domains at bridge nodes.
- **NOT** decrypt inner layers at domain bridge nodes (only the bridge node's own layer).

### 13.2 MUST NOT

- Implement ChaCha20, Poly1305, X25519, Ed25519, BLAKE3, or HKDF from scratch.
- Use ECB, CBC, or any non-AEAD cipher.
- Truncate authentication tags below 16 bytes.
- Skip authentication tag verification.
- Proceed with plaintext from failed decryption.
- Reuse (key, nonce) pairs.
- Derive a separate `cover_key`.
- Use domain-specific key derivation.
- Re-encrypt at domain boundaries.
- Log prohibited items:
  - Client IPs at relay/exit nodes.
  - Destinations at relay nodes.
  - `path_id` values.
  - Path composition.
  - Session IDs.
  - Traffic volume per path.
  - Pool query history.
  - Cross-domain traversal details per session.
  - Mesh island identifiers associated with specific sessions.

---

## 14. Recommended Cryptographic Libraries

| Language | Library | Notes |
|---|---|---|
| Rust | `ring` or `chacha20poly1305` (RustCrypto) | Both audited and constant-time. |
| Go | `golang.org/x/crypto/chacha20poly1305` | Standard library extension. |
| C | `libsodium` | Well-audited, widely deployed. |
| Python | `cryptography` (PyCA) | OpenSSL backend. |
| JavaScript | `@noble/ciphers` | Audited pure-JS. |

Reference implementation uses Rust with `ring` crate.
