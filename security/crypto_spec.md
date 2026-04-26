# BGP-X Cryptographic Specification

**Version**: 0.1.0-draft

---

## 1. Cryptographic Suite

| Function | Algorithm | Standard |
|---|---|---|
| Key agreement | X25519 | RFC 7748 |
| Signatures | Ed25519 | RFC 8032 |
| AEAD encryption | ChaCha20-Poly1305 | RFC 8439 |
| Hash | BLAKE3 | BLAKE3 paper |
| KDF | HKDF-SHA256 | RFC 5869 |
| PRNG | OS CSPRNG | OS-specific |

---

## 2. Cross-Domain Cryptographic Properties

### 2.1 Domain-Agnostic Crypto

BGP-X uses an **identical cryptographic suite** regardless of which routing domain a session is in. Whether a session is established over UDP/IP, WiFi 802.11s mesh, LoRa radio, Bluetooth BLE, or satellite:

- Identical X25519 key exchange
- Identical HKDF-SHA256 key derivation with identical salts and info strings
- Identical ChaCha20-Poly1305 session encryption
- Identical HANDSHAKE_INIT / HANDSHAKE_RESP / HANDSHAKE_DONE format
- Identical nonce construction (sequence number based)
- Identical replay protection (sliding window)
- Identical zeroization ordering

There are NO domain-specific keys, no domain-specific key derivation, and no domain-specific cipher suite.

### 2.2 Cross-Domain Onion Encryption

DOMAIN_BRIDGE hops (hop_type 0x06) are encrypted identically to RELAY hops:

