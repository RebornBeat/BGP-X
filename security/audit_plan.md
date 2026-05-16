# BGP-X Security Audit Plan

**Version**: 0.1.0-draft

This document specifies the security audit plan for BGP-X — what will be audited, by whom, when, and how findings will be handled.

---

## 1. Overview

BGP-X handles cryptographic operations and privacy-critical routing for users in potentially high-risk situations. Independent security audits are required before any production deployment.

All features in v1 are included in scope — nothing is deferred. The audit plan covers the complete v1 specification including cross-domain routing, unified DHT, N-hop unlimited, domain bridge nodes, and mesh island architecture.

---

## 2. Audit Scope

### 2.1 Protocol Specification Audit

**Target**: All BGP-X protocol specification documents (this repository)

**Focus areas**:
- Session key derivation correctness (dh1 || dh2 → HKDF chain)
- Forward secrecy properties (ephemeral key destruction ordering)
- Handshake authentication guarantees (two-part HANDSHAKE_INIT structure; why cleartext pubkey is safe)
- Nonce uniqueness and reuse resistance (two independent sequence spaces; per-session keys)
- Onion encryption correctness (path_id in all layers; return path architecture; no re-encryption at domain boundaries)
- Replay protection (bidirectional sequence windows; 64-position sliding window)
- Pool trust model (individual verification requirement; trust non-transitivity)
- Pool curator key rotation (dual-signature security; grace period correctness)
- ECH negotiation security (no ECH oracle via timing; correct fallback behavior)
- Cover traffic indistinguishability (session_key = COVER key; same in all domains)
- **Domain-agnostic crypto**: verify that identical key derivation in all domains is specified correctly and there are no domain-specific key paths
- **Cross-domain onion construction**: DOMAIN_BRIDGE hop type; no re-encryption at bridges; inner layers remain encrypted
- **Cross-domain path_id routing**: return traffic across domain boundaries; bridge node cross-domain path table; opaque forwarding
- **Unified DHT security**: replacing two-DHT model; domain bridge record authenticity; mesh island record authenticity; symmetric key derivation for bridge records
- **N-hop unlimited**: no protocol mechanisms that could impose implicit hop limits; no hop counter fields that could be exploited

**When**: Before reference implementation begins.

---

### 2.2 Reference Implementation Audit (Rust Core)

**Target**: bgpx-node, bgpx-client (control client), bgpx-sdk

**Focus areas**:
- Memory safety (unsafe Rust usage, if any)
- Cryptographic constant-time operations
- Key material handling (zeroization order, mlock)
- Input validation and parsing (onion layers, mesh fragments, pool advertisements)
- Session state machine correctness
- Path table (path_id → predecessor) race conditions and data structure correctness
- **Cross-domain path_id table**: `CrossDomainPathTable` implementation; race conditions in concurrent insert/lookup; expiry correctness; bridge node cross-domain entry creation in HANDSHAKE_DONE
- **DomainBridgeManager**: transition correctness; no re-encryption; correct return routing; bridge availability tracking
- **MeshIslandManager**: DHT publication correctness; island advertisement timing
- Stream ID parity enforcement
- Session re-handshake key migration: old keys zeroized only after streams migrate
- **Unified DHT**: no per-domain DHT instances; single routing table
- ForwardingPipeline concurrency: sharded path tables; receiver/worker/sender thread model

**When**: Before testnet launch.

---

### 2.3 Network Protocol Fuzzing

**Target**: Packet parsing pipeline in bgpx-node

**Method**: Structure-aware fuzzing (cargo-fuzz/libFuzzer), mutation-based, coverage-guided

**Focus areas**:
- Malformed common headers
- Truncated onion layers
- Invalid path_id (all zeros)
- Invalid pool advertisements
- Fragment reassembly edge cases (out of order, duplicates, timeout)
- Oversized packets
- **Invalid domain_id type values** (unknown type bytes)
- **DOMAIN_BRIDGE at non-bridge-capable node** (should silently drop)
- **MESH_ENTRY/MESH_EXIT/MESH_RELAY at non-mesh node** (should silently drop)
- **Invalid domain bridge records** (malformed domain_id pairs, invalid signatures)
- **Mesh island advertisement fuzzing** (invalid island_id, missing fields)
- **Cross-domain path_id table overflow** (many concurrent path_id entries)

