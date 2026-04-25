# BGP-X Encryption Layers Specification

**Version**: 0.1.0-draft

This document provides a complete specification of the encryption scheme used in BGP-X — covering the onion layering construction, the cryptographic primitives for each layer, nonce construction, key derivation, and implementation requirements.

---

## 1. Overview

BGP-X uses **layered symmetric encryption** (onion encryption) to ensure that no relay node can decrypt more than its own layer of a packet. Each layer is independently encrypted using a key shared only between the client and that specific hop.

The encryption scheme must satisfy:

- **Unlinkability**: A relay node cannot correlate an incoming packet with an outgoing packet based on ciphertext content
- **Unforgeability**: A relay node cannot inject or modify packets without detection
- **Forward secrecy**: Compromise of long-term node keys does not expose past session traffic
- **Efficiency**: Encryption and decryption must be fast enough for wire-speed operation

---

## 2. Cryptographic Primitive: ChaCha20-Poly1305

BGP-X uses **ChaCha20-Poly1305** (RFC 8439) as the AEAD (Authenticated Encryption with Associated Data) cipher for all onion layers.

### 2.1 Why ChaCha20-Poly1305

| Property | Detail |
|---|---|
| Performance without AES-NI | ChaCha20 is faster than AES-GCM on hardware without AES hardware acceleration (common in embedded/router hardware targeted by BGP-X firmware) |
| Performance with AES-NI | Slightly slower than AES-GCM on x86 with AES-NI, but acceptably so |
| Side-channel resistance | ChaCha20 is designed to be constant-time; no S-box lookups that could leak via cache timing |
| Key size | 256-bit key |
| Nonce size | 96-bit (12-byte) nonce |
| Tag size | 128-bit (16-byte) Poly1305 authentication tag |
| Authentication | Poly1305 provides full AEAD — ciphertext integrity is verified before decryption |

### 2.2 Security parameters

- **Key**: 32 bytes (256 bits), unique per (session, hop)
- **Nonce**: 12 bytes, constructed from sequence number (see Section 4)
- **Tag**: 16 bytes, appended to ciphertext
- **AAD**: 32-byte BGP-X common header

---

## 3. Key Hierarchy

BGP-X uses a hierarchy of keys derived from the handshake:

```
Handshake Key Material
    │
    ├── Session Key (32 bytes)
    │       └── Used for: onion layer encryption at this hop
    │
    ├── KeepalIve Key (32 bytes)
    │       └── Used for: KEEPALIVE message authentication
    │
    └── Cover Key (32 bytes)
            └── Used for: COVER message padding generation
```

All keys are derived from the handshake's shared secret using HKDF-SHA256 with distinct info strings, preventing key reuse across purposes.

### 3.1 Session key derivation (recap from handshake spec)

```
input_key_material = X25519(client_ephemeral, node_static) || X25519(client_ephemeral, node_ephemeral)

session_key = HKDF-SHA256(
    IKM  = input_key_material,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

### 3.2 Auxiliary key derivation

```
keepalive_key = HKDF-SHA256(
    IKM  = input_key_material,
    salt = BLAKE3("bgpx-keepalive-key-v1"),
    info = session_id || node_id,
    L    = 32
)

