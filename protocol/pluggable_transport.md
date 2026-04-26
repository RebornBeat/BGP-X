# BGP-X Pluggable Transport Specification

**Version**: 0.1.0-draft

---

## 1. Overview and Purpose

Pluggable Transport (PT) provides a shim between the BGP-X overlay protocol and the physical transport. Its purpose is **traffic analysis resistance** — making BGP-X traffic patterns difficult to identify and block at the ISP level.

PT is NOT a cryptographic security mechanism. BGP-X's ChaCha20-Poly1305 provides content security independently of PT. PT provides additional obfuscation that makes traffic patterns harder to fingerprint.

PT stack position:
```
BGP-X Overlay (onion-encrypted packets)
         │
         ▼
Pluggable Transport shim
  (obfuscates packet patterns, sizes, timing)
         │
         ▼
Physical Transport (UDP/IP for internet, radio for mesh)
```

---

## 2. PT Interface

PT uses a subprocess model. The BGP-X daemon communicates with a PT binary via stdin/stdout.

### Control Channel (JSON, newline-delimited)

```json
→ {"cmd": "start", "mode": "client", "listen_addr": "127.0.0.1:7475"}
← {"status": "ok", "local_addr": "127.0.0.1:7475"}

→ {"cmd": "connect", "remote_addr": "203.0.113.1:7474"}
← {"status": "ok", "conn_id": "a1b2c3d4"}

→ {"cmd": "close", "conn_id": "a1b2c3d4"}
← {"status": "ok"}

→ {"cmd": "stop"}
← {"status": "ok"}
```

### Data Channel (binary, length-prefixed frames)

```
Frame header: [conn_id: 4 bytes] [length: 4 bytes BE]
Frame data:   [length bytes of payload]
```

### PT Modes

| Mode | Description |
|---|---|
| `client` | Outbound obfuscation (client connecting to relay) |
| `server` | Inbound deobfuscation (relay accepting connections) |
| `both` | Bidirectional (relay-to-relay) |

---

## 3. Built-In PT: obfs4-Style

BGP-X includes a built-in PT implementing obfs4-style obfuscation.

### Algorithm

**Key exchange at PT session start**:
1. Both parties generate X25519 ephemeral keypairs
2. X25519 key exchange using Elligator2-encoded public keys (indistinguishable from random bytes)
3. Derive stream cipher key: `HKDF-SHA256(shared_secret, "bgpx-pt-key-v1", 32 bytes)`

**Per-packet obfuscation**:
1. Generate random padding: `random(0, MAX_PADDING)` bytes
2. Construct: `[original BGP-X packet][random padding][padding_length: 2 bytes]`
3. Encrypt with ChaCha20 stream cipher using derived PT key and packet counter
4. Transmit: obfuscated bytes with no recognizable structure

**Key properties**:
- No identifiable TLS fingerprint
- No identifiable X25519 public key structure (Elligator2 encoding)
- Random-padded packet sizes (no fixed patterns)
- No recognizable protocol magic bytes
- Optional timing jitter: 0-50ms per packet (configurable)

---

## 4. External PT Subprocess

```toml
[pluggable_transport]
enabled = true
mode = "external"
binary_path = "/usr/lib/bgpx/pt/obfs4proxy"
args = ["--enableLogging=false"]
listen_addr = "127.0.0.1:7475"
timing_jitter_max_ms = 50
max_padding_bytes = 200
apply_to_mesh = false
```

---

## 5. PT for Mesh Transport

PT can be applied to mesh radio transports to obfuscate BGP-X patterns over radio.

For LoRa: PT overhead must fit within LoRa MTU. Use minimal PT (random padding only, no timing jitter — LoRa duty cycle already introduces natural timing variation).

---

## 6. PT Security Model

PT provides:
- Traffic analysis resistance
- Protocol fingerprint resistance
- Port-based blocking resistance

PT does NOT provide:
- Cryptographic security (BGP-X ChaCha20-Poly1305 provides this independently)
- Guaranteed undetectability against sophisticated ML-based DPI
- Timing attack resistance (partial mitigation only)

If PT is bypassed or detected: BGP-X's underlying security is unchanged. The adversary still cannot decrypt BGP-X traffic. They may identify that BGP-X traffic is present.

---

## 7. PT Overhead

| Type | Amount |
|---|---|
| CPU overhead | ~15% additional |
| Bandwidth overhead | ~5-20% (random padding) |
| Latency overhead (jitter) | 0-50ms (configurable) |
| Throughput reduction | ~10-15% |

---

## 8. Configuration Reference

```toml
[pluggable_transport]
enabled = false
mode = "builtin"             # "builtin" or "external"
binary_path = ""             # For mode = "external"
listen_addr = "127.0.0.1"
listen_port = 7475
timing_jitter_max_ms = 50
max_padding_bytes = 200
apply_to_mesh = false
```