**When**: Concurrent with implementation audit.

---

### 2.4 Cross-Domain Routing Audit (NEW)

**Target**: Domain bridge manager, cross-domain path table, domain transition logic

**Focus areas**:
- **No re-encryption at domain boundaries**: verify bridge node forwards inner onion layers unchanged
- **Cross-domain path_id table correctness**: correct (source_domain, source_addr) → (target_domain, target_addr) mapping; correct return routing from both sides
- **Bridge node cross-domain path table initialization in HANDSHAKE_DONE**: path_id correctly extracted; cross-domain entry created before any traffic forwarded
- **Clearnet client reaching mesh service**: verify privacy properties (entry node sees client IP; bridge node sees only predecessor addresses; mesh service sees only last mesh relay)
- **Any-to-any routing validation**: clearnet→mesh, mesh→clearnet, mesh→mesh all work correctly
- **N-hop unlimited verification**: no implicit hop limits in parser, session state machine, or forwarding pipeline
- **Domain bridge availability**: unavailable bridge correctly marked; traffic not silently discarded; ERR_DOMAIN_BRIDGE_UNAVAILABLE returned
- **Unified DHT**: verify no separate per-domain DHT instances created; bridge records use symmetric key derivation correctly

**When**: Before testnet launch, concurrent with implementation audit.

---

### 2.5 Pool Security Audit

**Target**: Pool advertisement handling, membership verification, key rotation

**Focus areas**:
- Pool signature verification completeness
- Individual node verification independence from pool membership
- Cross-pool diversity enforcement
- Pool curator key rotation dual-signature verification
- Pool member re-verification 24hr grace period logic
- Pool fallback to default logic
- **Domain-scoped pool validation**: domain_scope field correctly restricts member nodes
- **Domain-bridge pool validation**: bridges field correctly verified; members must serve both bridged domains
- **Pool individual verification for bridge pools**: bridge capability checked per-node, not just per-pool

**When**: Before testnet launch.

---

### 2.6 ECH Security Audit

**Target**: Exit node ECH implementation

**Focus areas**:
- Correct ECH negotiation behavior
- Fallback behavior security (no ECH oracle via timing differences)
- DoH for ECH config retrieval (DNS query routing through overlay when possible)
- DNSSEC validation
- ECS stripping completeness (verify no client subnet in DNS queries)
- ECH retry logic (one retry without ECH if require_ech=false)
- ECH non-applicability for mesh island native services (no TLS ClientHello)

**When**: Before testnet launch.

---

### 2.7 Mesh Transport Audit

**Target**: Mesh transport layer, beacon verification, fragment reassembly, domain bridge radio

**Focus areas**:
- MESH_BEACON signature verification before DHT routing table update
- MESH_FRAGMENT reassembly security (buffer overflow, timeout, duplicate fragments)
- Transport MTU enforcement
- Multi-transport selection security
- Meshtastic adaptation layer security (serial protocol boundary)
- **Domain bridge radio forwarding**: no information leakage from clearnet to mesh radio
- **Bridge node cross-domain path_id routing via radio**: correct opaque forwarding of return blobs across domain boundary

**When**: Before mesh-mode testnet launch.

---

### 2.8 Pluggable Transport Audit

**Target**: PT subprocess interface and built-in obfs4-style PT

**Focus areas**:
- PT subprocess isolation (cannot affect main daemon security)
- Elligator2 implementation correctness
- Random padding distribution uniformity
- PT bypass resistance
- No timing oracle through PT error handling

**When**: Before testnet launch.

---

### 2.9 Geographic Plausibility Audit

**Target**: RTT measurement and scoring system

**Focus areas**:
- RTT measurement constant-time (no timing side channels)
- Domain-specific threshold table correctness (clearnet vs WiFi mesh vs LoRa vs satellite)
- Bridge node independent per-domain scoring correctness
- Satellite exempt correctly implemented
- Score integration with path selection (correct weighting; no hard exclusion)

**When**: Concurrent with implementation audit.

---

### 2.10 Return Path Architecture Audit

**Target**: Cross-domain path_id routing, bridge node return handling

**Focus areas**:
- **Intermediate relay opacity for return traffic**: verify no relay decrypts return blobs
- **Bridge node cross-domain return routing**: correct lookup in cross_domain_path_table; correct transport selection; correct forwarding direction
- **Two-table isolation**: single-domain path_table and cross_domain_path_table are correctly isolated; no cross-contamination
- **Path_id expiry**: both tables expire entries correctly; stale entries silently dropped
- **Concurrent access**: path tables are thread-safe under high concurrency

