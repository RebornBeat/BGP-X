# BGP-X Packet Format Specification

**Version**: 0.1.0-draft

This document specifies the complete wire encoding of all BGP-X packet types, with a focus on the onion packet structure that forms the core of the data plane.

---

## 1. Encoding Conventions

- All multi-byte integers are big-endian (network byte order)
- All byte sequences are unsigned unless stated otherwise
- Variable-length fields are preceded by a length prefix unless the length is determined by the packet's Payload Length field
- All lengths are in bytes unless stated otherwise
- Diagrams show bits from most-significant (left) to least-significant (right)

---

## 2. Transport Framing

BGP-X packets are UDP datagrams. The UDP payload contains exactly one BGP-X message per datagram.

```
[UDP Datagram]
  [IP Header]
  [UDP Header]
  [BGP-X Common Header]
  [BGP-X Message Payload]
```

There is no BGP-X framing layer above UDP. Each UDP datagram is one self-contained BGP-X message.

---

## 3. Common Header (Revisited, Full Bit Layout)

```
Byte offset:
  0        1        2        3
  ┌────────┬────────┬────────────────┐
  │Version │MsgType │   Reserved     │
  └────────┴────────┴────────────────┘
  4        5        6        7
  ┌─────────────────────────────────┐
  │          Session ID             │
  │           (16 bytes)            │
  │                                 │
  │                                 │
  └─────────────────────────────────┘
  20       21       22       23
  ┌─────────────────────────────────┐
  │        Sequence Number          │
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

---

## 4. Onion Packet Structure

The onion packet is the central data structure of BGP-X. It is the payload of a RELAY message.

### 4.1 Overview

For a path of N hops, the client constructs N nested encryption layers. The outermost layer is addressed to the entry node (hop 1). Each hop decrypts its layer and reveals the next hop address and the remaining ciphertext for subsequent hops.

```
Client constructs:
┌─────────────────────────────────────────────────┐
│  Encrypted for Entry Node (hop 1)               │
│  ┌───────────────────────────────────────────┐  │
│  │  Encrypted for Relay 1 (hop 2)            │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │  Encrypted for Relay 2 (hop 3)      │  │  │
│  │  │  ┌───────────────────────────────┐  │  │  │
│  │  │  │  Encrypted for Exit (hop 4)   │  │  │  │
│  │  │  │  [Destination + Payload]      │  │  │  │
│  │  │  └───────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 4.2 Single Onion Layer Format

Each decrypted onion layer has the following structure:

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer Header                             │
├──────────────────┬──────────────────────────────────────────┤
│  Hop Type        │  1 byte                                  │
│  Next Hop        │  40 bytes (NodeID 32B + addr 4B + port 2B + pad 2B) │
│  Stream ID       │  4 bytes                                 │
│  Flags           │  2 bytes                                 │
│  Reserved        │  2 bytes                                 │
├─────────────────────────────────────────────────────────────┤
│                    Remaining Ciphertext                     │
│              (payload for next and subsequent hops)         │
│                      (variable)                             │
└─────────────────────────────────────────────────────────────┘
```

#### Hop Type Values

| Value | Meaning |
|---|---|
| 0x01 | Relay — forward to another BGP-X node |
| 0x02 | Exit — forward to clearnet destination (IPv4) |
| 0x03 | Exit — forward to clearnet destination (IPv6) |
| 0x04 | Exit — forward to domain name |
| 0x05 | Delivery — this is the final hop (BGP-X native service) |

#### Next Hop Field Encoding

The Next Hop field is always 40 bytes:

For Hop Type 0x01 (Relay to another BGP-X node):

| Bytes | Content |
|---|---|
| 0–31 | NodeID of next BGP-X relay node |
| 32–35 | IPv4 address of next node |
| 36–37 | UDP port of next node |
| 38–39 | Reserved (0x0000) |

For Hop Type 0x02 (Exit to IPv4):

| Bytes | Content |
|---|---|
| 0–3 | Destination IPv4 address |
| 4–5 | Destination TCP/UDP port |
| 6–39 | Zero-padded |

For Hop Type 0x03 (Exit to IPv6):

| Bytes | Content |
|---|---|
| 0–15 | Destination IPv6 address |
| 16–17 | Destination TCP/UDP port |
| 18–39 | Zero-padded |

For Hop Type 0x04 (Exit to domain name):

| Bytes | Content |
|---|---|
| 0 | Domain name length (1–38 bytes) |
| 1–N | Domain name (ASCII, no null terminator) |
| N+1–37 | Zero-padded |
| 38–39 | Destination TCP/UDP port |

For Hop Type 0x05 (BGP-X native service delivery):

| Bytes | Content |
|---|---|
| 0–31 | ServiceID (Ed25519 public key of destination service) |
| 32–39 | Zero-padded |

### 4.3 Encryption of Each Layer

Each layer is encrypted with ChaCha20-Poly1305 using:

- **Key**: Session key derived during handshake for this specific hop
- **Nonce**: 12 bytes, constructed as: [4 bytes zero-pad] [8 bytes packet sequence number]
- **AAD (Additional Authenticated Data)**: The 32-byte BGP-X common header (excluding payload)
- **Plaintext**: Layer Header + Remaining Ciphertext
- **Ciphertext**: Encrypted plaintext + 16-byte Poly1305 authentication tag

```
Ciphertext = ChaCha20-Poly1305-Encrypt(
    key = session_key[hop_i],
    nonce = [0x00000000] || sequence_number,
    aad = bgpx_common_header,
    plaintext = layer_header || remaining_ciphertext
)
```

The authentication tag is appended to the ciphertext. Decryption MUST verify the authentication tag before any processing of the plaintext.

### 4.4 Onion Construction Algorithm

```
function construct_onion(payload, path, session_keys):
    # path = [entry, relay_1, ..., relay_n, exit]
    # session_keys = [key_entry, key_relay_1, ..., key_exit]

    # Start with the innermost layer (exit hop)
    current = construct_layer(
        hop_type = exit_type(path[-1]),
        next_hop = destination,
        payload = payload,
        key = session_keys[-1],
        sequence = sequence_number
    )

    # Wrap each subsequent layer from exit toward entry
    for i in range(len(path) - 2, -1, -1):
        current = construct_layer(
            hop_type = 0x01,  # relay
            next_hop = path[i + 1],
            payload = current,
            key = session_keys[i],
            sequence = sequence_number
        )

    return current
