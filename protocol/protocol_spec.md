# BGP-X Protocol Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

---

## 1. Introduction

This document is the authoritative specification for the BGP-X wire protocol. It defines:

- The message types exchanged between BGP-X nodes
- The structure and encoding of all protocol messages
- The state machines governing node behavior
- The rules for session establishment, maintenance, and teardown
- The behavior required of compliant implementations

All BGP-X implementations must conform to this specification. Where this document is ambiguous, implementors should err toward the more privacy-preserving interpretation and open an issue for clarification.

---

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

| Term | Definition |
|---|---|
| Node | Any participant in the BGP-X overlay network |
| Client | An initiating party that constructs and sends overlay packets |
| Entry node | The first relay hop; knows client IP, not destination |
| Relay node | A middle hop; knows only previous and next hop |
| Exit node / Gateway | The final hop for clearnet traffic; knows destination, not client IP |
| Path | An ordered sequence of nodes from entry to exit |
| Stream | A logical bidirectional channel multiplexed over a path |
| Session | The cryptographic context for communication between a client and a path |
| Onion packet | A layered-encrypted packet that is progressively decrypted at each hop |
| NodeID | BLAKE3(Ed25519_public_key) — 32 bytes |
| ServiceID | Ed25519 public key of a BGP-X native service — 32 bytes |
| Advertisement | A signed record published by a node to the DHT |
| DHT | Distributed Hash Table used for node discovery |

---

## 3. Protocol Overview

BGP-X defines two protocol layers:

### 3.1 Underlay (Transport)

BGP-X runs over UDP/IP. All BGP-X packets are UDP datagrams. The protocol does not use TCP between relay nodes.

Default BGP-X port: **7474/UDP**

Implementations MUST support IPv4. Implementations SHOULD support IPv6. Dual-stack operation is RECOMMENDED.

### 3.2 Overlay (BGP-X Protocol)

The BGP-X overlay protocol operates within UDP payloads. It defines:

- Packet types and their wire formats
- Onion encryption structure
- Session establishment and teardown
- Stream multiplexing
- Control plane messages (node advertisement, DHT operations)

---

## 4. Message Types

BGP-X defines the following top-level message types. All messages are identified by a 1-byte type field in the outer header.

| Type ID | Name | Direction | Description |
|---|---|---|---|
| 0x01 | RELAY | Any → Any | Forward an onion-encrypted payload |
| 0x02 | HANDSHAKE_INIT | Client → Node | Initiate session key exchange |
| 0x03 | HANDSHAKE_RESP | Node → Client | Respond to key exchange |
| 0x04 | HANDSHAKE_DONE | Client → Node | Complete key exchange |
| 0x05 | DATA | Within session | Encrypted application data |
| 0x06 | STREAM_OPEN | Within session | Open a new multiplexed stream |
| 0x07 | STREAM_CLOSE | Within session | Close a multiplexed stream |
| 0x08 | STREAM_DATA | Within session | Data on a specific stream |
| 0x09 | KEEPALIVE | Any → Any | Maintain session liveness |
| 0x0A | NODE_ADVERTISE | Node → DHT | Publish node advertisement |
| 0x0B | DHT_FIND_NODE | Any → DHT | Locate nodes near a target ID |
| 0x0C | DHT_FIND_NODE_RESP | DHT → Any | Response to DHT_FIND_NODE |
| 0x0D | DHT_GET | Any → DHT | Retrieve a DHT record |
| 0x0E | DHT_GET_RESP | DHT → Any | Response to DHT_GET |
| 0x0F | DHT_PUT | Any → DHT | Store a DHT record |
| 0x10 | ERROR | Any → Any | Signal a protocol error |
| 0x11 | COVER | Any → Any | Cover traffic (no payload) |

---

## 5. Common Header

