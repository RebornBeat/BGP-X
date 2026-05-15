# BGP-X Pluggable Transport Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X Pluggable Transport (PT) framework — the mechanism for obfuscating BGP-X traffic patterns to resist ISP-level blocking and deep packet inspection.

---

## 1. Overview and Purpose

Pluggable Transport provides a shim between the BGP-X overlay protocol and the physical transport (UDP socket, mesh radio). Its purpose is **traffic analysis resistance** — making BGP-X traffic patterns difficult to identify and block.

PT is NOT a cryptographic security mechanism. BGP-X's ChaCha20-Poly1305 encryption provides content security regardless of whether PT is active. PT provides an additional layer that makes the traffic patterns harder to fingerprint.

PT sits at this position in the stack:

```
BGP-X Overlay (onion-encrypted packets)
         │
         ▼
Pluggable Transport shim
  (obfuscates packet patterns, sizes, timing)
         │
         ▼
Physical Transport
  (UDP/IP for internet, radio for mesh)
```

---

## 2. PT Interface

The PT framework uses a subprocess model. The BGP-X daemon communicates with a PT binary via stdin/stdout:

```
bgpx-node ←→ [stdin/stdout JSON commands + binary pipe] ←→ PT subprocess
```

### 2.1 PT Binary Protocol

**Control channel** (JSON newline-delimited, stdin/stdout):

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

**Error responses**:

```json
← {"status": "error", "error": "connection_failed", "message": "remote unreachable"}
← {"status": "error", "error": "invalid_command", "message": "unknown cmd"}
```

**Data channel** (binary, multiplexed on same connection using length-prefixed frames):

```
Frame header: [conn_id: 4 bytes] [length: 4 bytes big-endian]
Frame data:   [length bytes of payload]

Example:
  conn_id = 0xa1b2c3d4
  length  = 0x00000100 (256 bytes)
  payload = [256 bytes of obfuscated BGP-X packet]

Full frame: a1b2c3d4 00000100 [256 bytes]
```

Multiple data frames may be sent in succession without waiting for response. The PT processes frames as they arrive.

### 2.2 PT Modes

| Mode | Description |
|---|---|
| `client` | Outbound obfuscation (client connecting to relay) |
| `server` | Inbound deobfuscation (relay accepting connections) |
| `both` | Bidirectional (relay-to-relay connections) |

---

## 3. Built-In PT: obfs4-Style

BGP-X includes a built-in PT implementation that provides obfs4-style obfuscation.

### 3.1 obfs4-Style Algorithm

**Key exchange** (at PT session start):

1. Both parties generate X25519 ephemeral keypairs
2. Perform X25519 key exchange using Elligator2-encoded public keys
   - Elligator2 makes Curve25519 public keys indistinguishable from random bytes
   - Prevents identifying PT sessions by detecting X25519 public key structure
3. Derive a stream cipher key: `HKDF-SHA256(shared_secret, "bgpx-pt-key-v1", 32 bytes)`

**Packet obfuscation** (per BGP-X packet):

```
1. Generate random padding length: random(0, MAX_PADDING)
2. Construct padded packet: [original BGP-X packet][random padding][padding_length: 2 bytes]
3. Encrypt: ChaCha20 stream cipher with derived PT key and packet counter as nonce
4. Transmit: obfuscated bytes with no recognizable structure
```

**Key properties**:
- No identifiable TLS fingerprint
- No identifiable X25519 public key structure (Elligator2)
- Packet sizes random-padded (no fixed patterns)
- No recognizable protocol magic bytes
- Timing: random jitter of 0-50ms added per packet (configurable)

### 3.2 Built-In PT Limitations

The built-in PT is effective against:
- Port-based blocking (random padding disrupts size-based detection)
- Simple protocol fingerprinting (no recognizable magic bytes or TLS fingerprint)

The built-in PT is NOT guaranteed effective against:
- Deep packet inspection (DPI) with machine learning classification
- Nation-state level traffic analysis with sophisticated pattern recognition
- Statistical correlation attacks over long time periods

**Recommendation**: For maximum obfuscation resistance in hostile network environments, use external PT implementations specifically designed for censorship circumvention (e.g., obfs4proxy with properly configured bridges).

The built-in PT is a reasonable default for users who want basic traffic obfuscation without additional configuration. It is NOT suitable for users facing sophisticated network adversaries.

---

## 4. External PT Subprocess

BGP-X supports external PT implementations via the subprocess interface. Any PT that implements the BGP-X PT binary protocol can be used.

### 4.1 Configuration

```toml
[pluggable_transport]
enabled = true
mode = "external"
binary_path = "/usr/lib/bgpx/pt/obfs4proxy"
args = ["--enableLogging=false"]
listen_addr = "127.0.0.1:7475"  # PT binds here; BGP-X connects to it
```

### 4.2 Subprocess Lifecycle

External PT subprocess lifecycle:

