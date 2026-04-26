# BGP-X Handshake Protocol Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X handshake is a 3-message exchange between a client and a single node. Performed independently for each node in the path, in order from entry to exit.

**The handshake is completely domain-agnostic.** Whether a session is established over UDP/IP, WiFi 802.11s, LoRa radio, Bluetooth BLE, or satellite: same message format, same X25519 key exchange, same ChaCha20-Poly1305 encryption, same HKDF-SHA256 key derivation, same HANDSHAKE_DONE path_id delivery.

This property is fundamental. One handshake specification serves all routing domains.

The handshake establishes:
- A shared session key (ChaCha20-Poly1305)
- Mutual authentication of node identity (client verifies node's signed DHT advertisement)
- Forward secrecy (ephemeral keys destroyed after handshake)
- path_id context for return routing (including cross-domain return routing)

The handshake does NOT establish:
- Client identity to the node (clients are anonymous by default)
- Path topology to the node (each node learns only its immediate neighbors)

---

## 2. Cryptographic Primitives

| Primitive | Algorithm |
|---|---|
| Key agreement | X25519 (ECDH over Curve25519) |
| Key derivation | HKDF-SHA256 (RFC 5869) |
| Symmetric encryption | ChaCha20-Poly1305 (RFC 8439) |
| Hash | BLAKE3 |

---

## 3. Key Types

### Client Ephemeral Keypair
Generated fresh for each session. Destroyed after session establishment.
```
(client_ephemeral_priv, client_ephemeral_pub) = X25519_keygen()
```

### Node Static Keypair
Long-term identity. Same keypair for all routing domains this node serves. Public key published in DHT advertisement. Private key never leaves the node.
```
(node_static_priv, node_static_pub)
```

### Node Ephemeral Keypair
Generated fresh for each handshake response. Destroyed after session establishment.
```
(node_ephemeral_priv, node_ephemeral_pub) = X25519_keygen()
```

---

## 4. Session Key Derivation

```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
    = X25519(node_static_priv, client_ephemeral_pub)

dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
    = X25519(node_ephemeral_priv, client_ephemeral_pub)

IKM = dh1 || dh2

session_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)

keepalive_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-keepalive-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

No separate cover_key. COVER uses session_key identically to RELAY. Applies in all routing domains.

---

## 5. Zeroization Schedule (MANDATORY)

```
After dh2 computed:
  client_ephemeral_priv → ZEROIZE immediately

After session_key derived:
  dh1 → ZEROIZE immediately
  dh2 → ZEROIZE immediately

After handshake complete:
  init_key → ZEROIZE immediately
  resp_key → ZEROIZE immediately
  node_ephemeral_priv → ZEROIZE immediately (node side)

When session closed:
  session_key → ZEROIZE
  keepalive_key → ZEROIZE
```

Implementations MUST use memory-safe zeroization (Rust `zeroize` crate, C `explicit_bzero`). Standard `memset` is insufficient — compilers may optimize it away.

---

## 6. HANDSHAKE_INIT (Client → Node)

**MUST be exactly 256 bytes total.**

**Why the client ephemeral public key is cleartext**: the node needs it to compute `dh1 = X25519(node_static_priv, client_ephemeral_pub)` before it can derive `init_key` to decrypt anything. The public key is not secret — only the private key is secret and never leaves the client.

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x02)
Bytes 32-63:   Client Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
Bytes 64-239:  Encrypted Payload (176 bytes, ChaCha20-Poly1305)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

**AAD**: bytes 0-63 (header + cleartext pubkey). Both portions authenticated even though only the second is encrypted.

**init_key derivation**:
```
init_key = HKDF-SHA256(
    IKM  = dh1,
    salt = BLAKE3("bgpx-init-key-v1"),
    info = "handshake-init",
    L    = 32
)
```

**Encrypted payload content** (160 bytes plaintext → 176 bytes ciphertext when 16-byte tag removed from total):
```
Session ID          (16 bytes, CSPRNG)
Timestamp           (8 bytes, uint64 Unix epoch)
Protocol Version    (2 bytes)
Extension Flags     (4 bytes)
Padding to 160 bytes
```

**Extension Flags** (same bits as Section 20 of protocol_spec.md):

| Bit | Extension |
|---|---|
| 0 | Cover traffic |
| 1 | Pluggable transport |
| 2 | Stream multiplexing |
| 3 | Path quality reporting |
| 4 | ECH support |
| 5 | Geographic plausibility |
| 6 | Pool support |
| 7 | Mesh transport |
| 8 | Advertisement withdrawal |
| 9 | Session re-handshake |
| 10 | Domain bridge support |
| 11 | Cross-domain path construction |
| 12 | Mesh island routing |
| 13-31 | Reserved (0) |

**Node validation of HANDSHAKE_INIT**:
1. Compute dh1 using cleartext client_ephemeral_pub
2. Derive init_key from dh1
3. Decrypt and authenticate encrypted payload
4. Verify timestamp within ±60 seconds of current time
5. Verify session_id not already in session table
6. Record client_ephemeral_pub, session_id

---

## 7. HANDSHAKE_RESP (Node → Client)

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x03)
Bytes 32-63:   Node Ephemeral Public Key (X25519, 32 bytes) ← CLEARTEXT
Bytes 64-239:  Encrypted Payload (176 bytes)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

**resp_key derivation**:
```
resp_key = HKDF-SHA256(
    IKM  = dh1 XOR dh2,
    salt = BLAKE3("bgpx-resp-key-v1"),
    info = session_id || "handshake-resp",
    L    = 32
)
```

This key requires BOTH node static key AND node ephemeral key. A MITM without node_ephemeral_priv cannot forge HANDSHAKE_RESP even with node_static_priv.

**Encrypted payload content**:
```
Session ID Echo         (16 bytes — MUST match session_id from INIT)
Timestamp               (8 bytes)
Selected Version        (2 bytes)
Accepted Extension Flags (4 bytes)
Confirmation Hash       (32 bytes) = BLAKE3("bgpx-resp-confirm" || session_key || session_id)
Padding
```

**Client validation of HANDSHAKE_RESP**:
1. Compute dh2 using cleartext node_ephemeral_pub
2. Derive session_key from dh1 and dh2
3. Derive resp_key and decrypt/authenticate response
4. Verify session_id echo matches
5. Verify confirmation hash
6. Zeroize: client_ephemeral_priv, dh1, dh2, init_key, resp_key

---

## 8. HANDSHAKE_DONE (Client → Node)

**MUST be exactly 128 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x04)
Bytes 32-111:  Encrypted Payload (80 bytes, using session_key)
Bytes 112-127: Poly1305 Authentication Tag (16 bytes)
```

**Encrypted payload content**:
```
Confirmation Hash   (32 bytes) = BLAKE3("bgpx-done-confirm" || session_key || session_id)
path_id             (8 bytes) — informs node of its path_id for return routing
Padding to 64 bytes
```

For domain bridge nodes: path_id in HANDSHAKE_DONE establishes the cross-domain path table entry — the bridge node stores `path_id → {clearnet_predecessor, mesh_predecessor}` ready for when traffic arrives.

**Node validation of HANDSHAKE_DONE**:
1. Decrypt using session_key
2. Verify confirmation hash
3. Record path_id → source_addr in path table
4. For bridge nodes: initialize cross-domain path table entry for this path_id
5. Transition session to ESTABLISHED
6. Zeroize: node_ephemeral_priv, dh1, dh2, init_key, resp_key

---

## 9. Full Handshake Sequence

```
Client                                               Node
  │                                                    │
  │──── HANDSHAKE_INIT ──────────────────────────────►│
  │     [bytes 32-63: client_ephemeral_pub CLEARTEXT]  │
  │     [bytes 64+: encrypted(session_id, timestamp,   │
  │                 version, extensions)]              │
  │                                                    │
  │     Node computes dh1 using cleartext pubkey       │
  │     Node decrypts init payload                     │
  │     Node generates node_ephemeral keypair          │
  │     Node computes dh2                              │
  │     Node derives session_key, resp_key             │
  │                                                    │
  │◄─── HANDSHAKE_RESP ──────────────────────────────│
  │     [bytes 32-63: node_ephemeral_pub CLEARTEXT]   │
  │     [bytes 64+: encrypted(session_id_echo,        │
  │                 version, extensions,               │
  │                 confirmation_hash)]                │
  │                                                    │
  │     Client computes dh2 using cleartext pubkey     │
  │     Client derives session_key                     │
  │     Client verifies confirmation hash              │
  │     Client zeroizes ephemeral keys                 │
  │                                                    │
  │──── HANDSHAKE_DONE ──────────────────────────────►│
  │     [encrypted(confirmation_hash, path_id)]        │
  │                                                    │
  │     Node verifies confirmation hash                │
  │     Node stores path_id → source_addr              │
  │     Node zeroizes ephemeral keys                   │
  │     Session: ESTABLISHED                           │
  │                                                    │
  │◄──────────── RELAY / DATA / KEEPALIVE ───────────►│
```

---

## 10. Path-Wide Handshake Order (Cross-Domain)

For a cross-domain path: clearnet relays first (entry → subsequent clearnet relays), then domain bridge node, then mesh relays (delivered via bridge node's established session).

**Example**: clearnet relay 1 → relay 2 → bridge node → mesh relay 1 → mesh relay 2

```
Step 1: Client ↔ clearnet relay 1: HANDSHAKE (direct UDP)
        [relay 1 session ESTABLISHED]

Step 2: Client ↔ clearnet relay 2: HANDSHAKE (via relay 1 session)
        [relay 2 session ESTABLISHED]

Step 3: Client ↔ bridge node: HANDSHAKE (via relay 2 session; arrives as UDP at bridge)
        [bridge node session ESTABLISHED]

Step 4: Client ↔ mesh relay 1: HANDSHAKE (via bridge node session)
        Bridge node forwards HANDSHAKE_INIT into mesh via radio
        Mesh relay 1 receives via WiFi mesh / LoRa
        Mesh relay 1 sends HANDSHAKE_RESP via radio → bridge node → client
        [mesh relay 1 session ESTABLISHED]

Step 5: Client ↔ mesh relay 2: HANDSHAKE (via established chain)
        [mesh relay 2 session ESTABLISHED]
```

**Critical property**: the mesh relay never directly receives a UDP packet from the client. The bridge node delivers all handshake traffic for mesh hops via its radio. From the mesh relay's perspective, a standard session is being established. The mesh relay does not know the session request originated from the clearnet.

**Privacy**: entry nodes (first hop) are the only nodes that learn the client's real transport address. Mesh relays learn only the preceding mesh node's address.

---

## 11. Domain-Agnostic Handshake: Technical Basis

The handshake is a pure cryptographic protocol. It references no IP addresses, port numbers, or transport-specific fields in its cryptographic material. The session_id and node_id used in key derivation are application-layer identifiers unrelated to transport.

The BGP-X transport abstraction layer (UDP socket, WiFi mesh raw socket, LoRa serial interface) handles physical delivery below the handshake layer. The handshake code is identical regardless of transport.

---

## 12. Session Re-Handshake (24-Hour Key Rotation)

Sessions exceeding 24 hours SHOULD initiate a re-handshake on the same path, generating fresh ephemeral keys and new session keys.

**Procedure**:
1. Client generates new ephemeral keypair for this hop
2. Client sends HANDSHAKE_INIT on the existing session (session_id unchanged)
3. Node responds with new node_ephemeral_pub
4. New session_key derived from new ephemeral keys
5. In-flight streams complete using old session_key
6. New streams use new session_key
7. After all streams migrated: old session_key zeroized

Domain-agnostic — identical procedure for clearnet and mesh sessions. For a mesh session established via a bridge node, the re-handshake follows the same relay chain as initial establishment.

---

## 13. Handshake Timeouts by Domain

| Domain | Handshake Timeout |
|---|---|
| Clearnet | 10 seconds |
| WiFi mesh | 15 seconds |
| LoRa (any SF) | 60 seconds |
| Satellite LEO | 20 seconds |
| Satellite GEO | 15 seconds |

For cross-domain paths: each hop uses the timeout appropriate to its domain. Total path establishment time ≈ sum of per-hop timeouts.

MESH_FRAGMENT handles HANDSHAKE_INIT fragmentation for LoRa (256 bytes; LoRa SF12 MTU ~50 bytes). The handshake layer receives a complete 256-byte HANDSHAKE_INIT after reassembly.

---

## 14. Replay Attack Prevention

- Timestamp in HANDSHAKE_INIT: ±60 seconds acceptance window
- Session ID: 16-byte random; collision probability negligible (2^-128)
- Live session table lookup: duplicate session_id rejected
- Timestamps prevent HANDSHAKE_INIT replay across time windows
- After session close: session ID freed; probability of collision with new random ID negligible

---

## 15. Session Teardown

Sessions are torn down when:
- All streams closed and no KEEPALIVE sent for 90 seconds
- KEEPALIVE timeout (90 seconds without any traffic)
- ERROR message received
- Handshake re-attempt fails after 3 retries
- Node daemon shutting down (DRAIN state)

On teardown, nodes MUST:
1. Zeroize session_key from memory
2. Zeroize keepalive_key from memory
3. Remove session from session table
4. Remove path_id → source mappings for this session's path_ids
5. For bridge nodes: remove cross-domain path_id entries associated with this session
6. Drop queued packets for this session
