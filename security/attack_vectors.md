# BGP-X Attack Vectors

**Version**: 0.1.0-draft

This document enumerates specific attack vectors against BGP-X, provides analysis of each attack's feasibility, and specifies the mitigations built into the protocol and implementation.

---

## 1. Protocol-Level Attacks

### 1.1 Replay Attack

**Description**: An attacker captures a valid BGP-X packet and retransmits it later to replay the encrypted payload.

**Impact**: Potential duplicate delivery of application data; potential session state confusion.

**Feasibility**: Easy to attempt; low success probability.

**Mitigation**:
- 64-sequence-number sliding window per session per direction
- Two independent windows (inbound/outbound) prevent cross-direction replay
- Session keys are ephemeral — replaying a packet from a previous session fails because the session key no longer exists
- path_id is random per path; replay has no useful effect

**Status**: Mitigated.

---

### 1.2 Packet Injection

**Description**: An attacker on the network path forges a BGP-X packet and injects it into an active session.

**Impact**: Potential delivery of malicious data to application; potential session disruption.

**Feasibility**: Moderate to attempt; fails cryptographically.

**Mitigation**:
- ChaCha20-Poly1305 authentication tag covers all packet content and the header
- A forged packet without a valid session key produces a tag mismatch
- Forged packets are dropped silently before any plaintext is accessed
- Attacker cannot forge a valid tag without the session key (which they do not have)
- Authentication tag verification MUST be constant-time

**Status**: Mitigated (requires breaking ChaCha20-Poly1305 authentication).

---

### 1.3 Session ID Guessing

**Description**: An attacker tries to guess a valid 16-byte session ID to inject packets into an existing session.

**Impact**: If successful, attacker could inject packets into the session.

**Feasibility**: Computationally infeasible.

**Analysis**:
- Session IDs are 16 bytes (128 bits) of random data from CSPRNG
- Probability of guessing a valid session ID from 10,000 active sessions: 10,000 / 2^128 ≈ 2.9 × 10^-34
- Even 10^9 guesses per second would require 10^25 years to find a valid ID

**Mitigation**: Random session ID generation from CSPRNG.

**Status**: Mitigated.

---

### 1.4 Handshake MITM

**Description**: An attacker intercepts the HANDSHAKE_INIT message and attempts to impersonate the entry node to the client.

**Impact**: If successful, attacker could establish a session with the client and see their traffic.

**Feasibility**: Fails due to public key authentication.

**Analysis**:
- The client fetches the entry node's public key from the signed DHT advertisement before handshake
- HANDSHAKE_INIT has a two-part format:
  - **Cleartext portion (32 bytes)**: Client Ephemeral Public Key (X25519) — this MUST be cleartext because the node needs it to compute `dh1 = X25519(node_static_priv, client_ephemeral_pub)` before deriving any key material
  - **Encrypted portion**: All other handshake content, encrypted with `init_key` derived from `dh1`
- Only the entity with the node's static private key can decrypt and respond validly
- A MITM attacker who does not have the node's private key cannot produce a valid HANDSHAKE_RESP
- HANDSHAKE_RESP confirmation hash binds both ephemeral keys

**Mitigation**: Public key authentication via X25519 + Ed25519 signed advertisements. Two-part HANDSHAKE_INIT format.