Every BGP-X message begins with a common outer header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────┬───────────────┬───────────────────────────────┤
│    Version    │   Msg Type    │          Reserved             │
├───────────────┴───────────────┴───────────────────────────────┤
│                        Session ID                             │
│                        (16 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│                       Sequence Number                         │
│                         (8 bytes)                             │
├───────────────────────────────────────────────────────────────┤
│                     Payload Length                            │
│                        (4 bytes)                              │
├───────────────────────────────────────────────────────────────┤
│                         Payload                               │
│                      (variable)                               │
└───────────────────────────────────────────────────────────────┘
```

| Field | Size | Description |
|---|---|---|
| Version | 1 byte | Protocol version. Current: 0x01 |
| Msg Type | 1 byte | Message type identifier (see Section 4) |
| Reserved | 2 bytes | MUST be 0x0000. Reserved for future use. |
| Session ID | 16 bytes | Randomly generated per session. 0x00...00 for pre-session messages |
| Sequence Number | 8 bytes | Monotonically increasing per session. Used for replay protection. |
| Payload Length | 4 bytes | Length of the payload in bytes, big-endian |
| Payload | Variable | Message-type-specific content |

All multi-byte integer fields are big-endian (network byte order).

---

## 6. RELAY Message

The RELAY message is the primary data-forwarding message. It carries an onion-encrypted payload.

```
Common Header (Msg Type = 0x01)
├───────────────────────────────────────────────────────────────┤
│                     Onion Payload                             │
│          (ChaCha20-Poly1305 encrypted, this hop only)        │
└───────────────────────────────────────────────────────────────┘
```

### Relay node behavior

Upon receiving a RELAY message, a node MUST:

1. Verify the outer header (Version, Reserved fields)
2. Look up the session key associated with the Session ID
3. If no session exists for this Session ID, drop the packet silently and log the event
4. Attempt to decrypt the onion payload using the session key and the Sequence Number as the nonce
5. If decryption fails (authentication tag mismatch), drop the packet silently and log the event
6. Extract the next-hop NodeID and the remaining encrypted payload from the decrypted content
7. Forward the remaining encrypted payload (with a new outer header for the next hop) to the next-hop node

Relay nodes MUST NOT:

- Log the source address of RELAY messages
- Log the content of RELAY messages
- Attempt to decrypt more than their own layer
- Modify the payload before forwarding

---

## 7. HANDSHAKE Messages

See `/protocol/handshake.md` for the full handshake specification. Summary:

HANDSHAKE_INIT, HANDSHAKE_RESP, and HANDSHAKE_DONE form a 3-message handshake that establishes a shared session key between the client and each node in the path.

The handshake uses X25519 ECDH with HKDF-SHA256 for key derivation. Handshake messages are sent to each node in the path independently and in sequence (entry first, then each relay, then exit).

---

## 8. DATA Message

The DATA message carries encrypted application data within an established session.

```
Common Header (Msg Type = 0x05)
├───────────────────────────────────────────────────────────────┤
│                      Stream ID (4 bytes)                      │
├───────────────────────────────────────────────────────────────┤
│                    Flags (2 bytes)                            │
├───────────────────────────────────────────────────────────────┤
│               Encrypted Application Data                      │
│                      (variable)                               │
└───────────────────────────────────────────────────────────────┘
```

| Field | Size | Description |
|---|---|---|
| Stream ID | 4 bytes | Identifies the stream within this session |
| Flags | 2 bytes | See below |
| Encrypted Data | Variable | Application payload, encrypted with session key |

### Flags

| Bit | Name | Description |
|---|---|---|
| 0 | FIN | Last data segment on this stream |
| 1 | RST | Reset stream immediately |
| 2 | SYN | First data segment on a new stream (stream open) |
| 3–15 | Reserved | MUST be 0 |

---

## 9. STREAM Messages

### STREAM_OPEN (0x06)

Opens a new multiplexed stream within an existing session.

```
Common Header (Msg Type = 0x06)
├───────────────────────────────────────────────────────────────┤
│                     Stream ID (4 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│                  Destination (variable)                       │
│         (encrypted; contains target address and port)        │
└───────────────────────────────────────────────────────────────┘
```

The Destination field is encrypted under the session key. It contains:

- Address type (1 byte): 0x04 = IPv4, 0x06 = IPv6, 0x0A = domain name, 0xBB = BGP-X ServiceID
- Address (4, 16, or variable bytes depending on type)
- Port (2 bytes, big-endian)

### STREAM_CLOSE (0x07)

Closes a stream. Both parties MUST cease sending data on the stream after sending or receiving STREAM_CLOSE.

```
Common Header (Msg Type = 0x07)
├───────────────────────────────────────────────────────────────┤
│                     Stream ID (4 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│                  Close Reason (2 bytes)                       │
└───────────────────────────────────────────────────────────────┘
```

### Close Reasons

| Code | Meaning |
|---|---|
| 0x0000 | Normal close |
| 0x0001 | Remote error |
| 0x0002 | Destination unreachable |
| 0x0003 | Exit policy denied |
| 0x0004 | Timeout |
| 0x0005 | Protocol error |

---

## 10. KEEPALIVE Message

Maintains session liveness and prevents NAT timeout on the underlying UDP path.

```
Common Header (Msg Type = 0x09)
(No payload)
```

Nodes MUST send KEEPALIVE messages every 25 seconds on idle sessions.

Nodes MUST treat a session as dead if no KEEPALIVE or DATA message is received for 90 seconds and MUST tear down the session.

KEEPALIVE messages SHOULD be sent at irregular intervals (randomized within ±5 seconds of the 25-second target) to resist timing fingerprinting.

---

## 11. DHT Messages

### DHT_FIND_NODE (0x0B)

Query the DHT for nodes closest to a target NodeID.

```
Common Header (Msg Type = 0x0B)
├───────────────────────────────────────────────────────────────┤
│                   Target NodeID (32 bytes)                    │
├───────────────────────────────────────────────────────────────┤
│               Max Results (1 byte, 1–20)                      │
└───────────────────────────────────────────────────────────────┘
```

### DHT_FIND_NODE_RESP (0x0C)

```
Common Header (Msg Type = 0x0C)
├───────────────────────────────────────────────────────────────┤
│              Result Count (1 byte)                            │
├───────────────────────────────────────────────────────────────┤
│           NodeContact entries (Result Count × 40 bytes)       │
└───────────────────────────────────────────────────────────────┘
```

NodeContact entry (40 bytes):

| Field | Size | Description |
|---|---|---|
| NodeID | 32 bytes | BLAKE3 hash of node's public key |
| IP Address | 4 bytes | IPv4 address of the node |
| Port | 2 bytes | UDP port |
| Reserved | 2 bytes | Future use |

### DHT_PUT (0x0F)

Store a node advertisement record in the DHT.

```
Common Header (Msg Type = 0x0F)
├───────────────────────────────────────────────────────────────┤
│              Record Length (4 bytes)                          │
├───────────────────────────────────────────────────────────────┤
│           Signed Node Advertisement Record                    │
│                     (variable)                                │
└───────────────────────────────────────────────────────────────┘
```

### DHT_GET (0x0D)

Retrieve a node advertisement record by NodeID.

```
Common Header (Msg Type = 0x0D)
├───────────────────────────────────────────────────────────────┤
│                   Target NodeID (32 bytes)                    │
└───────────────────────────────────────────────────────────────┘
```

### DHT_GET_RESP (0x0E)

```
Common Header (Msg Type = 0x0E)
├───────────────────────────────────────────────────────────────┤
│              Found (1 byte): 0x01 = found, 0x00 = not found  │
├───────────────────────────────────────────────────────────────┤
│              Record Length (4 bytes, if Found = 0x01)         │
├───────────────────────────────────────────────────────────────┤
│           Signed Node Advertisement Record                    │
│               (variable, present if Found = 0x01)            │
└───────────────────────────────────────────────────────────────┘
```

---

## 12. ERROR Message

Signals a protocol error. Nodes MAY send ERROR messages; they MUST NOT rely on receiving them.

```
Common Header (Msg Type = 0x10)
├───────────────────────────────────────────────────────────────┤
│                Error Code (2 bytes)                           │
├───────────────────────────────────────────────────────────────┤
│           Error Message (UTF-8, max 256 bytes)                │
└───────────────────────────────────────────────────────────────┘
```

See `/protocol/error_handling.md` for the full error code table.

---

## 13. COVER Message

Cover traffic. The COVER message carries no meaningful payload and is used to disrupt traffic analysis.

```
Common Header (Msg Type = 0x11)
├───────────────────────────────────────────────────────────────┤
│              Padding (random bytes, variable length)          │
└───────────────────────────────────────────────────────────────┘
```

COVER messages MUST be indistinguishable in size from real traffic. The padding length SHOULD be drawn from the same distribution as real payloads on the same session.

Nodes MUST NOT log the receipt of COVER messages.

Relay nodes MUST forward COVER messages identically to RELAY messages — they must not be dropped or flagged.

---

## 14. Node Advertisement Record

Node advertisements are the primary unit of node discovery in BGP-X. They are stored in the DHT and retrieved by clients during path construction.

### Structure

```json
{
  "version": 1,
  "node_id": "<32-byte BLAKE3 hash, hex>",
  "public_key": "<32-byte Ed25519 public key, base64url>",
  "roles": ["relay"],
  "endpoints": [
    {
      "protocol": "udp",
      "address": "203.0.113.1",
      "port": 7474
    }
  ],
  "exit_policy": null,
  "bandwidth_mbps": 100,
  "latency_ms": 12,
  "uptime_pct": 99.1,
  "region": "NA",
  "country": "US",
  "asn": 12345,
  "operator_id": "<32-byte Ed25519 public key of operator, base64url>",
  "signed_at": "2026-04-24T00:00:00Z",
  "expires_at": "2026-04-25T00:00:00Z",
  "signature": "<64-byte Ed25519 signature over canonical JSON, base64url>"
}
```

### Field Definitions

| Field | Required | Description |
|---|---|---|
| version | REQUIRED | Advertisement format version. Currently 1. |
| node_id | REQUIRED | BLAKE3(public_key). 32 bytes, hex-encoded. |
| public_key | REQUIRED | Ed25519 public key. 32 bytes, base64url. |
| roles | REQUIRED | Array of: "relay", "entry", "exit", "discovery". At least one required. |
| endpoints | REQUIRED | Array of network endpoints where the node accepts connections. |
| exit_policy | CONDITIONAL | Required if roles includes "exit". See exit policy specification. |
| bandwidth_mbps | RECOMMENDED | Self-reported available bandwidth in Mbps. |
| latency_ms | RECOMMENDED | Self-reported round-trip latency to nearest major internet exchange, in ms. |
| uptime_pct | RECOMMENDED | Self-reported uptime percentage over the past 30 days. |
| region | REQUIRED | Two-letter region code: NA, EU, AP, SA, AF, ME. |
| country | REQUIRED | ISO 3166-1 alpha-2 country code. |
| asn | REQUIRED | Autonomous System Number of the node's network. |
| operator_id | RECOMMENDED | Ed25519 public key of the operator. Allows reputation tracking across nodes. |
| signed_at | REQUIRED | ISO 8601 timestamp of when the advertisement was signed. |
| expires_at | REQUIRED | ISO 8601 timestamp. MUST be no more than 48 hours after signed_at. |
| signature | REQUIRED | Ed25519 signature over the canonical JSON of all other fields. |

### Canonical JSON for Signing

Before signing, the advertisement MUST be serialized as canonical JSON:

- All keys sorted alphabetically
- No whitespace (compact encoding)
- The "signature" field MUST be excluded from the signed content
- UTF-8 encoding

### Signature Verification

Receivers MUST:

1. Recompute node_id = BLAKE3(public_key) and verify it matches the node_id field
2. Verify the signature over the canonical JSON using the public_key field
3. Verify that signed_at is not in the future (allowing 60 seconds of clock skew)
4. Verify that expires_at has not passed
5. Reject the advertisement if any verification step fails

---

## 15. Exit Policy

Nodes with the "exit" role MUST include a signed exit policy in their advertisement.

```json
{
  "version": 1,
  "allow_protocols": ["tcp", "udp"],
  "allow_ports": [80, 443, 8080, 8443],
  "deny_destinations": ["10.0.0.0/8", "192.168.0.0/16", "172.16.0.0/12"],
  "logging_policy": "none",
  "operator_contact": "operator@example.com",
  "jurisdiction": "US",
  "policy_signed_at": "2026-04-24T00:00:00Z",
  "policy_signature": "<Ed25519 signature>"
}
```

| Field | Description |
|---|---|
| allow_protocols | Protocols the gateway will forward: "tcp", "udp", or both |
| allow_ports | List of destination ports the gateway will forward. Use [0] for all ports. |
| deny_destinations | CIDR blocks the gateway will never forward to. Private IP ranges MUST be included. |
| logging_policy | "none" (no logs kept), "metadata" (timing/destination logged), "full" (all traffic logged) |
| operator_contact | Public contact for the gateway operator |
| jurisdiction | ISO 3166-1 country code of the legal jurisdiction the operator is subject to |
| policy_signed_at | Timestamp of policy signing |
| policy_signature | Ed25519 signature over canonical JSON of policy fields |

Gateways MUST adhere to their published exit policy. Gateways that are observed violating their exit policy SHOULD be reported and will be flagged in the reputation system.

---

## 16. Replay Protection

All BGP-X messages carry a Sequence Number in the common header. Recipients MUST maintain a sliding window of received sequence numbers per session.

- Window size: 64 sequence numbers
- A message whose sequence number falls outside the window or has already been received MUST be dropped silently
- The window advances as new valid messages are received

---

## 17. MTU and Fragmentation

BGP-X does not perform IP fragmentation. The maximum BGP-X payload size is chosen to fit within a standard MTU:

- Target path MTU: 1280 bytes (minimum IPv6 MTU; safe for all paths)
- BGP-X overhead (header + encryption): approximately 120 bytes
- Maximum application payload per packet: 1160 bytes

For application data that exceeds this limit, the stream layer MUST fragment the data across multiple packets and reassemble at the receiver.

---

## 18. Protocol State Machine

### 18.1 Node state machine

```
         ┌──────────┐
         │  INITIAL │
         └────┬─────┘
              │ Start node daemon
              ▼
         ┌──────────┐
         │BOOTSTRAP │  ← DHT join, publish advertisement
         └────┬─────┘
              │ Bootstrap complete
              ▼
         ┌──────────┐
         │  ACTIVE  │  ← Accept connections, relay traffic
         └────┬─────┘
              │ Shutdown signal
              ▼
         ┌──────────┐
         │  DRAIN   │  ← Complete in-flight sessions
         └────┬─────┘
              │ All sessions closed
              ▼
         ┌──────────┐
         │   STOP   │
         └──────────┘
```

### 18.2 Session state machine (per node, per session)

```
         ┌──────────────┐
         │   NO SESSION │
         └──────┬───────┘
                │ Receive HANDSHAKE_INIT
                ▼
         ┌──────────────┐
         │ HANDSHAKING  │
         └──────┬───────┘
                │ HANDSHAKE_DONE received and verified
                ▼
         ┌──────────────┐
         │  ESTABLISHED │  ← Relay RELAY/DATA/KEEPALIVE messages
         └──────┬───────┘
                │ Timeout or ERROR
                ▼
         ┌──────────────┐
         │    CLOSED    │
         └──────────────┘
```

---

## 19. Compliance Requirements

A BGP-X-compliant implementation MUST:

- Implement all message types in Section 4
- Implement the common header format in Section 5
- Implement the onion encryption scheme described in `/protocol/packet_format.md`
- Implement the handshake protocol described in `/protocol/handshake.md`
- Implement replay protection as described in Section 16
- Respect the MTU limits in Section 17
- Implement the state machines in Section 18
- Verify all node advertisement signatures before using a node
- Implement geographic and ASN diversity enforcement in path selection
- Never log client source IPs at relay or exit nodes

A BGP-X-compliant implementation SHOULD:

- Support IPv6
- Support cover traffic generation
- Support pluggable transport
- Implement the reputation system described in `/control-plane/reputation_system.md`

A BGP-X-compliant implementation MUST NOT:

- Allow disabling of signature verification
- Allow disabling of replay protection
- Log relay packet source addresses
- Attempt to decrypt onion layers beyond its own