```

### 4.5 Onion Unwrapping Algorithm (at each relay)

```
function unwrap_onion(packet, session_key, sequence_number):
    # Attempt decryption
    plaintext = ChaCha20-Poly1305-Decrypt(
        key = session_key,
        nonce = [0x00000000] || sequence_number,
        aad = packet.common_header,
        ciphertext = packet.payload
    )

    # If decryption fails (auth tag mismatch), drop silently
    if plaintext is None:
        return DROP

    # Parse layer header
    hop_type = plaintext[0]
    next_hop = plaintext[1:41]
    remaining = plaintext[49:]  # skip header fields

    return (hop_type, next_hop, remaining)
```

---

## 5. Handshake Packet Formats

See `/protocol/handshake.md` for handshake message formats. Below is a summary of the wire format for each handshake message.

### HANDSHAKE_INIT Payload

```
┌───────────────────────────────────────────────────────────────┐
│              Client Ephemeral Public Key (32 bytes)           │
│                     (X25519 key)                              │
├───────────────────────────────────────────────────────────────┤
│              Encrypted Init Payload (variable)                │
│  Contains: timestamp, supported versions, extension flags     │
│  Encrypted with: ChaCha20-Poly1305, key = HKDF(             │
│      IKM = X25519(client_ephemeral_priv, node_static_pub),   │
│      info = "bgpx-handshake-init"                            │
│  )                                                            │
└───────────────────────────────────────────────────────────────┘
```

### HANDSHAKE_RESP Payload

```
┌───────────────────────────────────────────────────────────────┐
│              Node Ephemeral Public Key (32 bytes)             │
│                     (X25519 key)                              │
├───────────────────────────────────────────────────────────────┤
│              Encrypted Resp Payload (variable)                │
│  Contains: session ID, selected version, extension flags      │
│  Encrypted with: ChaCha20-Poly1305, key = HKDF(             │
│      IKM = X25519(node_ephemeral_priv, client_ephemeral_pub) │
│           XOR                                                 │
│           X25519(node_static_priv, client_ephemeral_pub),    │
│      info = "bgpx-handshake-resp"                            │
│  )                                                            │
└───────────────────────────────────────────────────────────────┘
```

### HANDSHAKE_DONE Payload

```
┌───────────────────────────────────────────────────────────────┐
│              Encrypted Done Payload (variable)                │
│  Contains: confirmation hash, initial path segment            │
│  Encrypted with: ChaCha20-Poly1305, session key              │
└───────────────────────────────────────────────────────────────┘
```

---

## 6. Node Advertisement Wire Format

Node advertisements stored in the DHT are JSON documents encoded as UTF-8 and prepended with a 4-byte big-endian length prefix when transmitted.

```
┌───────────────────────────────────────────────────────────────┐
│              Record Length (4 bytes, big-endian)              │
├───────────────────────────────────────────────────────────────┤
│              Canonical JSON (variable)                        │
│        (UTF-8, compact, keys sorted alphabetically)           │
└───────────────────────────────────────────────────────────────┘
```

Maximum advertisement size: **4096 bytes** (including length prefix).

---

## 7. DHT Key Derivation

DHT storage keys for node advertisements are derived as:

```
dht_key = BLAKE3("bgpx-node-advert" || node_id)
```

This namespacing prevents collisions between different types of DHT records.

For future record types, a different prefix is used:

```
dht_key = BLAKE3("bgpx-<record-type>" || record_identifier)
```

---

## 8. Padding and Size Normalization

BGP-X implementations SHOULD normalize packet sizes to resist traffic analysis. All RELAY packets SHOULD be padded to a fixed size or one of a small set of allowed sizes before encryption.

Recommended size classes (total UDP payload including BGP-X header):

| Size Class | Total UDP Payload |
|---|---|
| Small | 256 bytes |
| Medium | 512 bytes |
| Large | 1024 bytes |
| Maximum | 1280 bytes |

Padding MUST be added inside the innermost encryption layer (before onion wrapping) so that it is encrypted and indistinguishable from payload data to any relay node.

---

## 9. Endianness and Encoding Reference

| Type | Encoding |
|---|---|
| uint8 | 1 byte, unsigned |
| uint16 | 2 bytes, big-endian, unsigned |
| uint32 | 4 bytes, big-endian, unsigned |
| uint64 | 8 bytes, big-endian, unsigned |
| bytes[N] | N bytes, no encoding transformation |
| NodeID | 32 bytes, BLAKE3 output |
| PublicKey | 32 bytes, Ed25519 or X25519 public key |
| Signature | 64 bytes, Ed25519 signature |
| Timestamp | ISO 8601 string in JSON; uint64 Unix epoch seconds on wire |

---

## 10. Test Vectors

Test vectors for onion encryption, handshake messages, and node advertisement signing will be published in `/security/audit_plan.md` and as machine-readable JSON in `/protocol/test_vectors/` once the reference implementation is underway.

All implementations MUST pass the published test vectors before being considered compliant.
