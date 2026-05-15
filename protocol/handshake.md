# BGP-X Handshake Protocol Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X session handshake — the process by which a client establishes a shared session key with each node in a path.

---

## 1. Overview

The BGP-X handshake is a 3-message exchange between a client and a single node. It is performed independently for each node in the path, in order from entry to exit.

**The handshake is completely domain-agnostic.** Whether a session is established over UDP/IP, WiFi 802.11s, LoRa radio, Bluetooth BLE, or satellite: same message format, same X25519 key exchange, same ChaCha20-Poly1305 encryption, same HKDF-SHA256 key derivation, same HANDSHAKE_DONE path_id delivery.

This property is fundamental. One handshake specification serves all routing domains.

The handshake establishes:

- A shared session key (used for all subsequent DATA, RELAY, KEEPALIVE, and COVER messages on this session)
- Mutual authentication of node identity (client verifies it is talking to the correct node by public key)
- Forward secrecy (session key is derived from ephemeral keys; compromise of long-term keys does not expose past sessions)
- path_id context (client informs node of its path_id for return routing, including cross-domain return routing)

The handshake does NOT establish:

- Client identity to the node (clients are anonymous by default)
- Path topology to the node (nodes learn only their immediate neighbors)

---

## 2. Cryptographic Primitives

The handshake uses the following primitives:

| Primitive | Algorithm |
|---|---|
| Key agreement | X25519 (ECDH over Curve25519) |
| Key derivation | HKDF-SHA256 (RFC 5869) |
| Symmetric encryption | ChaCha20-Poly1305 (RFC 8439) |
| Hash | BLAKE3 |
| Signatures (for advertisements) | Ed25519 |

---

## 3. Key Types

### Client Ephemeral Keypair

Generated fresh for each session. MUST be destroyed after session establishment.

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

The session key is derived via two-stage ECDH:

**Stage 1**: Client ephemeral × Node static

```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
    = X25519(node_static_priv, client_ephemeral_pub)
```

This allows the client to encrypt HANDSHAKE_INIT content using only the node's public key (from DHT advertisement) before the node has responded.

**This is WHY the client ephemeral public key MUST be cleartext in HANDSHAKE_INIT**: the node needs it to compute `dh1 = X25519(node_static_priv, client_ephemeral_pub)` and derive `init_key` to decrypt the encrypted portion. This is NOT a security problem — the ephemeral public key is not secret (only the private key is secret, which never leaves the client).

**Stage 2**: Client ephemeral × Node ephemeral

```
dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
    = X25519(node_ephemeral_priv, client_ephemeral_pub)
```

This adds forward secrecy — compromise of node_static_priv alone cannot recover the session key because dh2 requires node_ephemeral_priv (which is destroyed after handshake).

**Input key material**:

```
IKM = dh1 || dh2
```

**Session key**:

```
session_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

**Keepalive key**:

```
keepalive_key = HKDF-SHA256(
    IKM  = IKM,
    salt = BLAKE3("bgpx-keepalive-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

**Important**: There is NO separate cover_key. COVER packets use session_key identically to RELAY packets. This applies in all routing domains.

---

## 5. Zeroization Schedule (MANDATORY)

All intermediate key material MUST be zeroized in this order:

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

Implementations MUST use memory-safe zeroization (e.g., `zeroize` crate in Rust, `explicit_bzero` in C). Standard `memset` is not sufficient as compilers may optimize it away.

---

## 6. HANDSHAKE_INIT (Client → Node)

### Pre-conditions

- Client has retrieved and verified the node's DHT advertisement
- Client has generated a fresh ephemeral X25519 keypair
- Client has computed: `dh1 = X25519(client_ephemeral_priv, node_static_pub)`
- Client has derived: `init_key = HKDF-SHA256(dh1, BLAKE3("bgpx-init-key-v1"), "handshake-init", 32)`
- Client has generated: `session_id` (16 bytes CSPRNG)

### Wire Format

**MUST be exactly 256 bytes total.**

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x02)
Bytes 32-63:   Client Ephemeral Public Key (X25519, 32 bytes)
               ← CLEARTEXT (not encrypted)
               ← Node needs this to compute dh1 and decrypt everything else
Bytes 64-239:  Encrypted Payload (176 bytes, ChaCha20-Poly1305)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

**AAD** (authenticated but not encrypted): bytes 0-63 (header + cleartext pubkey). Both portions authenticated even though only the second is encrypted.

**Encrypted payload content** (160 bytes plaintext → 176 bytes ciphertext after 16-byte tag):

```
Session ID          (16 bytes, CSPRNG)
Timestamp           (8 bytes, uint64, Unix epoch)
Protocol Version    (2 bytes)
Extension Flags     (4 bytes)
Padding             (zero bytes to 160 bytes plaintext total)
```

### Extension Flags

| Bit | Extension |
|---|---|
| 0 | Cover traffic support |
| 1 | Pluggable transport |
| 2 | Stream multiplexing |
| 3 | Path quality reporting |
| 4 | ECH support |
| 5 | Geographic plausibility (OPTIONAL — jurisdiction declaration is optional) |
| 6 | Pool support |
| 7 | Mesh transport |
| 8 | Advertisement withdrawal |
| 9 | Session re-handshake |
| 10 | Domain bridge support |
| 11 | Cross-domain path construction |
| 12 | Mesh island routing |
| 13-31 | Reserved (0) |

### Node Validation of HANDSHAKE_INIT

Node MUST:

1. Compute dh1 using client_ephemeral_pub (cleartext from bytes 32-63)
2. Derive init_key from dh1
3. Decrypt and authenticate the encrypted payload
4. Verify timestamp is within ±60 seconds of current time (clock skew tolerance)
5. Verify session_id is not already in use in current session table
6. Record: client_ephemeral_pub, session_id

---

## 7. HANDSHAKE_RESP (Node → Client)

**MUST be exactly 256 bytes total.**

### Wire Format

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x03)
Bytes 32-63:   Node Ephemeral Public Key (X25519, 32 bytes)
               ← CLEARTEXT (client needs this to compute dh2)