1. `bgpx-node` starts the PT subprocess (fork/exec)
2. PT subprocess binds to `listen_addr`
3. `bgpx-node` sends `{"cmd": "start", ...}` via stdin
4. PT confirms ready via stdout
5. `bgpx-node` routes all new overlay connections through PT
6. On shutdown: `bgpx-node` sends `{"cmd": "stop"}`
7. PT subprocess exits cleanly
8. If PT subprocess crashes: `bgpx-node` detects via SIGCHLD and either restarts PT or falls back to non-PT transport (configurable)

**Failure handling**:
- If PT subprocess fails to start: log error, fall back to non-PT transport
- If PT subprocess crashes during operation: log error, attempt restart (max 3 attempts with exponential backoff), then fall back
- Fallback behavior is controlled by `fallback_on_failure` config option

---

## 5. PT for Mesh Transport

Pluggable Transport can also be applied to mesh radio transports (LoRa, WiFi mesh) to obfuscate BGP-X traffic patterns over radio.

For mesh transport PT:
- The PT operates above the transport layer but below the BGP-X overlay layer
- Packet sizes and timing are randomized within the transport's MTU constraints
- For LoRa: PT overhead must fit within LoRa MTU (requires very lightweight PT)

Recommended: for LoRa mesh, use minimal PT (random padding only, no timing jitter since LoRa duty cycle already introduces timing variation).

---

## 6. PT Security Model

PT provides:
- **Traffic analysis resistance**: patterns harder to identify
- **Protocol fingerprint resistance**: not identifiable as BGP-X
- **Port-based blocking resistance**: random padding disrupts size-based detection

PT does NOT provide:
- **Cryptographic security**: BGP-X ChaCha20-Poly1305 provides this independently
- **Guaranteed undetectability**: sufficiently sophisticated DPI may detect patterns
- **Timing attack resistance**: PT timing jitter provides partial mitigation only

**Important**: If PT is compromised or bypassed, BGP-X's underlying security is unchanged. The adversary still cannot decrypt BGP-X traffic. They may be able to identify that BGP-X traffic is present.

---

## 7. PT Overhead

| Overhead Type | Amount | Notes |
|---|---|---|
| CPU | ~15% | Encryption + padding computation |
| Bandwidth | ~5-20% | Random padding |
| Latency | 0-50ms | Configurable timing jitter |
| Throughput reduction | ~10-15% | CPU + bandwidth overhead |

PT is disabled by default. Enable only when ISP blocking or traffic analysis is a concern.

---

## 8. PT Configuration Reference

```toml
[pluggable_transport]
# Enable PT
enabled = false

# PT mode: "builtin" or "external"
mode = "builtin"

# For mode = "external": path to PT binary
binary_path = "/usr/lib/bgpx/pt/obfs4proxy"

# PT listen address (bgpx-node connects to this)
listen_addr = "127.0.0.1"
listen_port = 7475

# Timing jitter range in milliseconds (0 = no jitter)
timing_jitter_max_ms = 50

# Maximum padding bytes added per packet (0 = no padding)
max_padding_bytes = 200

# Whether to apply PT to mesh transports as well
apply_to_mesh = false

# Fallback behavior if PT subprocess fails
fallback_on_failure = true    # If true, use non-PT transport when PT fails
                            # If false, refuse connections when PT unavailable
```

---

## 9. Test Vectors

Test vectors for PT obfuscation verification. All implementations MUST pass these tests.

### 9.1 Elligator2 Key Exchange (3 test cases)

Verify that Elligator2-encoded X25519 public keys are indistinguishable from random bytes:

| Test | Input | Verification |
|---|---|---|
| `pt_elligator2_1` | X25519 public key `0x...` | Encoded output passes NIST SP 800-22 randomness test (simplified: no fixed patterns in first 8 bytes) |
| `pt_elligator2_2` | Different X25519 public key | Encoded output is different from test 1 (no key leakage) |
| `pt_elligator2_3` | All-zeros public key (edge case) | Encoded output still indistinguishable from random |

### 9.2 Packet Obfuscation Round-Trip (5 test cases)

| Test | Input Packet | Verification |
|---|---|---|
| `pt_roundtrip_1` | 64-byte packet | `deobfuscate(obfuscate(packet, key, nonce), key, nonce) == packet` |
| `pt_roundtrip_2` | 128-byte packet | Same verification |
| `pt_roundtrip_3` | 256-byte packet | Same verification |
| `pt_roundtrip_4` | 1024-byte packet | Same verification |
| `pt_roundtrip_5` | 1500-byte packet (MTU edge) | Same verification |

### 9.3 Padding Distribution (Statistical Test)

| Test | Description | Verification |
|---|---|---|
| `pt_padding_dist` | Obfuscate 1000 identical packets with padding enabled | Padding lengths follow uniform distribution with p > 0.05 (chi-squared test) |

### 9.4 No Fixed Magic Bytes

| Test | Description | Verification |
|---|---|---|
| `pt_no_magic` | Obfuscate 100 packets, concatenate first 100 bytes of each | No 4-byte sequence appears with frequency > 5% (no magic bytes) |

### 9.5 Test Vector Location

Full test vector files with hex-encoded inputs/outputs will be published in `/protocol/test_vectors/pt_*.txt` during implementation phase.
