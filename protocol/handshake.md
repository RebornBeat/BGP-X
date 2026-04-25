# BGP-X Handshake Protocol Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X session handshake — the process by which a client establishes a shared session key with each node in a path.

---

## 1. Overview

The BGP-X handshake is a 3-message exchange between a client and a single node. It is performed independently for each node in the path, in order from entry to exit.

The handshake establishes:

- A shared session key (used for all subsequent DATA and RELAY messages on this session)
- Mutual authentication of node identity (client verifies it is talking to the correct node by public key)
- Forward secrecy (session key is derived from ephemeral keys; compromise of long-term keys does not expose past sessions)

The handshake does NOT establish:

- Client identity to the node (clients are anonymous; nodes do not learn client identity)
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
| Signature (node advertisement) | Ed25519 |

---

## 3. Key Types

### Client ephemeral keypair

Generated fresh for each session (and destroyed after session establishment).

```
(client_ephemeral_priv, client_ephemeral_pub) = X25519_keygen()
```

### Node static keypair

The node's long-term identity keypair. Public key is published in the node's DHT advertisement. Private key never leaves the node.

```
(node_static_priv, node_static_pub)
```

### Node ephemeral keypair

Generated fresh for each handshake response. Destroyed after session establishment.

```
(node_ephemeral_priv, node_ephemeral_pub) = X25519_keygen()
```

---

## 4. Session Key Derivation

The session key is derived via a two-stage ECDH:

**Stage 1**: Client ephemeral × Node static

```
dh1 = X25519(client_ephemeral_priv, node_static_pub)
    = X25519(node_static_priv, client_ephemeral_pub)
```

This allows the client to encrypt HANDSHAKE_INIT using only the node's public key (from the DHT advertisement) before the node responds.

**Stage 2**: Client ephemeral × Node ephemeral

```
dh2 = X25519(client_ephemeral_priv, node_ephemeral_pub)
    = X25519(node_ephemeral_priv, client_ephemeral_pub)
```

This adds forward secrecy: compromise of the node's static key alone does not expose the session.

**Key material**:

```
input_key_material = dh1 || dh2
```

**Session key derivation**:

```
session_key = HKDF-SHA256(
    IKM  = input_key_material,
    salt = BLAKE3("bgpx-session-key-v1"),
    info = session_id || node_id,
    L    = 32
)
```

The session_id is the 16-byte random value from the common header, established in HANDSHAKE_INIT.

---

## 5. Handshake Messages

### 5.1 HANDSHAKE_INIT (Client → Node)

The client sends HANDSHAKE_INIT to the node to initiate session establishment.

#### Pre-conditions

- Client has retrieved and verified the node's DHT advertisement
- Client has generated a fresh ephemeral X25519 keypair
- Client has generated a random 16-byte session_id

#### Encryption of HANDSHAKE_INIT payload