Bytes 64-239:  Encrypted Payload (176 bytes)
Bytes 240-255: Poly1305 Authentication Tag (16 bytes)
```

**AAD**: bytes 0-63 (header + cleartext pubkey).

**Response key** (used to encrypt HANDSHAKE_RESP):

```
resp_key = HKDF-SHA256(
    IKM  = dh1 XOR dh2,
    salt = BLAKE3("bgpx-resp-key-v1"),
    info = session_id || "handshake-resp",
    L    = 32
)
```

This key requires BOTH node static key AND node ephemeral key, making it impossible for a MITM to forge the response even if they know the node's static private key alone (they also need dh2 which requires node_ephemeral_priv).

**Encrypted payload content**:

```
Session ID Echo         (16 bytes, MUST match session_id from INIT)
Timestamp               (8 bytes)
Selected Version        (2 bytes)
Accepted Extension Flags (4 bytes)
Confirmation Hash       (32 bytes)
  = BLAKE3("bgpx-resp-confirm" || session_key || session_id)
Padding
```

### Client Validation of HANDSHAKE_RESP

Client MUST:

1. Compute dh2 using node_ephemeral_pub (cleartext from response bytes 32-63)
2. Derive session_key from dh1 and dh2
3. Derive resp_key and decrypt/authenticate the response
4. Verify session_id echo matches sent session_id
5. Verify confirmation hash
6. Record session_key
7. Zeroize: client_ephemeral_priv, dh1, dh2, init_key, resp_key

---

## 8. HANDSHAKE_DONE (Client → Node)

**MUST be exactly 128 bytes total.**

### Wire Format

```
Bytes 0-31:    BGP-X Common Header (Msg Type = 0x04)
Bytes 32-111:  Encrypted Payload (80 bytes, using session_key)
Bytes 112-127: Poly1305 Authentication Tag (16 bytes)
```

**Encrypted payload content**:

```
Confirmation Hash       (32 bytes)
  = BLAKE3("bgpx-done-confirm" || session_key || session_id)
path_id                 (8 bytes) — informs node of its path_id for return routing
Padding to 64 bytes
```

### path_id Field

The path_id (8 bytes) is:
- Generated by the client using CSPRNG
- Unique per path instance
- Included in EVERY onion layer of the same path
- The same value at every hop of the same path
- Never reused within a session
- Not linked to any client identity
- Not logged by any relay node

When a relay decrypts its layer and extracts path_id, it stores `path_id → source_addr` in an in-memory table. This mapping enables return traffic routing without requiring the relay to know the full path structure.

For domain bridge nodes: path_id in HANDSHAKE_DONE establishes the cross-domain path table entry — the bridge node stores `path_id → {clearnet_predecessor, mesh_predecessor}` ready for when traffic arrives.

### Node Validation of HANDSHAKE_DONE

Node MUST:

1. Decrypt using session_key
2. Verify confirmation hash
3. Record path_id → source_addr in path table
4. For domain bridge nodes: initialize cross-domain path table entry:
   - Store path_id → predecessor in current domain's path table
   - Store path_id → {predecessor_in_domain_A, predecessor_in_domain_B} in cross-domain path table
   - Both predecessors recorded; enables return routing across domain boundary
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
  │     For bridge nodes: cross-domain path_id entry   │
  │     Node zeroizes ephemeral keys                   │
  │     Session: ESTABLISHED                           │
  │                                                    │
  │◄──────────── RELAY / DATA / KEEPALIVE ───────────►│
```