**When**: Before testnet launch.

---

### 2.11 Routing Policy Engine Audit

**Target**: Routing policy evaluation and enforcement

**Focus areas**:
- Rule evaluation order correctness (first match wins)
- Policy bypass resistance
- DNS routing policy security
- SDK socket TCP authentication enforcement
- Privacy of routing statistics (no identifiers in aggregate stats)
- **Domain-aware rules**: domain_segments specifications correctly passed to path construction

**When**: Before testnet launch.

---

### 2.12 Exit Gateway Audit

**Target**: Exit node/gateway implementation

**Focus areas**:
- Exit policy enforcement correctness (cannot be bypassed)
- Private IP range blocking completeness (SSRF prevention)
- Connection limit enforcement
- Logging policy technical enforcement (no access logs when logging_policy="none")
- DNS DoH with DNSSEC and ECS stripping
- ECH implementation
- Exit policy versioning
- **Exit node context in cross-domain paths**: verify exit node behavior is identical regardless of how many domains the path traversed before reaching the exit

**When**: Before testnet launch.

---

### 2.13 Anonymity Analysis (Academic Collaboration)

**Target**: Protocol design, especially cross-domain paths

**Method**:
- Formal analysis of anonymity properties in single-domain and cross-domain configurations
- Timing/volume correlation simulation against cross-domain paths
- Comparison to Tor under equivalent adversary
- Analysis of domain bridge node as attack surface (adversary controlling bridge)
- N-hop unlimited analysis (does path length beyond default 4 meaningfully increase anonymity)

**When**: Concurrent with testnet, before mainnet.

---

## 3. Auditor Requirements

Auditors are selected based on:

1. **Relevant expertise**: Demonstrated by published research, CVEs, or prior audit reports in the relevant domain
2. **Independence**: No financial or personal relationship with BGP-X project leadership
3. **Reputation**: Known and respected in the security research community
4. **Availability**: Can complete the audit within the required timeline

**Protocol audit**: Expertise in applied cryptography; anonymity network design; cross-domain routing security; protocol analysis (formal methods preferred).

**Implementation audit**: Rust security auditing; async Rust (Tokio) concurrency; cryptographic library integration; cross-domain routing implementations.

**Mesh/radio audit**: Embedded systems security; radio protocol analysis; LoRa/WiFi mesh.

Candidate audit firms for implementation audits (non-exhaustive, no endorsement implied):
- Trail of Bits
- NCC Group
- Cure53
- Quarkslab
- Security Research Labs

Candidate academic groups for anonymity analysis:
- EPFL SPRING Lab
- UCL Information Security Group
- Princeton CITP
- ETH Zurich System Security Group

---

## 4. Finding Classification

Findings are classified by severity:

| Severity | Description | Resolution Requirement |
|---|---|---|
| Critical | Allows deanonymization of users or remote code execution | Fix before any public deployment |
| High | Significantly weakens stated security properties; cross-domain routing compromise | Fix before testnet launch |
| Medium | Partially weakens security properties, enables DoS, or domain isolation bypass | Fix before mainnet |
| Low | Minor weakness or defense-in-depth improvement | Fix in next release cycle |
| Informational | Best practice suggestion; no direct security impact | Document and address in future |

---

## 5. Finding Handling Process

```
1. Auditor submits finding to BGP-X security team (private channel)

2. BGP-X team acknowledges within 48 hours

3. BGP-X team reproduces and assesses severity

4. Fix is developed and reviewed:
   ├── Critical/High: fix developed in private branch
   ├── Medium/Low: fix developed in main branch with coordinated disclosure date

5. Fix is reviewed by auditor before disclosure

6. Coordinated disclosure:
   ├── Critical: 90 days from finding, or when fix deployed (whichever is earlier)
   ├── High: 90 days
   ├── Medium/Low: 180 days or next release cycle

7. Public disclosure:
   ├── Audit report published in full
   ├── CVE assigned for Critical/High findings
   ├── Changelog updated
   └── Security advisory published
```

---

## 6. Test Vectors

Test vectors will be published as machine-readable JSON files in `/protocol/test_vectors/`. Required test vectors:

