# BGP-X Attack Vectors

**Version**: 0.1.0-draft

---

## 1. Protocol-Level Attacks

### 1.1 Replay Attack

**Method**: record valid BGP-X packet; retransmit at later time.

**Mitigation**: per-session 64-position sliding window inbound sequence number tracking. Replayed sequence numbers detected and silently dropped. Independent inbound/outbound sequence spaces prevent cross-directional replay.

**Status**: Mitigated.

### 1.2 Packet Injection

**Method**: inject forged BGP-X packets into an established session.

**Mitigation**: ChaCha20-Poly1305 authentication tag covers entire payload. Injected packets fail authentication (Poly1305 tag verification) and are silently dropped. MUST verify tag before any plaintext access.

**Status**: Mitigated.

### 1.3 Session Guessing Attack

**Method**: guess valid Session ID (16 bytes = 128 bits) to inject packets.

**Mitigation**: 2^-128 collision probability. Computationally infeasible. Even with correct session_id, authentication tag requires correct session_key.

**Status**: Mitigated.

### 1.4 Handshake MITM

**Method**: intercept HANDSHAKE_INIT and substitute attacker's ephemeral key; intercept HANDSHAKE_RESP and substitute forged response.

**Mitigation**: client verifies HANDSHAKE_RESP against the node's Ed25519 public key published in DHT advertisement. Confirmation hash in HANDSHAKE_RESP and HANDSHAKE_DONE binds both parties to the same session_key derived from the actual node's static key. MITM without node_static_priv cannot produce valid confirmation hash.

**Status**: Mitigated (requires DHT to accurately reflect node identity).

### 1.5 Version Downgrade Attack

**Method**: attacker intercepts version negotiation and forces older vulnerable version.

**Mitigation**: version list encrypted in HANDSHAKE_INIT payload (authenticated and encrypted). Attacker cannot modify without breaking authentication tag. Both parties verify the negotiated version against their minimum supported version.

**Status**: Mitigated.

### 1.6 Path Manipulation

**Method**: relay node alters next-hop field to redirect path.

**Mitigation**: onion layers are encrypted for specific nodes only (using their session key). A relay node receiving its layer cannot alter inner layers without breaking their authentication tags. Client detects end-to-end path failure via KEEPALIVE timeout.

**Status**: Mitigated.

### 1.7 path_id Guessing

**Method**: adversary guesses valid path_id to inject return traffic into active session.

**Mitigation**: path_id is 8 bytes = 64 bits. 2^-64 guessing probability per attempt. Not all-zeros (protocol-level rejection). Even with correct path_id, injected traffic must pass authentication tag verification at the recipient node.

**Status**: Sufficiently mitigated.

---

## 2. Cross-Domain Specific Attack Vectors

### 2.1 Domain Bridge MITM

**Method**: adversary positions themselves between a domain bridge node and the next relay in the target domain; observes traffic in both domains.

**Mitigation**:
- All onion layers ChaCha20-Poly1305 encrypted — bridge does not decrypt inner layers
- DOMAIN_BRIDGE layer encrypted to bridge node's session key specifically
- Authentication tags prevent modification at every layer
- Bridge node's signed advertisement binds identity to NodeID

**Status**: Mitigated — same protection as any relay MITM.

### 2.2 Fake Mesh Island

**Method**: adversary creates a fake mesh island with signed (but controlled) island advertisement and controlled relay nodes; attempts to route victim traffic through their infrastructure.

**Mitigation**:
- MESH_ISLAND_ADVERTISE Ed25519 signed by island curator or bridge nodes
- Individual node verification required for every relay — each "mesh relay" needs valid self-signed advertisement
- Reputation system degrades misbehaving nodes over time
- Pool curator verification for island pools
- Weighted random selection means controlled nodes compete probabilistically

**Status**: Partially mitigated. Adversary must operate real nodes with valid self-signed advertisements. Standard adversary model applies once inside controlled island.

### 2.3 Bridge Node Sybil Attack

**Method**: adversary operates many real bridge nodes for a specific domain pair, attempting to dominate all cross-domain traffic for that pair.

**Mitigation**:
- Operator diversity: no two bridge nodes with same operator_id in one path
- Weighted random selection: adversary bridge nodes compete with legitimate ones proportionally
- Reputation scoring: bridge nodes with low uptime or failures score lower
- Multiple independent bridge pools discourage single-operator dominance

**Status**: Partially mitigated. Requires significant infrastructure investment by adversary. Operator diversity is the primary defense.

### 2.4 Domain Isolation Attack (Availability)

**Method**: adversary takes offline all bridge nodes connecting a mesh island to other domains via DDoS or BGP hijack.

**Mitigation**:
- Multiple independent bridge nodes (different operators, jurisdictions, ASNs) reduces single-point-of-failure
- LoRa backup links between islands (not dependent on clearnet)
- Store-and-forward for critical messages during outage

**Status**: Availability risk, not a privacy failure. BGP-X cannot protect against physical infrastructure disruption.

### 2.5 Cross-Domain Timing Correlation

**Method**: adversary observes traffic entering clearnet at entry node AND mesh radio traffic at island; attempts to correlate to identify source-destination pairs.