---

## 10. Path-Wide Handshake Order (Cross-Domain)

A BGP-X path of N hops requires N independent handshakes — one with each node.

**Order**: Entry first (direct UDP), then each subsequent hop via established sessions.

For a cross-domain path: clearnet relays first (entry → subsequent clearnet relays), then domain bridge node, then mesh relays (delivered via bridge node's established session).

**Example**: clearnet relay 1 → relay 2 → bridge node → mesh relay 1 → mesh relay 2

```
Step 1: Client ↔ clearnet relay 1: HANDSHAKE (direct UDP)
        [relay 1 session ESTABLISHED]

Step 2: Client ↔ clearnet relay 2: HANDSHAKE (via relay 1 session)
        [relay 2 session ESTABLISHED]

Step 3: Client ↔ bridge node: HANDSHAKE (via relay 2 session; arrives as UDP at bridge)
        Bridge node receives via clearnet UDP
        Bridge node session ESTABLISHED

Step 4: Client ↔ mesh relay 1: HANDSHAKE (via bridge node session)
        Bridge node forwards HANDSHAKE_INIT into mesh via radio
        Mesh relay 1 receives via WiFi mesh / LoRa
        Mesh relay 1 sends HANDSHAKE_RESP via radio → bridge node → client
        [mesh relay 1 session ESTABLISHED]

Step 5: Client ↔ mesh relay 2: HANDSHAKE (via established chain)
        [mesh relay 2 session ESTABLISHED]
```

**Critical property**: the mesh relay never directly receives a UDP packet from the client. The bridge node delivers all handshake traffic for mesh hops via its radio. From the mesh relay's perspective, a standard session is being established. The mesh relay does not know the session request originated from the clearnet.

**Privacy**: entry nodes (first hop) are the only nodes that learn the client's real transport address. Mesh relays learn only the preceding mesh node's address. Deeper handshakes are routed through established sessions, so deeper nodes never see the client IP directly.

---

## 11. Cross-Domain Session Establishment

When a domain bridge node receives HANDSHAKE_DONE from a cross-domain path:

1. The bridge has already received HANDSHAKE_DONE via one domain (e.g., clearnet)
2. The bridge stores path_id → clearnet_predecessor in the clearnet domain path table
3. The bridge forwards HANDSHAKE_INIT into the next domain (e.g., mesh) via its radio
4. When the mesh relay responds, the bridge receives HANDSHAKE_RESP via mesh
5. The bridge now has a session in both domains for this cross-domain path
6. The bridge creates a cross-domain path table entry linking both sessions

Return traffic from the mesh relay arrives at the bridge with a path_id. The bridge looks up the cross-domain entry and forwards to the clearnet predecessor. The reverse is also true — return traffic from the clearnet arrives at the bridge with a path_id, and the bridge forwards to the mesh predecessor.

All of this happens without the bridge decrypting inner onion layers — only the outermost DOMAIN_BRIDGE layer is decrypted at the bridge.

---

## 12. Domain-Agnostic Handshake: Technical Basis

The handshake is a pure cryptographic protocol. It references no IP addresses, port numbers, or transport-specific fields in its cryptographic material. The session_id and node_id used in key derivation are application-layer identifiers unrelated to transport.

The BGP-X transport abstraction layer (UDP socket, WiFi mesh raw socket, LoRa serial interface, BLE GATT) handles physical delivery below the handshake layer. The handshake code is identical regardless of transport.

**Cryptographic algorithms are domain-agnostic**:

- X25519 key exchange: identical in all domains
- ChaCha20-Poly1305 encryption: identical in all domains
- HKDF-SHA256 key derivation: identical in all domains
- No domain-specific nonce spaces
- No domain-specific key derivation paths
- No domain-specific cipher variations

The same session_key protects traffic whether the session is over fiber, WiFi mesh, LoRa, BLE, or satellite.

---

## 13. Session Re-Handshake (24-Hour Key Rotation)

Sessions exceeding 24 hours SHOULD initiate a re-handshake on the same path, generating fresh ephemeral keys and new session keys.

### Procedure

1. Client generates new ephemeral keypair for this hop
2. Client sends HANDSHAKE_INIT on the existing session (session_id unchanged — this is a within-session key rotation)
3. Node responds with new node_ephemeral_pub
4. New session_key derived from new ephemeral keys
5. In-flight streams complete using old session_key
6. New streams use new session_key
7. After all streams migrated: old session_key zeroized

Domain-agnostic — identical procedure for clearnet and mesh sessions. For a mesh session established via a bridge node, the re-handshake follows the same relay chain as initial establishment.

### Re-Handshake Detection

The HANDSHAKE_INIT sequence number on an established session (non-zero session_id) signals a re-handshake intent. Nodes MUST support this and transition to the new key smoothly.

---

## 14. Handshake Timeouts by Domain

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

## 15. Replay Attack Prevention

- **Timestamp in HANDSHAKE_INIT**: ±60 seconds acceptance window
- **Session ID**: 16-byte random; collision probability negligible (2^-128)
- **Live session table lookup**: duplicate session_id rejected
- **Timestamps prevent HANDSHAKE_INIT replay across time windows**
- **After session close**: session ID freed; probability of collision with new random ID is negligible (2^-128)

The "already in use" check is a **live session table lookup only** — no persistent "used session ID" database is required. Session IDs are random 128-bit values; collisions are statistically impossible. The timestamp window provides replay protection across time.

---

## 16. Mesh Transport Handshake

The handshake protocol is unchanged for mesh transports. The same crypto, same message formats, same key derivation.

Differences:
- Transport delivery (LoRa, WiFi mesh, BLE) instead of UDP
- MESH_FRAGMENT (0x19) may be needed for low-MTU transports (particularly LoRa)
- Higher latency expected (LoRa: 100ms-5s per hop); handshake timeout extended accordingly

Implementation MUST reassemble MESH_FRAGMENTs before processing handshake messages.

---

## 17. Mesh Island Grace Period Handling

If a mesh island is disconnected from the clearnet during the grace period (e.g., during a pool curator key rotation):

1. Bridge nodes cache relevant records
2. When the island reconnects, bridge nodes propagate records to mesh nodes
3. Grace period countdown pauses while island is offline
4. Alternatively: curator may extend grace period when scheduling events during expected island downtime

Curators should avoid scheduling critical operations during expected island maintenance windows.

---

## 18. Session Teardown

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

---

## 19. Security Properties

### 19.1 Mutual Authentication

The node is authenticated: client verifies node static public key against DHT advertisement signature before initiating handshake. A MITM without the node's static private key cannot complete the handshake (cannot derive init_key, cannot decrypt HANDSHAKE_INIT, cannot produce valid HANDSHAKE_RESP).

The client is NOT authenticated by default. This is by design — nodes do not learn client identity.

### 19.2 Forward Secrecy

Session keys are derived from ephemeral keys (dh2) that are destroyed after handshake completion. Compromise of node_static_priv reveals dh1 but not dh2. Without dh2, session_key cannot be derived. Past sessions are safe under static key compromise.

### 19.3 No Nonce Reuse

Sequence numbers are strictly monotonically increasing per direction. Two independent sequence number spaces per session (inbound/outbound) prevent cross-direction nonce reuse. Session keys are never reused across sessions.

### 19.4 path_id Privacy

path_id is delivered in HANDSHAKE_DONE encrypted with session_key — only the specific node receives it. Each node knows only its own path_id → predecessor mapping. No node can reconstruct the full path from path_id alone.

For domain bridge nodes: the bridge knows path_id → {clearnet_predecessor, mesh_predecessor} but cannot see client identity or service identity.

### 19.5 Domain-Agnostic Security

Security properties are identical in all routing domains. The handshake protocol does not change based on whether the transport is UDP, WiFi mesh, LoRa, BLE, or satellite. Onion encryption, key derivation, and authentication are transport-independent.

---

## 20. Implementation Requirements

Implementations MUST:

- Generate X25519 keypairs using a cryptographically secure random number generator
- Destroy ephemeral private keys immediately after use (use zeroize in memory)
- Validate the timestamp in HANDSHAKE_INIT within ±60-second tolerance
- Pad HANDSHAKE_INIT to exactly 256 bytes
- Pad HANDSHAKE_RESP to exactly 256 bytes
- Pad HANDSHAKE_DONE to exactly 128 bytes
- Relay handshakes for hops beyond entry through established sessions
- Implement bidirectional sequence number spaces (inbound and outbound independent)
- Support session re-handshake on sessions older than 24 hours
- Support MESH_FRAGMENT reassembly before processing handshake messages on low-MTU transports
- Store cross-domain path_id entries at domain bridge nodes
- Implement geographic plausibility as OPTIONAL — applies only if jurisdiction declared

Implementations MUST NOT:

- Reuse ephemeral keys across sessions
- Log client ephemeral public keys
- Log session_id values
- Log path_id values
- Accept handshakes with timestamps outside the tolerance window
- Proceed to ESTABLISHED state without verifying confirmation hashes
- Reuse (session_key, nonce) pairs
- Use a separate cover_key (COVER uses session_key)
- Use standard memset for key zeroization (use explicit_bzero, SecureZeroMemory, or zeroize crate)