### 6.1 Cryptographic Primitives

- X25519 key exchange: 10 test vectors from RFC 7748
- Ed25519 sign + verify: 10 test vectors from RFC 8032
- ChaCha20-Poly1305: 10 test vectors from RFC 8439
- BLAKE3: 10 test vectors from BLAKE3 specification
- HKDF-SHA256: 10 test vectors from RFC 5869

### 6.2 BGP-X-Specific Operations

- Session key derivation (all four HKDF calls): 5 vectors each
- Onion construction (3, 4, 5-hop paths with path_id): 3 vectors each
- Onion decryption per hop: vectors matching construction
- HANDSHAKE_INIT (two-part format): 3 vectors
- HANDSHAKE_RESP: 3 vectors
- HANDSHAKE_DONE (with path_id): 3 vectors
- Full handshake round-trip: 3 vectors
- Node advertisement sign/verify: 5 vectors
- Pool advertisement sign/verify: 5 vectors
- Pool member verification: 5 vectors
- ECH stream flag encoding/decoding: 3 vectors each
- Geographic plausibility scoring: 5 vectors
- PATH_QUALITY_REPORT generation/decryption: 3 vectors
- MESH_BEACON sign/verify: 3 vectors
- MESH_FRAGMENT fragmentation/reassembly: 5 vectors each
- Pool curator key rotation (valid): 3 vectors
- NODE_WITHDRAW sign/verify: 3 vectors
- path_id generation (never all-zeros): 5 vectors
- Two independent sequence spaces: 3 vectors

### 6.3 Negative Cases (MUST REJECT)

- Replay detection: 5 vectors
- Authentication failure: 5 vectors
- Expired advertisements: 3 vectors
- Invalid signatures: 5 vectors
- Invalid path_id (all zeros): 3 vectors
- Stream ID parity violation: 4 vectors
- Oversized packets: 3 vectors
- Pool curator key rotation (expired): 3 vectors
- Pool curator key rotation (invalid signature): 3 vectors
- Pool curator key rotation (wrong old key): 2 vectors
- Fragmented packet reassembly timeout: 2 vectors

### 6.4 Cross-Domain Specific (NEW)

1. DOMAIN_BRIDGE onion layer construction: 5 vectors per domain pair type (clearnet→mesh, mesh→clearnet, mesh→mesh, clearnet→satellite)
2. Bridge node onion unwrapping and cross-domain forwarding: matching vectors
3. **No re-encryption verification**: constructed inner layers at bridge == forwarded inner layers (bit-for-bit identical)
4. Cross-domain path_id table entry creation at bridge: 5 vectors
5. Cross-domain return routing: 3 vectors (return blob arrives at bridge → routed to correct predecessor domain)
6. N-hop unlimited path construction (10, 15, 20 hops): 3 vectors each
7. Negative: DOMAIN_BRIDGE at non-bridge node (must silently drop): 3 vectors
8. Negative: invalid domain_id type in DOMAIN_BRIDGE (must silently drop): 3 vectors
9. PATH_QUALITY_REPORT with domain_id field (20-byte payload): 3 vectors
10. DOMAIN_ADVERTISE sign and verify: 3 vectors
11. MESH_ISLAND_ADVERTISE sign and verify: 3 vectors
12. Unified DHT key computation for bridge records (symmetric ordering): 5 vectors

All implementations MUST pass all published test vectors before being considered specification-compliant.

---

## 7. Ongoing Security Program

After mainnet launch, BGP-X will maintain:

### 7.1 Bug bounty program

A public bug bounty program will be established with rewards:

| Severity | Reward Range |
|---|---|
| Critical | $10,000 – $50,000 |
| High | $2,000 – $10,000 |
| Medium | $500 – $2,000 |
| Low | $100 – $500 |

Cross-domain routing vulnerabilities (deanonymization via domain bridge, mesh island isolation bypass, cross-domain path composition leakage) are classified at Critical/High by default.

### 7.2 Annual re-audit

A full re-audit of the reference implementation will be commissioned annually, or within 60 days of any MAJOR protocol version release.

### 7.3 Dependency monitoring

All cryptographic dependencies are monitored via:
- `cargo-audit` (Rust advisory database)
- Automated dependency update PRs (Dependabot or equivalent)
- Manual review of any cryptographic library update before merging
