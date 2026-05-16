# BGP-X Cryptographic Specification

**Version**: 0.1.0-draft

This document is the complete cryptographic specification for BGP-X. It defines every cryptographic operation performed by the protocol, the exact algorithm parameters, the rationale for each choice, and the implementation requirements for all routing domains.

---

## 1. Cryptographic Suite

BGP-X version 1 uses the following cryptographic suite:

| Function | Algorithm | Standard | Key/Output Size |
|---|---|---|---|
| Asymmetric key agreement | X25519 | RFC 7748 | 32-byte output |
| Asymmetric signatures | Ed25519 | RFC 8032 | 32-byte public key, 64-byte signature |
| Symmetric AEAD encryption | ChaCha20-Poly1305 | RFC 8439 | 32-byte key, 12-byte nonce, 16-byte tag |
| Cryptographic hash | BLAKE3 | BLAKE3 spec | 32-byte output (default) |
| Key derivation | HKDF-SHA256 | RFC 5869 | Variable output |
| Pseudo-random generation | OS CSPRNG | Platform-specific | — |

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

**CRITICAL**: COVER packets use session_key identically to RELAY packets. There is NO separate cover_key. This applies in ALL routing domains — clearnet, mesh, LoRa, satellite, BLE.

External observers in any domain — clearnet UDP, WiFi mesh 802.11s, LoRa radio — cannot distinguish COVER from RELAY because:
- Same ChaCha20-Poly1305 cipher
- Same session_key
- Same packet structure
- Same size class distribution

---

## 3. Algorithm Rationale

### 3.1 X25519 (Key Agreement)

X25519 is Elliptic Curve Diffie-Hellman over Curve25519.

**Why X25519**:
- Designed for safe, efficient implementation in software
- Constant-time implementation is straightforward (no conditional branches based on secret data)
- 128-bit security level (equivalent to 3072-bit RSA)
- Widely implemented and audited
- No patent concerns
- Fast: ~0.1ms per key exchange on modern hardware

**Security properties**:
- Computationally hard to reverse: finding the private key from the public key requires solving the discrete logarithm problem on Curve25519
- No small-subgroup attacks (Curve25519 has prime-order subgroup of size ≈ 2^252)

**Parameters**:
- Curve: Curve25519 (y² = x³ + 486662x² + x over GF(2^255 - 19))
- Generator point: standard Curve25519 base point
- Key size: 32 bytes (256 bits)
- Output: 32-byte shared secret

### 3.2 Ed25519 (Signatures)

Ed25519 is an EdDSA signature scheme over Twisted Edwards Curve25519.

**Why Ed25519**:
- Fast signing and verification (~0.05ms and ~0.1ms on modern hardware)
- Short signatures (64 bytes)
- Short public keys (32 bytes)
- Deterministic (no random nonce required; eliminates nonce reuse vulnerability that affects ECDSA)
- Constant-time implementation straightforward

**Use in BGP-X**:
- Signing node advertisements
- Signing pool advertisements
- Signing pool member records
- Signing exit policies
- Signing node withdrawal records
- Signing key rotation records (dual-signature)
- Signing mesh beacons
- Signing domain bridge advertisements
- Signing mesh island advertisements

**Parameters**:
- Curve: Twisted Edwards Curve25519
- Hash function: SHA-512 (internal to Ed25519; not exposed to BGP-X)
- Key size: 32-byte private seed, 32-byte public key
- Signature size: 64 bytes

### 3.3 ChaCha20-Poly1305 (AEAD Encryption)

ChaCha20-Poly1305 is an Authenticated Encryption with Associated Data (AEAD) scheme combining the ChaCha20 stream cipher with the Poly1305 MAC.