**Mitigation**:
- Cover traffic in all domains (COVER uses session_key; externally identical to RELAY)
- KEEPALIVE jitter ±5 seconds (MANDATORY in all domains)
- Domain bridge processing latency adds noise (variable radio propagation)
- Different fundamental latency profiles between clearnet and mesh
- Multiple bridge nodes: adversary cannot know which bridge was used

**Status**: Residual risk for global adversaries with multi-domain observation capability. Acknowledged in threat model.

### 2.6 Rogue DOMAIN_ADVERTISE Record

**Method**: adversary publishes fake DOMAIN_ADVERTISE records claiming a bridge between two domains; attempts to be selected as a bridge node.

**Mitigation**:
- DOMAIN_ADVERTISE records Ed25519 signed by the claiming node's private key
- Individual node advertisement verification required before bridge node selection
- Node's own advertisement must confirm it serves both bridged domains
- If bridge doesn't work: KEEPALIVE timeout within 90 seconds; client rebuilds path with different bridge
- Persistent fake advertisements → reputation penalty BRIDGE_CLAIMED_AVAILABLE_OFFLINE

**Status**: Mitigated. Adversary must operate a real node that can actually serve both domains.

### 2.7 Mesh Island DHT Poisoning

**Method**: adversary floods unified DHT with fake MESH_ISLAND_ADVERTISE or DOMAIN_ADVERTISE records.

**Mitigation**:
- All DHT records Ed25519 signed
- Storage nodes MUST verify signatures before storing (unsigned or invalid records rejected)
- Reputation system penalizes nodes publishing fake records (FAKE_DOMAIN_ADVERTISEMENT: -15.0)

**Status**: Mitigated at the cryptographic layer.

---

## 3. Implementation-Specific Attack Vectors

### 3.1 Buffer Overflow in Packet Parsing

**Method**: malformed BGP-X packet with incorrect length fields triggers buffer read/write beyond packet bounds.

**Mitigation**: all parsing uses Rust safe code (no raw pointer arithmetic); bounds checks enforced by compiler. All length fields validated against actual datagram size before use. Fuzzing required during implementation testing.

**Status**: Mitigated by implementation choice (Rust) and input validation.

### 3.2 Timing Side-Channel in Crypto

**Method**: measure time taken to process authenticated packets vs. failed authentication to extract session key bits.

**Mitigation**: ChaCha20-Poly1305 from audited library with constant-time implementation. Poly1305 authentication tag verification MUST be constant-time. Accept/reject decisions must not branch on secret data.

**Status**: Dependent on correct library usage.

### 3.3 Key Material in Swap

**Method**: read session keys from system swap file after process exits.

**Mitigation**: `mlock()` on all key material to prevent swapping to disk. Explicit zeroization using `zeroize` crate before deallocation.

**Status**: Mitigated.

### 3.4 PT Bypass

**Method**: DPI identifies BGP-X traffic despite PT obfuscation; ISP blocks port 7474.

**Mitigation**: PT makes traffic appear random. If bypassed: BGP-X security (crypto) unchanged. Alternative ports configurable. Multiple PT modes available.

**Status**: Partial — no PT is perfect. BGP-X security unaffected by PT bypass.

### 3.5 ECH Oracle at Exit Node

**Method**: timing or error-based oracle attack on ECH implementation at exit node reveals domain name.

**Mitigation**: ECH from audited TLS library. No timing branches on decryption success/failure. Error responses standardized (no domain-specific error detail).

**Status**: Dependent on TLS library implementation.

### 3.6 Routing Policy Bypass

**Method**: attacker manipulates routing tables on host OS to bypass BGP-X TUN interface; traffic exits directly without going through overlay.

**Mitigation**: kill switch blocks non-BGP-X traffic when BGP-X is active. TUN interface takes priority via routing metric. BGP-X daemon monitors routing table for unexpected changes. Default-deny policy for unmatched traffic.

**Status**: Mitigated with kill switch enabled.

### 3.7 Rogue Mesh Beacon

**Method**: attacker broadcasts MESH_BEACON with fake NodeID to poison DHT routing table.

**Mitigation**: MESH_BEACON is Ed25519 signed. Signature verification mandatory before DHT update. NodeID = BLAKE3(public_key) — forge attempt fails without private key.

**Status**: Mitigated.

### 3.8 Radio Interception (Mesh Domain)

**Method**: attacker intercepts mesh radio frames.

**Mitigation**: all BGP-X packets ChaCha20-Poly1305 encrypted with session keys. Radio interceptor sees ciphertext only. Traffic analysis mitigated by cover traffic and KEEPALIVE jitter.

**Status**: Content protected. Timing analysis partially mitigated.

### 3.9 Cross-Domain Path Table Overflow

**Method**: adversary floods a bridge node with new sessions to exhaust cross-domain path table memory.

**Mitigation**:
- Path table entries expire at session idle TTL (90 seconds default)
- Background cleanup every 30 seconds
- Maximum session count configurable (default 10,000)
- Rate limiting on new handshake acceptance

**Status**: Mitigated.

### 3.10 Domain Bridge Timing Oracle

**Method**: adversary measures timing differences between bridge nodes to infer which bridge was selected for specific destination domains.

**Mitigation**:
- Bridge selection uses weighted random (not deterministic)
- Radio transmission time for mesh varies (physical randomness)
- Cover traffic in both domains
- KEEPALIVE jitter

**Status**: Partially mitigated.