**Status**: Mitigated (requires compromising the node's static private key).

---

### 1.5 Version Downgrade Attack

**Description**: An attacker attempts to force a client and node to use an older, potentially vulnerable protocol version.

**Impact**: Potential exposure to vulnerabilities fixed in newer versions.

**Feasibility**: Possible if version negotiation is not authenticated.

**Mitigation**:
- Version negotiation occurs inside the handshake, which is authenticated
- Version list encrypted in HANDSHAKE_INIT payload (authenticated and encrypted)
- The selected version is included in HANDSHAKE_RESP, which is encrypted and authenticated with the session key material
- An attacker cannot force a different version without breaking authentication
- Clients can specify minimum acceptable version

**Status**: Mitigated.

---

### 1.6 Path Manipulation by Malicious Node

**Description**: A malicious relay node modifies the forwarding path — routing packets to a different next hop than indicated by the onion layer.

**Impact**: Potential path deviation; privacy degradation.

**Feasibility**: Partial — a node can drop or delay packets, but cannot decrypt and re-route them to a different path.

**Analysis**:
- A relay node decrypts its own onion layer and sees the next hop address
- It can choose not to forward (drop) — detected via KEEPALIVE timeout
- It cannot decrypt inner layers to change the deeper path
- It cannot inject itself into a path position it was not assigned

**Mitigation**:
- KEEPALIVE timeout detects non-forwarding nodes
- Reputation system penalizes nodes with high KEEPALIVE timeout rates
- Path is rebuilt when any node goes unresponsive

**Status**: Partially mitigated (drop attacks detectable; inner path tampering impossible).

---

### 1.7 path_id Guessing

**Description**: An attacker guesses a valid path_id to inject return traffic into an active session.

**Impact**: If successful, attacker could inject return traffic.

**Feasibility**: Computationally infeasible.

**Analysis**:
- path_id is 8 bytes (64 bits) of random data from CSPRNG
- Probability ≈ 10,000 / 2^64 ≈ 5.4 × 10^-16 per guess
- Even with correct path_id, injected return traffic would fail ChaCha20-Poly1305 verification at client (wrong session key)

**Mitigation**: Random path_id generation; cryptographic authentication.

**Status**: Mitigated.

---

## 2. Cross-Domain Specific Attack Vectors

### 2.1 Domain Bridge MITM

**Description**: An adversary positions themselves between a domain bridge node and the next relay in the target domain; observes traffic in both domains.

**Impact**: Potential source-destination linkage if adversary controls bridge and observes both sides.

**Feasibility**: Same as any relay MITM.

**Mitigation**:
- All onion layers ChaCha20-Poly1305 encrypted — bridge does not decrypt inner layers
- DOMAIN_BRIDGE layer encrypted to bridge node's session key specifically
- Authentication tags prevent modification at every layer
- Bridge node's signed advertisement binds identity to NodeID
- Bridge node sees only its immediate predecessor in each domain — not the full path

**Status**: Mitigated — same protection as any relay MITM.

---

### 2.2 Fake Mesh Island

**Description**: An adversary creates a fake mesh island with signed (but controlled) island advertisement and controlled relay nodes; attempts to route victim traffic through their infrastructure.

**Impact**: Victim traffic traverses adversary-controlled infrastructure.

**Feasibility**: Possible with significant infrastructure investment.

**Mitigation**:
- MESH_ISLAND_ADVERTISE Ed25519 signed by island curator or bridge nodes
- Individual node verification required for every relay — each "mesh relay" needs valid self-signed advertisement
- Reputation system degrades misbehaving nodes over time
- Pool curator verification for island pools
- Weighted random selection means controlled nodes compete probabilistically
- Domain-scoped pools enable trust isolation for specific islands

**Status**: Partially mitigated. Adversary must operate real nodes with valid self-signed advertisements. Standard adversary model applies once inside controlled island.

---

### 2.3 Bridge Node Sybil Attack

**Description**: An adversary operates many real bridge nodes for a specific domain pair, attempting to dominate all cross-domain traffic for that pair.

**Impact**: Increased probability that adversary-controlled bridges appear in cross-domain paths.

**Feasibility**: Requires significant infrastructure investment.

**Mitigation**:
- Operator diversity: no two bridge nodes with same operator_id in one path
- Weighted random selection: adversary bridge nodes compete with legitimate ones proportionally
- Reputation scoring: bridge nodes with low uptime or failures score lower
- Multiple independent bridge pools discourage single-operator dominance
- Domain-bridge pools enable trust isolation for specific domain pairs

**Status**: Partially mitigated. Requires significant infrastructure investment by adversary. Operator diversity is the primary defense.

---

### 2.4 Domain Isolation Attack (Availability)

**Description**: An adversary takes offline all bridge nodes connecting a mesh island to other domains via DDoS or BGP hijack.

**Impact**: Mesh island becomes isolated from clearnet and other islands.

**Feasibility**: Possible against small deployments.

**Mitigation**:
- Multiple independent bridge nodes (different operators, jurisdictions, ASNs) reduces single-point-of-failure
- LoRa backup links between islands (not dependent on clearnet)
- Store-and-forward for critical messages during outage
- Alternative bridge discovery through unified DHT

**Status**: Availability risk, not a privacy failure. BGP-X cannot protect against physical infrastructure disruption.

---

### 2.5 Cross-Domain Timing Correlation

**Description**: An adversary observes traffic entering clearnet at entry node AND mesh radio traffic at island; attempts to correlate to identify source-destination pairs.

**Impact**: Potential source-destination linkage.

**Feasibility**: Requires global passive adversary with multi-domain observation capability.

**Mitigation**:
- Cover traffic in all domains (COVER uses session_key; externally identical to RELAY)
- KEEPALIVE jitter ±5 seconds (MANDATORY in all domains)
- Domain bridge processing latency adds noise (variable radio propagation)
- Different fundamental latency profiles between clearnet and mesh
- Multiple bridge nodes: adversary cannot know which bridge was used
- Multi-segment pools create multiple trust boundaries

**Status**: Residual risk for global adversaries with multi-domain observation capability. Acknowledged in threat model.

---

### 2.6 Rogue DOMAIN_ADVERTISE Record

**Description**: An adversary publishes fake DOMAIN_ADVERTISE records claiming a bridge between two domains; attempts to be selected as a bridge node.

**Impact**: Adversary-controlled bridge inserted into cross-domain paths.

**Feasibility**: Limited by cryptographic verification.

**Mitigation**:
- DOMAIN_ADVERTISE records Ed25519 signed by the claiming node's private key
- Individual node advertisement verification required before bridge node selection
- Node's own advertisement must confirm it serves both bridged domains
- If bridge doesn't work: KEEPALIVE timeout within 90 seconds; client rebuilds path with different bridge
- Persistent fake advertisements → reputation penalty BRIDGE_CLAIMED_AVAILABLE_OFFLINE

**Status**: Mitigated. Adversary must operate a real node that can actually serve both domains.

---

### 2.7 Mesh Island DHT Poisoning

**Description**: An adversary floods unified DHT with fake MESH_ISLAND_ADVERTISE or DOMAIN_ADVERTISE records.

**Impact**: DHT returns adversary-controlled records.

**Feasibility**: Limited by cryptographic verification.

**Mitigation**:
- All DHT records Ed25519 signed
- Storage nodes MUST verify signatures before storing (unsigned or invalid records rejected)
- Reputation system penalizes nodes publishing fake records (FAKE_DOMAIN_ADVERTISEMENT: -15.0)
- Unified DHT: no separate mesh DHT to poison; mesh nodes access unified DHT via bridge nodes

**Status**: Mitigated at the cryptographic layer.

---

### 2.8 Satellite Timing Correlation

**Description**: High and consistent satellite latency makes timing correlation easier (less randomness to exploit).

**Impact**: Easier timing correlation for satellite-connected paths.

**Feasibility**: Higher risk for satellite paths than terrestrial.

**Mitigation**:
- Cover traffic more effective on satellite links (predictable latency means cover traffic timing is also consistent)
- Satellite gateway tagged in advertisement so paths can be configured appropriately
- Multiple bridge nodes create alternative paths
- N-hop unlimited allows longer paths with more mixing

**Status**: Residual risk. Acknowledged in threat model.

---

## 3. Node Discovery Attacks

### 3.1 DHT Poisoning

**Description**: An attacker floods the DHT with malicious node advertisements to crowd out legitimate nodes and force clients to use attacker-controlled nodes.

**Impact**: Increased probability that attacker-controlled nodes appear in paths.

**Feasibility**: Moderate at scale.

**Mitigation**:
- All advertisements must be signed with valid Ed25519 keys
- Invalid signatures are rejected immediately
- Storage nodes MUST verify signatures before accepting DHT_PUT
- Advertisement TTL limits how long any record persists
- Path selection uses weighted random sampling, not top-N selection
- Diversity constraints limit how many adversary nodes can appear in one path
- Reputation system penalizes new nodes that exhibit poor behavior

**Residual risk**: A very large Sybil operation could gain meaningful presence in the node pool. Mitigated but not eliminated.

**Status**: Partially mitigated.

---

### 3.2 Malicious Bootstrap Node

**Description**: An attacker compromises or replaces a bootstrap node to provide malicious routing table entries to new nodes.

**Impact**: New nodes join an attacker-influenced DHT partition.

**Feasibility**: Possible if bootstrap node list is not secured.

**Mitigation**:
- Bootstrap nodes are hardcoded with their NodeIDs and public keys in the software distribution
- A replacement bootstrap node cannot present a valid advertisement for the original NodeID without the private key
- Clients verify all DHT responses against known public keys
- Multiple geographically distributed bootstrap nodes reduce single-point compromise impact
- DNS-based fallback with DoH (`_bgpx-bootstrap._udp.bgpx.network` TXT records)

**Status**: Mitigated (requires compromising bootstrap node private key).

---

### 3.3 Eclipse Attack

**Description**: An attacker surrounds a target node with attacker-controlled DHT peers, isolating it from the honest DHT.

**Impact**: Target node receives only adversary-curated routing information.

**Feasibility**: Difficult; requires many nodes and precise DHT positioning.

**Mitigation**:
- Kademlia k-bucket replacement policy prefers long-lived nodes over new nodes
- Bootstrap phase uses multiple independent bootstrap nodes
- k=20 (20 nodes per bucket) raises the cost of eclipse attacks vs. k=8
- Periodic bucket refresh queries discover alternative paths to key DHT regions

**Status**: Partially mitigated.

---

## 4. Traffic Analysis Attacks

### 4.1 Timing Correlation

**Description**: An adversary observing traffic entering the overlay (at the entry node) and exiting the overlay (at the exit) correlates packet timing to link sender to receiver.

**Impact**: Source-destination linkage.

**Feasibility**: Requires simultaneous observation of entry and exit.

**Mitigation**:
- Variable path length (more hops = more timing disruption)
- Geographic diversity (physical distance = variable latency)
- Stream multiplexing (packet timing is ambiguous across multiple streams)
- Cover traffic (COVER uses session_key, externally identical to RELAY)
- KEEPALIVE randomization ±5 seconds (MANDATORY in all domains)
- Multi-segment pools (multiple trust boundaries)

**Residual risk**: The primary unsolved challenge for all low-latency anonymity networks. No complete mitigation exists without sacrificing latency.

**Status**: Partially mitigated. Acknowledged fundamental limitation.

---

### 4.2 Volume Correlation

**Description**: An adversary correlates the volume of traffic entering the overlay with volume exiting to link sender to receiver.

**Impact**: Source-destination linkage.

**Feasibility**: Requires simultaneous observation of entry and exit.

**Mitigation**:
- Packet size normalization to size classes
- Cover traffic adds artificial volume
- Stream multiplexing creates volume ambiguity

**Residual risk**: Perfect volume correlation prevention requires fixed-rate transmission, which wastes significant bandwidth and increases latency. BGP-X uses partial mitigations rather than fixed-rate.

**Status**: Partially mitigated.

---

### 4.3 Website Fingerprinting

**Description**: An adversary observing only the traffic between the client and the entry node identifies which website the client is visiting based on the pattern of packet sizes and timings — without breaking encryption.

**Impact**: Destination identification without source-destination linkage.

**Feasibility**: Demonstrated against Tor in academic research under controlled conditions.

**Mitigation**:
- Packet size normalization disrupts fingerprinting features
- Cover traffic adds noise
- Multiple concurrent streams share the same observable traffic pattern
- HTTP/2 multiplexing changes traffic patterns vs HTTP/1.1

**Residual risk**: Website fingerprinting is an active research area. Current defenses raise the false positive rate for attackers but do not eliminate the attack.

**Status**: Partially mitigated.

---

## 5. Pool-Specific Attacks

### 5.1 Pool Poisoning

**Description**: An adversary gains curator identity control; inserts malicious nodes.

**Impact**: Malicious nodes appear in trusted pool.

**Mitigation**:
- Individual node verification: adversary cannot forge individual node Ed25519 signatures
- Reputation system: inserted malicious nodes start at 60.0 and must be caught misbehaving
- Cross-pool diversity: even with poisoned pool in one segment, other segments use different pool
- Pool advertisements signed by curator key; key rotation mechanism available

**Residual risk**: Compromised curator can insert their real (operational) nodes until detected.

---

### 5.2 Pool Sybil

**Description**: An adversary creates many real nodes and convinces curator to include them.

**Impact**: Adversary gains proportional presence in pool.

**Mitigation**:
- Each node starts at 60.0 reputation and must earn higher scores
- Weighted random selection: Sybil nodes don't dominate
- Cross-pool operator diversity limits per-operator presence in a path
- Pool size thresholds: pools below minimum members trigger warning

---

### 5.3 Pool Curator Key Compromise

**Description**: An adversary steals pool curator's private key.

**Impact**: Adversary can sign new pool advertisements with malicious members.

**Mitigation**:
- Key rotation mechanism with dual-signature rotation records
- Emergency rotation with 2-hour acceptance window for compromise_confirmed
- Individual node signatures (curator can add malicious nodes but cannot forge individual signatures)
- Clients detect anomalous pool-level behavior and avoid pool
- 24-hour grace period for member re-verification after rotation

---

### 5.4 Pool Curator Coercion

**Description**: Legal coercion of pool curator to include malicious nodes.

**Impact**: Coerced curator adds adversary nodes to pool.

**Mitigation**:
- Individual node verification still applies (coerced curator cannot bypass)
- Curator can publish pool with honest nodes; coerced curator included "honest-looking" adversary nodes
- Cross-pool diversity limits impact to one segment
- Reputation system catches misbehaving nodes over time
- Pool membership does NOT require jurisdiction declaration

---

## 6. Denial of Service Attacks

### 6.1 Session Flooding

**Description**: An attacker sends many HANDSHAKE_INIT messages to exhaust the target node's session table.

**Impact**: Legitimate clients cannot establish sessions.

**Mitigation**:
- HANDSHAKE_INIT requires decryption with the node's public key — computationally costly for attacker
- Session table has configurable maximum (default 10,000)
- Rate limiting on HANDSHAKE_INIT per source address
- Silent rejection when full (no amplifiable error response)

**Status**: Partially mitigated. High-volume DDoS from many source IPs remains possible.

---

### 6.2 Packet Amplification

**Description**: A small request triggers a large response.

**Impact**: DDoS amplification.

**Analysis**: BGP-X responses approximately match request size. DHT responses bounded. No significant amplification factor.

**Status**: Mitigated.

---

### 6.3 Gateway Overload

**Description**: Opens many streams through single exit gateway.

**Impact**: Exit node becomes unavailable.

**Mitigation**:
- Per-session connection limits
- Total outbound connection cap
- Gateway reputation degrades if overloaded; selected less frequently
- Multiple exit gateways in pool

---

### 6.4 Cross-Domain Path Table Overflow

**Description**: Adversary floods a bridge node with new sessions to exhaust cross-domain path table memory.

**Impact**: Bridge node cannot track return paths.

**Mitigation**:
- Path table entries expire at session idle TTL (90 seconds default)
- Background cleanup every 30 seconds
- Maximum session count configurable (default 10,000)
- Rate limiting on new handshake acceptance
- Separate single-domain and cross-domain path tables limit cross-domain table exposure

**Status**: Mitigated.

---

## 7. Implementation-Level Attacks

### 7.1 Buffer Overflow in Packet Parsing

**Description**: Maliciously crafted packet triggers memory safety vulnerability in packet parsing.

**Impact**: Potential remote code execution on the relay node.

**Mitigation**:
- Reference implementation is written in Rust (memory-safe by default)
- No unsafe Rust in the packet parsing path (enforced by code review policy)
- Packet length fields are bounds-checked before any access
- Fuzzing of the packet parsing pipeline using cargo-fuzz

**Status**: Strongly mitigated by Rust memory safety.

---

### 7.2 Timing Side-Channel in Cryptographic Operations

**Description**: An attacker measures the time taken to process packets and extracts key material from timing differences.

**Impact**: Potential session key recovery.

**Mitigation**:
- ChaCha20-Poly1305 implementation uses constant-time operations
- Session ID lookup uses constant-time comparison
- Authentication tag comparison is constant-time (no early exit on mismatch)
- Cryptographic library selection criteria require constant-time implementations

**Status**: Mitigated (dependent on correctness of cryptographic library implementations).

---

### 7.3 Key Material in Swap Space

**Description**: The OS writes session keys to swap space, allowing an adversary with physical access to the swap partition to recover keys.

**Impact**: Recovery of past session keys.

**Mitigation**:
- Session keys are stored in `mlock`-protected memory on Linux
- `mlock` prevents the OS from swapping the memory page to disk
- On process exit, zeroize is called on all key material before freeing
- Guard pages around key memory

**Status**: Mitigated on Linux. Platform support for memory locking varies.

---

## 8. Implementation-Specific Attacks

### 8.1 PT Bypass

**Description**: Adversary identifies BGP-X traffic despite pluggable transport.

**Impact**: ISP can block or identify BGP-X participation.

**Mitigation**: obfs4-style obfuscation designed for DPI resistance; Elligator2 key encoding; random-length padding; timing jitter.

**Residual risk**: Sufficiently sophisticated DPI with ML may detect patterns.

---

### 8.2 ECH Oracle at Exit Node

**Description**: Timing or error-based oracle attack on ECH implementation at exit node reveals domain name.

**Impact**: Destination domain revealed despite ECH.

**Mitigation**: ECH from audited TLS library. No timing branches on decryption success/failure. Error responses standardized (no domain-specific error detail). Silent retry without ECH before stream failure.

**Residual risk**: Timing difference between ECH and standard TLS handshakes may be measurable at network level.

---

### 8.3 Routing Policy Bypass (LAN)

**Description**: LAN device sends traffic directly to internet, bypassing overlay.

**Impact**: Traffic bypasses privacy protections.

**Mitigation**: BGP-X only mode eliminates all bypass; per-device rules enforce overlay; kill switch blocks non-BGP-X traffic; firewall rules can prevent direct UDP on 7474.

---

### 8.4 Radio Interception (Mesh Domain)

**Description**: Adversary captures mesh radio transmissions.

**Impact**: Adversary obtains encrypted traffic.

**Mitigation**: All BGP-X packets ChaCha20-Poly1305 encrypted with session keys. Radio interceptor sees ciphertext only. Traffic analysis mitigated by cover traffic and KEEPALIVE jitter.

**Status**: Content protected. Timing analysis partially mitigated.

---

### 8.5 Rogue Mesh Beacon

**Description**: Adversary broadcasts fake MESH_BEACON with fake NodeID to poison DHT routing table.

**Impact**: DHT routing table contaminated with adversary nodes.

**Mitigation**: MESH_BEACON is Ed25519 signed. Signature verification mandatory before DHT update. NodeID = BLAKE3(public_key) — forge attempt fails without private key.

**Status**: Mitigated.

---

### 8.6 Gateway Censorship

**Description**: Gateway operator refuses to forward certain traffic.

**Impact**: Certain destinations blocked.

**Mitigation**: Multiple gateways in different jurisdictions; pool-based gateway selection allows avoiding censoring gateways; exit policy versioning enables detecting policy changes; reputation system penalizes unresponsive gateways.

---

### 8.7 Domain Bridge Timing Oracle

**Description**: Adversary measures timing differences between bridge nodes to infer which bridge was selected for specific destination domains.

**Impact**: Adversary learns path selection patterns.

**Mitigation**:
- Bridge selection uses weighted random (not deterministic)
- Radio transmission time for mesh varies (physical randomness)
- Cover traffic in both domains
- KEEPALIVE jitter
- Multiple bridge nodes per domain pair

**Status**: Partially mitigated.

---

## 9. Social and Operational Attacks

### 9.1 Malicious Operator

**Description**: A node operator intentionally operates their node to harm users — for example, logging traffic, manipulating packets, or publishing false reputation data.

**Impact**: Variable depending on the operator's capabilities and position.

**Mitigation**:
- No single operator can see both source and destination (architecture prevents it)
- Packet manipulation is detected via authentication tag failures
- False reputation data requires the operator to have high reputation themselves (their nodes start at 60.0 and must earn more)
- Exit policy violations are detectable and reportable

**Residual risk**: A malicious operator who has established high reputation could temporarily provide degraded service before being detected. The time-to-detection depends on the volume of observations.

**Status**: Partially mitigated.

---

### 9.2 Legal Coercion of Node Operators

**Description**: Law enforcement compels a node operator to log traffic, install surveillance software, or reveal session information.

**Impact**: Potential deanonymization of users whose traffic passed through the coerced node.

**Mitigation**:
- A coerced entry node can log client IPs but cannot see destinations
- A coerced exit node can log destinations but cannot see client IPs
- A coerced relay node cannot see either
- No single node's logs are sufficient to link source to destination
- Coercion of both entry and exit simultaneously would require separate legal orders in different jurisdictions (enforced by country diversity constraint)

**Residual risk**: Simultaneous legal orders in multiple jurisdictions (possible for well-resourced intelligence agencies).

**Status**: Partially mitigated.

---

### 9.3 Pool Curator Coercion

**Description**: Legal coercion of pool curator to include malicious nodes or reveal pool membership information.

**Impact**: Pool becomes untrustworthy.

**Mitigation**:
- Individual node verification still applies
- Pool membership is public (curator cannot reveal what is already public)
- Cross-pool diversity limits impact
- Curator key rotation enables replacing compromised curator

---

## 10. Acceptance Criteria for Attack Simulations

| Attack Scenario | Metric | Required Threshold |
|---|---|---|
| Passive correlation (10% node control) | Success rate | < 15% |
| Passive correlation (10% node control) | False positive rate | > 30% |
| Pool Sybil (20% node control) | Path presence | < 35% |
| Entry + exit control (15% node control) | Same-path entry+exit | < 5% |
| DoS path exhaustion (20% node control) | Path success rate | > 85% |
| Global passive (no cover traffic) | Correlation success | < 60% |
| Global passive (cover 100 Kbps) | Correlation success | < 40% |
| Pool poisoning | Malicious node selection rate | < 20% |
| Rogue beacon attack | Routing table contamination | < 5% |
| Domain bridge Sybil (20% control) | Bridge path presence | < 30% |
| Fake mesh island | Client connection success | < 10% (to fake island) |
| Cross-domain timing (with cover) | Correlation success | < 50% |
| Domain isolation (all bridges down) | Alternative path availability | > 80% (if alternative bridges exist) |
| Routing policy bypass (BGP-X only mode) | Bypass success | 0% |
| ECH oracle | Domain identification rate | < 20% |