cover_key = HKDF-SHA256(
    IKM  = input_key_material,
    salt = BLAKE3("bgpx-cover-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

---

## 4. Nonce Construction

ChaCha20-Poly1305 requires a 12-byte nonce that MUST be unique per (key, message) pair.

BGP-X constructs nonces deterministically from the packet sequence number:

```
nonce = [0x00, 0x00, 0x00, 0x00] || sequence_number_as_8_bytes_big_endian
```

The first 4 bytes are always zero. The last 8 bytes are the 64-bit packet sequence number in big-endian format.

### 4.1 Nonce uniqueness guarantee

Since each session uses a unique session key, and sequence numbers within a session are strictly monotonically increasing:

- No two packets in the same session will have the same sequence number
- No two packets with the same sequence number can use the same key (different sessions = different keys)
- Nonce uniqueness is guaranteed across all BGP-X packets for a given key

### 4.2 Nonce reuse prevention

Nonce reuse with the same key catastrophically breaks ChaCha20-Poly1305 security (an attacker can recover the keystream). BGP-X prevents nonce reuse by:

- Using the sequence number as a nonce — sequence numbers are strictly increasing
- The session table tracks the maximum outbound sequence number
- Session keys are never reused across sessions (fresh handshake = fresh keys)
- On crash recovery, sessions are NOT resumed (new handshake required) to avoid sequence number ambiguity

---

## 5. Onion Layer Construction (Client Side)

The client constructs the full onion packet before transmitting any data. This is a layered encryption process, working from the innermost (exit) layer outward to the outermost (entry) layer.

### 5.1 Input

```
path         = [entry_node, relay_1, ..., relay_n, exit_node]
session_keys = [key_entry, key_relay_1, ..., key_relay_n, key_exit]
payload      = application data (plaintext)
destination  = clearnet address or BGP-X ServiceID
sequence     = current outbound sequence number
session_id   = 16-byte session identifier
```

### 5.2 Construction algorithm

```
function construct_onion_packet(path, session_keys, payload, destination, sequence, session_id):

    N = len(path)

    # === Step 1: Build innermost layer (exit node) ===
    inner_plaintext = LayerHeader {
        hop_type:  determine_exit_hop_type(destination),
        next_hop:  encode_destination(destination),
        flags:     0x0000,
        stream_id: current_stream_id,
        reserved:  0x0000
    } || payload

    inner_plaintext = pad_to_size_class(inner_plaintext)

    inner_header = CommonHeader {
        version:         0x01,
        msg_type:        0x01,
        reserved:        0x0000,
        session_id:      session_id,
        sequence_number: sequence,
        payload_length:  len(inner_plaintext) + 16  // +16 for Poly1305 tag
    }

    inner_ciphertext = chacha20_poly1305_encrypt(
        key   = session_keys[N-1],
        nonce = construct_nonce(sequence),
        aad   = inner_header.serialize(),
        pt    = inner_plaintext
    )

    current_ciphertext = inner_ciphertext

    # === Step 2: Wrap each subsequent layer from exit-1 toward entry ===
    for i in range(N-2, -1, -1):

        node = path[i]
        key  = session_keys[i]

        layer_plaintext = LayerHeader {
            hop_type:  0x01,  // RELAY
            next_hop:  encode_bgpx_node(path[i+1]),
            flags:     0x0000,
            stream_id: current_stream_id,
            reserved:  0x0000
        } || current_ciphertext

        layer_plaintext = pad_to_size_class(layer_plaintext)

        layer_header = CommonHeader {
            version:         0x01,
            msg_type:        0x01,
            reserved:        0x0000,
            session_id:      session_id,
            sequence_number: sequence,
            payload_length:  len(layer_plaintext) + 16
        }

        layer_ciphertext = chacha20_poly1305_encrypt(
            key   = key,
            nonce = construct_nonce(sequence),
            aad   = layer_header.serialize(),
            pt    = layer_plaintext
        )

        current_ciphertext = layer_ciphertext

    return current_ciphertext  # This is the outermost layer, sent to entry node
```

### 5.3 Size class padding

Before encrypting each layer, the plaintext is padded to the nearest size class:

| Size Class | Threshold |
|---|---|
| 256 bytes | plaintext <= 224 bytes (after padding, reaches 256 before tag) |
| 512 bytes | plaintext <= 480 bytes |
| 1024 bytes | plaintext <= 992 bytes |
| 1248 bytes (max) | all larger plaintexts |

Padding is random bytes appended to the plaintext. The padding length is encoded in the last 2 bytes of the padded block so the receiver can strip it.

All padded plaintexts have the structure:

```
[original content][random padding bytes][padding_length: uint16]
```

---

## 6. Onion Layer Decryption (Relay Node Side)

Each relay node decrypts exactly one layer of the onion.

```
function decrypt_onion_layer(session_key, header, ciphertext, sequence):

    # Verify sequence number (replay protection)
    if not check_sequence(sequence):
        return DROP

    # Construct nonce
    nonce = construct_nonce(sequence)

    # Decrypt
    plaintext = chacha20_poly1305_decrypt(
        key        = session_key,
        nonce      = nonce,
        aad        = header.serialize(),
        ciphertext = ciphertext
    )

    if plaintext is None:
        return DROP  # Authentication failure

    # Strip padding
    padding_length = uint16_from_bytes(plaintext[-2:])
    if padding_length > len(plaintext) - 2:
        return DROP  # Invalid padding length
    plaintext = plaintext[:len(plaintext) - 2 - padding_length]

    # Parse layer header
    layer = parse_layer_header(plaintext)
    if layer is None:
        return DROP

    return layer
```

---

## 7. Encryption at the Gateway (Exit Node)

The exit node decrypts the final onion layer and receives the plaintext application data. At this point, the exit node has:

- The destination address (clearnet IP/domain or BGP-X ServiceID)
- The plaintext application payload (e.g., HTTP request bytes)

The exit node forwards this plaintext to the destination over the public internet. If the destination uses HTTPS (strongly recommended), the application data was already encrypted by TLS before being wrapped in BGP-X onion layers, providing defense in depth:

```
[TLS-encrypted application data]
    wrapped in
[BGP-X onion layer N (exit)]
    wrapped in
[BGP-X onion layer N-1 (relay)]
    ...
    wrapped in
[BGP-X onion layer 1 (entry)]
```

The exit node strips all BGP-X layers but cannot read the TLS-encrypted content.

---

## 8. Return Path Encryption

Responses from the destination travel back through the path in reverse.

### 8.1 Response forwarding by exit node

When the exit node receives a response from the destination:

1. It does NOT re-encrypt with onion layers (it does not have the client's keys for the other nodes)
2. Instead, it forwards the response back over the established return session to the previous relay

Return sessions are bidirectional — the session key established during the handshake is used for both directions (with separate sequence number spaces for inbound and outbound).

### 8.2 Inbound vs outbound sequence numbers

Each session maintains two sequence number spaces:

- **Outbound**: Sequence numbers for packets sent by this node to the next hop
- **Inbound**: Sequence numbers for packets received from the previous hop

This prevents sequence number collisions between the two directions of traffic.

---

## 9. Cover Traffic Encryption

COVER packets (Msg Type 0x11) use the same encryption scheme as RELAY packets, using the `cover_key` derived during handshake rather than the `session_key`.

This means COVER packets are cryptographically distinguishable from RELAY packets only by the session key used — which only the recipient can determine. To an outside observer or relay node, COVER packets are indistinguishable from RELAY packets in size and timing.

---

## 10. Cryptographic Agility

BGP-X supports cryptographic agility through protocol versioning. The current cryptographic suite (version 1) is:

| Function | Algorithm |
|---|---|
| Key agreement | X25519 |
| KDF | HKDF-SHA256 |
| AEAD | ChaCha20-Poly1305 |
| Hash | BLAKE3 |
| Signatures | Ed25519 |

Future protocol versions MAY introduce new cryptographic suites. When a new suite is introduced:

- It is identified by a suite ID negotiated during the handshake
- Old suites remain supported for the deprecation period
- Implementations MUST NOT mix ciphers within a single session

Suite ID is included in the HANDSHAKE_INIT extension flags and HANDSHAKE_RESP selected extension flags.

---

## 11. Implementation Requirements

Implementations MUST:

- Use a cryptographic library with constant-time ChaCha20-Poly1305 implementation
- Verify the Poly1305 authentication tag before processing any decrypted plaintext
- Zeroize session keys from memory when a session is closed
- Lock session key memory with `mlock()` or equivalent to prevent swapping (RECOMMENDED)
- Use a CSPRNG for all random number generation (session IDs, sequence number initialization, padding bytes)
- Never log session keys or derived key material
- Never reuse a (key, nonce) pair

Implementations MUST NOT:

- Implement their own ChaCha20, Poly1305, X25519, Ed25519, or HKDF from scratch — use audited libraries
- Use ECB mode, CBC mode, or any non-AEAD cipher for onion layers
- Truncate authentication tags
- Skip authentication tag verification as an optimization
- Proceed with plaintext from a failed decryption (even partial)

---

## 12. Recommended Cryptographic Libraries

| Language | Library | Notes |
|---|---|---|
| Rust | `ring` or `chacha20poly1305` (RustCrypto) | Both are audited and constant-time |
| Go | `golang.org/x/crypto/chacha20poly1305` | Standard library extension |
| C | `libsodium` | Well-audited; widely deployed |
| Python | `cryptography` (PyCA) | Uses OpenSSL backend |
| JavaScript | `@noble/ciphers` | Audited pure-JS implementation |

The reference implementation uses Rust with the `ring` crate.