- Client establishes a session with the bridge node (X25519 handshake via the bridge's clearnet endpoint)
- The DOMAIN_BRIDGE onion layer is encrypted using ChaCha20-Poly1305 with the bridge node's session_key
- The remaining inner layers (encrypted for mesh hops) are forwarded opaque by the bridge node without modification
- No re-encryption occurs at domain boundaries
- The bridge node cannot decrypt inner layers (it does not have the mesh session keys)

### 2.3 No Domain-Specific Keys

There are no "clearnet keys" or "mesh keys". Each node-to-node session has one session_key derived from the X25519 handshake. That session_key is used for all traffic on that session regardless of which routing domain the session traverses.

### 2.4 Cover Traffic (Domain-Agnostic)

COVER uses session_key identically in all routing domains. External observers in any domain — clearnet UDP, WiFi mesh 802.11s, LoRa radio — cannot distinguish COVER from RELAY.

---

## 3. Algorithm Rationale

### Why X25519

- Constant-time scalar multiplication (no timing side channels)
- Short keys (32 bytes) — suitable for constrained hardware
- No cofactor issues in this application
- Widely implemented and audited

### Why Ed25519

- Compact signatures (64 bytes)
- Deterministic (no per-signature randomness needed)
- Fast verification
- No bad-key attacks when using standard Ristretto255 representations

### Why ChaCha20-Poly1305

- Constant-time on all platforms (no AES-NI requirement)
- Fast on embedded ARM hardware (key requirement for mesh nodes)
- No padding oracle issues (AEAD)
- Simple nonce management (sequential counter)

### Why BLAKE3

- Fast on all platforms including embedded
- Parallel hashing capability
- 256-bit output default (suitable for all BGP-X uses)
- Simple API

### Why Not AES-GCM

AES-GCM requires hardware AES acceleration for constant-time operation. BGP-X mesh nodes run on embedded ARM without AES-NI. ChaCha20-Poly1305 is constant-time in software on all platforms.

### Why Not Ed448

Key and signature sizes are larger (57 and 114 bytes vs 32 and 64 bytes). Slower. No security advantage over Ed25519 for BGP-X's threat model.

---

## 4. Key Derivation Specifications

### Session Key

```
IKM        = dh1 || dh2
            where dh1 = X25519(client_ephemeral_priv, node_static_pub)
                  dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
salt       = BLAKE3("bgpx-session-key-v1")  [32 bytes]
info       = session_id || node_id          [16 + 32 = 48 bytes]
L          = 32
Output     = session_key                    [32 bytes]
```

### Keepalive Key

```
IKM        = dh1 || dh2  (same as session key derivation)
salt       = BLAKE3("bgpx-keepalive-key-v1")
info       = session_id || node_id
L          = 32
Output     = keepalive_key  [32 bytes]
```

### Init Key (HANDSHAKE_INIT encryption)

```
IKM        = dh1  (computed from client_ephemeral and node_static only)
salt       = BLAKE3("bgpx-init-key-v1")
info       = "handshake-init"
L          = 32
Output     = init_key  [32 bytes]
```

### Response Key (HANDSHAKE_RESP encryption)

```
IKM        = dh1 XOR dh2  (bitwise XOR of 32-byte values)
salt       = BLAKE3("bgpx-resp-key-v1")
info       = session_id || "handshake-resp"
L          = 32
Output     = resp_key  [32 bytes]
```

The resp_key requires BOTH node static and node ephemeral keys. A MITM adversary who compromises only node_static_priv cannot forge HANDSHAKE_RESP because they cannot compute dh2 without node_ephemeral_priv.

---

## 5. Nonce Construction

```
nonce = 0x00000000 || sequence_number  (12 bytes total)
      = 4 zero bytes || 8 bytes big-endian sequence number
```

Maximum sequence number before rekeying: 2^64 - 1. Counter never wraps in practice (at 1M packets/second: ~585,000 years). 24-hour re-handshake provides additional key rotation well before any counter concern.

---

## 6. Signature Scheme

### Node Advertisement Signature

```
message = canonical_json(advertisement, exclude=["signature"])
         encoded as UTF-8 bytes

signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-advert-v1" || message))
```

Prehashing with BLAKE3 prevents length-extension issues and reduces signature size overhead.

### Pool Advertisement Signature

```
message = canonical_json(pool_advertisement, exclude=["signature"])
signature = Ed25519_sign(curator_private_key, BLAKE3("bgpx-pool-advert-v1" || message))
```

### Pool Member Signature

```
message = BLAKE3("bgpx-pool-member-v1" || pool_id || node_id || added_at_timestamp_BE8)
signature = Ed25519_sign(curator_private_key, message)
```

### Domain Bridge Advertisement Signature

```
message = canonical_json(bridge_advert, exclude=["signature"])
signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-bridge-advert-v1" || message))
```

### Mesh Island Advertisement Signature

```
message = canonical_json(island_advert, exclude=["signature"])
signature = Ed25519_sign(curator_private_key, BLAKE3("bgpx-island-advert-v1" || message))
```

### MESH_BEACON Signature

```
message = NodeID || PublicKey || TransportsMask || RoutingTableSize || Timestamp ||
          BridgeCapable || ServedDomainCount || ServedDomainIDs
signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-beacon-v1" || message))
```

---

## 7. Pool Curator Key Cryptography

Curator keys are Ed25519 keypairs separate from any node key. Stored in separate file with separate permissions. Key rotation via dual-signature records (see `/protocol/pool_curator_key_rotation.md`).

---

## 8. ECH Cryptography (Exit Nodes Only)

ECH (Encrypted Client Hello) uses the TLS 1.3 ECH draft standard. Key encapsulation via HPKE (X25519 + HKDF-SHA256 + AES-128-GCM). ECH keys published in DNS HTTPS records by destination servers. BGP-X exit nodes use ECH when: ech_capable=true AND destination supports ECH (discovered via DNS). Not applicable to mesh island service endpoints (no TLS ClientHello to public internet).

---

## 9. Random Number Generation

All CSPRNG usage: OS-provided randomness only.

| Platform | Source |
|---|---|
| Linux | getrandom() syscall (kernel CSPRNG) |
| macOS | getentropy() or /dev/urandom |
| Windows | BCryptGenRandom() |
| Embedded (no OS) | Hardware RNG via STM32 RNG peripheral or equivalent |

Minimum entropy requirements: 256 bits of entropy before generating any key material. On embedded devices: hardware RNG MUST be tested at startup.

---

## 10. Security Margins

| Primitive | Key size | Security margin |
|---|---|---|
| X25519 | 32 bytes | ~128-bit equivalent |
| Ed25519 | 32-byte key, 64-byte sig | ~128-bit equivalent |
| ChaCha20-Poly1305 | 32-byte key | 256-bit key, 128-bit auth |
| BLAKE3 | — | 128-bit collision resistance |
| HKDF-SHA256 | 32-byte output | 256-bit PRF |

All primitives provide at least 128-bit security against classical computers. BGP-X does not claim post-quantum resistance in v1.

---

## 11. Zeroization Requirements

```rust
// Correct zeroization using the zeroize crate
use zeroize::Zeroize;

let mut session_key: [u8; 32] = derive_session_key(...);
// ... use session_key ...
session_key.zeroize();  // MANDATORY on session close

let mut dh1: [u8; 32] = x25519(client_ephemeral_priv, node_static_pub);
let mut dh2: [u8; 32] = x25519(client_ephemeral_priv, node_ephemeral_pub);
let session_key = derive_session_key(&dh1, &dh2);
dh1.zeroize();  // MANDATORY immediately after session_key derived
dh2.zeroize();  // MANDATORY immediately after session_key derived
```

`std::mem::drop()` and `memset()` are NOT sufficient — compilers optimize them away. MUST use `zeroize` crate or equivalent OS-level `explicit_bzero`.

---

## 12. Recommended Libraries

| Function | Rust | C |
|---|---|---|
| X25519 | `x25519-dalek` | `libsodium` |
| Ed25519 | `ed25519-dalek` | `libsodium` |
| ChaCha20-Poly1305 | `chacha20poly1305` (RustCrypto) | `libsodium` |
| BLAKE3 | `blake3` | `BLAKE3/c` |
| HKDF | `hkdf` (RustCrypto) | `libsodium` |
| Zeroize | `zeroize` | `libsodium sodium_memzero()` |

All listed libraries have received independent security audits. Do NOT use home-grown implementations of any cryptographic primitive.