The client cannot use the session_key yet (it requires the node's ephemeral key). Instead, it encrypts the init payload using an intermediate key derived from Stage 1 only:

```
init_key = HKDF-SHA256(
    IKM  = dh1,
    salt = BLAKE3("bgpx-init-key-v1"),
    info = "handshake-init",
    L    = 32
)
```

#### HANDSHAKE_INIT Payload (after decryption)

```
┌───────────────────────────────────────────────────────────────┐
│      Client Ephemeral Public Key (32 bytes, X25519)           │
├───────────────────────────────────────────────────────────────┤
│      Session ID (16 bytes, random)                            │
├───────────────────────────────────────────────────────────────┤
│      Timestamp (8 bytes, Unix epoch seconds, uint64)          │
├───────────────────────────────────────────────────────────────┤
│      Protocol Version (2 bytes)                               │
├───────────────────────────────────────────────────────────────┤
│      Extension Flags (4 bytes)                                │
├───────────────────────────────────────────────────────────────┤
│      Padding (variable, zero bytes to normalize size)         │
└───────────────────────────────────────────────────────────────┘
```

HANDSHAKE_INIT MUST be padded to exactly 256 bytes (including the ChaCha20-Poly1305 16-byte tag).

#### Extension Flags

| Bit | Extension |
|---|---|
| 0 | Cover traffic support |
| 1 | Pluggable transport support |
| 2 | Stream multiplexing |
| 3 | Path quality reporting |
| 4–31 | Reserved |

#### Node validation of HANDSHAKE_INIT

The node MUST:

1. Decrypt the init payload using init_key
2. Verify the timestamp is within 60 seconds of the node's current time (clock skew tolerance)
3. Check the session_id is not already in use (replay detection)
4. Record the client's ephemeral public key and session_id

---

### 5.2 HANDSHAKE_RESP (Node → Client)

The node responds with its ephemeral public key and a confirmation.

#### HANDSHAKE_RESP Payload

The response is encrypted using the resp_key:

```
resp_key = HKDF-SHA256(
    IKM  = dh1 XOR dh2,
    salt = BLAKE3("bgpx-resp-key-v1"),
    info = session_id || "handshake-resp",
    L    = 32
)
```

Note: This key requires both the node's static private key AND its ephemeral private key, so it cannot be decrypted by anyone who does not have the node ephemeral private key.

#### HANDSHAKE_RESP Payload (after decryption)

```
┌───────────────────────────────────────────────────────────────┐
│      Node Ephemeral Public Key (32 bytes, X25519)             │
├───────────────────────────────────────────────────────────────┤
│      Session ID Echo (16 bytes)                               │
│      (MUST match session_id from HANDSHAKE_INIT)              │
├───────────────────────────────────────────────────────────────┤
│      Timestamp (8 bytes)                                      │
├───────────────────────────────────────────────────────────────┤
│      Selected Protocol Version (2 bytes)                      │
├───────────────────────────────────────────────────────────────┤
│      Accepted Extension Flags (4 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│      Confirmation Hash (32 bytes)                             │
│      BLAKE3("bgpx-resp-confirm" || session_key || session_id) │
├───────────────────────────────────────────────────────────────┤
│      Padding (variable)                                       │
└───────────────────────────────────────────────────────────────┘
```

HANDSHAKE_RESP MUST be padded to exactly 256 bytes.

#### Client validation of HANDSHAKE_RESP

The client MUST:

1. Compute dh2 using the received node ephemeral public key
2. Derive the session_key using both dh1 and dh2
3. Decrypt and verify the HANDSHAKE_RESP payload
4. Verify session_id echo matches the sent session_id
5. Verify the confirmation hash
6. Record the session_key and begin using it

---

### 5.3 HANDSHAKE_DONE (Client → Node)

The client confirms successful key exchange.

#### HANDSHAKE_DONE Payload (encrypted with session_key)

```
┌───────────────────────────────────────────────────────────────┐
│      Confirmation Hash (32 bytes)                             │
│      BLAKE3("bgpx-done-confirm" || session_key || session_id) │
├───────────────────────────────────────────────────────────────┤
│      Padding (variable)                                       │
└───────────────────────────────────────────────────────────────┘
```

HANDSHAKE_DONE MUST be padded to exactly 128 bytes.

#### Node validation of HANDSHAKE_DONE

The node MUST:

1. Decrypt using the derived session_key
2. Verify the confirmation hash
3. If valid, transition session to ESTABLISHED state
4. Destroy the node ephemeral private key

---

## 6. Full Handshake Sequence

```
Client                                               Node
  │                                                    │
  │──── HANDSHAKE_INIT (encrypted with init_key) ─────►│
  │     [client_ephemeral_pub, session_id, timestamp]   │
  │                                                    │
  │◄─── HANDSHAKE_RESP (encrypted with resp_key) ──────│
  │     [node_ephemeral_pub, session_id_echo,           │
  │      confirmation_hash]                             │
  │                                                    │
  │  Client derives session_key                         │
  │                                                    │
  │──── HANDSHAKE_DONE (encrypted with session_key) ───►│
  │     [confirmation_hash]                             │
  │                                                    │
  │  Session ESTABLISHED                                │
  │◄────────── RELAY / DATA messages ─────────────────►│
  │                                                    │
```

---

## 7. Path-Wide Handshake

A BGP-X path of N hops requires N independent handshakes — one with each node.

**Handshake order**: Entry first, then each relay in order, then the exit node.

**Privacy consideration**: The client performs each handshake by sending HANDSHAKE_INIT directly to each node via its IP address. This is done before the path is operational, so the client's real IP is visible to each node during handshake. This is unavoidable — the entry node's session must be established before it can relay handshake traffic to deeper nodes.

**Alternative for deeper nodes (RECOMMENDED)**: Once the session with the entry node is established, handshakes with relay nodes and the exit node SHOULD be relayed through already-established hops, preventing deeper nodes from learning the client's real IP.

```
Handshake with entry node: direct UDP
Handshake with relay_1: via established entry session
Handshake with relay_2: via established entry → relay_1 sessions
Handshake with exit: via established entry → relay_1 → relay_2 sessions
```

This ensures only the entry node ever learns the client's real IP address.

---

## 8. Session Teardown

Sessions are torn down when:

- The client sends a STREAM_CLOSE on all open streams and no KEEPALIVE is sent
- A KEEPALIVE timeout occurs (90 seconds without traffic)
- An ERROR message is received
- The node daemon is shutting down (DRAIN state)

On teardown, nodes MUST:

- Destroy the session key from memory (use zeroize)
- Remove the session from the session table
- Drop any queued packets for this session

---

## 9. Security Properties

### Mutual authentication

- The node is authenticated: the client verifies the node's static public key against the DHT advertisement before initiating the handshake. A MITM who does not possess the node's static private key cannot complete the handshake.
- The client is NOT authenticated: nodes do not learn client identity. This is by design.

### Forward secrecy

- Session keys are derived from ephemeral keys (dh2) that are destroyed after handshake completion
- Compromise of a node's static private key (node_static_priv) reveals dh1 but not dh2
- Without dh2, the session_key cannot be derived
- Past sessions are safe even under static key compromise

### Replay prevention

- HANDSHAKE_INIT includes a timestamp (60-second window)
- Session IDs are randomly generated and checked for uniqueness
- Sequence numbers in the common header prevent replay of established-session messages

### Key freshness

- Client ephemeral keys: fresh per session
- Node ephemeral keys: fresh per handshake (per session per node)
- Session keys: fresh per session

---

## 10. Implementation Requirements

Implementations MUST:

- Generate X25519 keypairs using a cryptographically secure random number generator
- Destroy ephemeral private keys immediately after use (use zeroize in memory)
- Validate the timestamp in HANDSHAKE_INIT within 60-second tolerance
- Pad HANDSHAKE_INIT to exactly 256 bytes
- Pad HANDSHAKE_RESP to exactly 256 bytes
- Pad HANDSHAKE_DONE to exactly 128 bytes
- Relay handshakes for hops beyond entry through established sessions

Implementations MUST NOT:

- Reuse ephemeral keys across sessions
- Log client ephemeral public keys
- Accept handshakes with timestamps outside the tolerance window
- Proceed to ESTABLISHED state without verifying confirmation hashes