**Why ChaCha20-Poly1305**:
- Excellent software performance without hardware acceleration
- Constant-time implementation is natural (stream cipher; no S-box lookups)
- 256-bit key (full security margin; no related-key attacks known)
- Widely implemented and audited (RFC 8439)
- Chosen by TLS 1.3 as a mandatory cipher suite
- Suitable for embedded/router hardware (BGP-X's firmware target)

**Comparison to AES-GCM**:
- ChaCha20 is slower than AES-GCM on x86 hardware with AES-NI
- ChaCha20 is faster than AES-GCM on hardware without AES-NI (including most MIPS and many ARM router chips)
- ChaCha20 is more resistant to timing side-channels (stream cipher vs. S-box)
- AES-GCM has nonce reuse vulnerabilities more severe than ChaCha20-Poly1305; BGP-X's nonce construction prevents reuse regardless

**Parameters**:
- Key size: 32 bytes (256 bits)
- Nonce size: 12 bytes (96 bits)
- Authentication tag size: 16 bytes (128 bits)
- Maximum plaintext length: 2^38 - 64 bytes (not a practical limit)

### 3.4 BLAKE3 (Hashing)

**Why BLAKE3**:
- Fastest cryptographic hash function available in software
- Designed for modern CPU architectures (SIMD-optimized)
- 256-bit output (default; can be extended)
- Parallelizable for large inputs
- Verified security against known attacks (published 2020; based on BLAKE2)

**Use in BGP-X**:
- NodeID derivation: `NodeID = BLAKE3(public_key_bytes)`
- DHT key derivation: `dht_key = BLAKE3("bgpx-node-advert-v1" || node_id)`
- HKDF salt derivation
- Confirmation hashes in handshake
- Signature prehashing

### 3.5 HKDF-SHA256 (Key Derivation)

HKDF (RFC 5869) is an HMAC-based Key Derivation Function.

**Why HKDF-SHA256**:
- Standard, well-specified key derivation function
- Provably secure under standard assumptions
- Suitable for deriving multiple independent keys from shared secret material
- SHA-256 is the underlying hash; fast and widely available

**Parameters in BGP-X**:
```
HKDF-SHA256(
    IKM  = input key material (from X25519 output),
    salt = BLAKE3(context_string),
    info = purpose-specific string || session-specific data,
    L    = output key length in bytes
)
```

The BLAKE3-derived salt provides domain separation between uses of the same IKM for different purposes.

### 3.6 Why Not AES-GCM

AES-GCM requires hardware AES acceleration for constant-time operation. BGP-X mesh nodes run on embedded ARM without AES-NI. ChaCha20-Poly1305 is constant-time in software on all platforms.

### 3.7 Why Not Ed448

Key and signature sizes are larger (57 and 114 bytes vs 32 and 64 bytes). Slower. No security advantage over Ed25519 for BGP-X's threat model.

---

## 4. Key Derivation Specifications

### 4.1 Session Key

```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
dh2 = X25519(client_ephemeral_priv, node_ephemeral_priv)

session_key = HKDF-SHA256(
    IKM  = dh1 || dh2,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

### 4.2 Handshake Init Key (Stage 1 Only)

```
init_key = HKDF-SHA256(
    IKM  = dh1,
    salt = BLAKE3("bgpx-init-key-v1"),
    info = "handshake-init",
    L    = 32
)
```

### 4.3 Handshake Response Key

```
resp_key = HKDF-SHA256(
    IKM  = dh1 XOR dh2,
    salt = BLAKE3("bgpx-resp-key-v1"),
    info = session_id || "handshake-resp",
    L    = 32
)
```

The resp_key requires BOTH node static and node ephemeral keys. A MITM adversary who compromises only node_static_priv cannot forge HANDSHAKE_RESP because they cannot compute dh2 without node_ephemeral_priv.

### 4.4 Keepalive Key

```
keepalive_key = HKDF-SHA256(
    IKM  = dh1 || dh2,
    salt = BLAKE3("bgpx-keepalive-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

### 4.5 NO Cover Key

**There is no cover_key.** COVER packets use session_key identically to RELAY packets.

```
# DO NOT derive this — cover_key does not exist
# COVER uses session_key directly
```

---

## 5. Nonce Construction

ChaCha20-Poly1305 nonces are 12 bytes. BGP-X constructs nonces as:

```
nonce = 0x00000000 || sequence_number_big_endian_8_bytes
```

The first 4 bytes are always zero. The last 8 bytes are the 64-bit packet sequence number in big-endian encoding.

### 5.1 Bidirectional Sequence Number Spaces

Each session maintains TWO independent sequence number spaces:

- **Outbound**: sequence numbers for packets sent by this node to the next hop
- **Inbound**: sequence numbers for packets received from the previous hop

The common header carries the outbound sequence number. The replay detection window tracks the inbound sequence space separately.

This separation prevents nonce reuse even though the same session key is used for both directions.

### 5.2 Uniqueness Guarantee

Nonces are unique per (key, session) pair because:
- Session keys are derived from ephemeral key material (unique per session)
- Sequence numbers within a session are strictly monotonically increasing
- Two independent spaces (inbound/outbound) prevent cross-direction nonce reuse
- Therefore, no (key, nonce) pair is ever reused

### 5.3 Nonce Reuse Prevention

The daemon must NOT resume sessions after a crash. On restart, all prior sessions are abandoned and new sessions require fresh handshakes, generating fresh keys and fresh sequence number spaces.

Maximum sequence number before rekeying: 2^64 - 1. Counter never wraps in practice (at 1M packets/second: ~585,000 years). 24-hour re-handshake provides additional key rotation well before any counter concern.

---

## 6. Random Number Generation

All random values in BGP-X (session IDs, ephemeral keypairs, initial sequence numbers, padding bytes, path_id values) MUST be generated using the operating system's cryptographically secure pseudo-random number generator (CSPRNG):

- Linux: `getrandom(2)` syscall or `/dev/urandom` (post-boot)
- macOS: `SecRandomCopyBytes` or `arc4random_buf`
- Windows: `BCryptGenRandom`
- Embedded (no OS): Hardware RNG via STM32 RNG peripheral or equivalent

The Rust `ring` crate's `rand::SystemRandom` implementation is used in the reference implementation and provides these OS primitives with appropriate fallbacks.

**What MUST use CSPRNG**:
- Session ID generation (16 bytes)
- path_id generation (8 bytes, never all-zeros)
- Ephemeral X25519 keypairs
- Initial sequence number selection
- Padding byte generation
- Node advertisement nonce fields
- Pool IDs (where not derived from curator key)

**What MUST NOT use non-CSPRNG**:
- Any value that affects security properties
- Any value that might be guessable or predictable

**Minimum entropy requirements**: 256 bits of entropy before generating any key material. On embedded devices: hardware RNG MUST be tested at startup.

---

## 7. Signature Scheme Details

### 7.1 Ed25519 Signing

```
signature = Ed25519-Sign(private_key, message_bytes)
```

BGP-X uses deterministic Ed25519 (RFC 8032 §5.1) — no random nonce is needed or used.

### 7.2 Message Signing Contexts

**Node Advertisement Signature**:
```
message = canonical_json(advertisement, exclude=["signature"])
         encoded as UTF-8 bytes

signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-advert-v1" || message))
```

**Pool Advertisement Signature**:
```
message = canonical_json(pool_advertisement, exclude=["signature"])
signature = Ed25519_sign(curator_private_key, BLAKE3("bgpx-pool-advert-v1" || message))
```

**Pool Member Signature**:
```
message = BLAKE3("bgpx-pool-member-v1" || pool_id || node_id || added_at_timestamp_BE8)
signature = Ed25519_sign(curator_private_key, message)
```

**Domain Bridge Advertisement Signature**:
```
message = canonical_json(bridge_advert, exclude=["signature"])
signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-bridge-advert-v1" || message))
```

**Mesh Island Advertisement Signature**:
```
message = canonical_json(island_advert, exclude=["signature"])
signature = Ed25519_sign(curator_private_key, BLAKE3("bgpx-island-advert-v1" || message))
```

**MESH_BEACON Signature**:
```
message = NodeID || PublicKey || TransportsMask || RoutingTableSize || Timestamp ||
          BridgeCapable || ServedDomainCount || ServedDomainIDs
signature = Ed25519_sign(node_private_key, BLAKE3("bgpx-beacon-v1" || message))
```

**Node Withdrawal Signature**:
```
message = BLAKE3("bgpx-withdrawal-v1" || node_id || timestamp)
signature = Ed25519_sign(node_private_key, message)
```

### 7.3 Ed25519 Verification

```
valid = Ed25519-Verify(public_key, message_bytes, signature)
```

Verification MUST check:
- The signature is exactly 64 bytes
- The public key is exactly 32 bytes and is a valid Curve25519 point
- The signature is valid for the given message and public key

Verification MUST NOT proceed with partial results. Either the signature is valid or it is not.

---

## 8. Pool Curator Key Cryptography

Pool curator keys are Ed25519 keypairs **separate from node identity keys**.

Key properties:
- Stored in separate file with separate permissions
- Key rotation via dual-signature records
- Can be rotated without affecting node identity
- Used for: pool advertisement signing, pool member record signing, key rotation records

### 8.1 Key Rotation Cryptography

Pool key rotation uses dual-signature records:
- Both old key and new key sign the rotation record
- This provides cryptographic continuity verification
- An adversary who only has the new key cannot create a valid rotation record

See `/protocol/pool_curator_key_rotation.md` for full specification.

---

## 9. ECH Cryptography (Exit Nodes Only)

ECH (Encrypted Client Hello) uses the TLS 1.3 ECH draft standard.

Key properties:
- Key encapsulation via HPKE (X25519 + HKDF-SHA256 + AES-128-GCM)
- ECH keys published in DNS HTTPS records by destination servers
- BGP-X exit nodes use ECH when: ech_capable=true AND destination supports ECH (discovered via DNS)

**ECH key management is at the TLS layer** — not part of BGP-X key hierarchy. BGP-X exit nodes:
- Do NOT derive or manage ECH keys
- Fetch ECH configuration from DNS HTTPS records via DoH
- Use the destination server's ECH public key in TLS ClientHello construction

ECH implementation uses the TLS library's ECH support (e.g., BoringSSL, OpenSSL 3.x+).

---

## 10. Pluggable Transport Cryptography

PT operates BELOW the BGP-X crypto layer. PT is traffic analysis resistance, NOT a cryptographic security guarantee.

### 10.1 Built-In PT Cryptography

Built-in obfs4-style PT uses:
- **Elligator2**: for X25519 public key encoding (prevents identification of X25519 keys)
- **ChaCha20**: stream cipher for stream obfuscation (separate from BGP-X's ChaCha20-Poly1305)

This is separate from and IN ADDITION TO BGP-X's own ChaCha20-Poly1305 encryption. PT adds obfuscation at a different layer.

### 10.2 Security Model

PT is a traffic analysis resistance mechanism, NOT a cryptographic security guarantee:
- Effective against: port-based blocking, simple protocol fingerprinting
- NOT guaranteed effective against: DPI with ML classification, sophisticated pattern recognition

---

## 11. Security Margins

| Primitive | Key Size | Security Level | Post-Quantum Secure |
|---|---|---|---|
| X25519 | 32 bytes | ~128-bit | No |
| Ed25519 | 32-byte key, 64-byte sig | ~128-bit | No |
| ChaCha20-Poly1305 | 32-byte key | 256-bit key, 128-bit auth | Partially (symmetric key sufficient; key exchange not) |
| BLAKE3 | — | 128-bit collision / 256-bit preimage | Partially |
| HKDF-SHA256 | 32-byte output | 128-bit PRF | Partially |

**Post-quantum status**: BGP-X version 1 does NOT provide post-quantum security. The X25519 key agreement is broken by a cryptographically relevant quantum computer using Shor's algorithm. A post-quantum key exchange (ML-KEM / Kyber) is planned for a future protocol version.

---

## 12. Zeroization Requirements (MANDATORY)

Key material MUST be zeroized in this order after use:

```
After dh2 computed:
  → client_ephemeral_priv (zeroize immediately)

After session_key derived:
  → dh1 (zeroize immediately)
  → dh2 (zeroize immediately)

After handshake complete:
  → init_key (zeroize immediately)
  → resp_key (zeroize immediately)
  → node_ephemeral_priv (node side, zeroize immediately)

When session closed:
  → session_key (zeroize)
  → keepalive_key (zeroize)
```

### Zeroization Implementation

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

**Critical**: `std::mem::drop()` and `memset()` are NOT sufficient — compilers optimize them away. MUST use `zeroize` crate or equivalent OS-level `explicit_bzero`.

**Memory locking**: Implementations MUST use `mlock()` or equivalent to prevent session key swapping to disk.

---

## 13. Implementation Requirements

### MUST

- Use audited library implementations (no home-grown crypto)
- Verify Poly1305 authentication tag before processing any plaintext
- Use session_key for COVER packets (NOT a separate cover_key)
- Maintain two independent sequence number spaces per session
- Zeroize key material in the mandatory order
- Use `mlock()` or equivalent to prevent session key swapping to disk
- Support ECH at exit nodes when ech_capable=true
- Use OS CSPRNG for all random generation
- Implement domain-agnostic crypto (same algorithms for all routing domains)
- Support cross-domain sessions with identical crypto

### MUST NOT

- Implement ChaCha20, Poly1305, X25519, Ed25519, BLAKE3, or HKDF from scratch
- Use ECB, CBC, or non-AEAD cipher
- Truncate authentication tags
- Skip authentication tag verification
- Proceed with plaintext from failed decryption
- Reuse (key, nonce) pairs
- Derive a separate cover_key
- Log or persist key material in plaintext
- Create domain-specific cipher suites

---

## 14. Recommended Cryptographic Libraries

| Language | Library | Notes |
|---|---|---|
| Rust | `ring` (primary) | Audited, constant-time |
| Rust | `chacha20poly1305` (RustCrypto) | Alternative |
| Rust | `x25519-dalek` | X25519 |
| Rust | `ed25519-dalek` | Ed25519 |
| Rust | `blake3` | BLAKE3 |
| Rust | `zeroize` | Memory clearing |
| Go | `golang.org/x/crypto/chacha20poly1305` | Standard library |
| C | `libsodium` | Well-audited, widely deployed |
| Python | `cryptography` (PyCA) | OpenSSL backend |
| JavaScript | `@noble/ciphers`, `@noble/curves` | Audited pure-JS |

**Reference implementation**: Rust with `ring` crate.

All listed libraries have received independent security audits. Do NOT use home-grown implementations of any cryptographic primitive.
