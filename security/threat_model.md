# BGP-X Threat Model

**Version**: 0.1.0-draft

---

## 1. Design Philosophy

BGP-X is an engineering system, not a magic shield. Its privacy properties are bounded by cryptographic assumptions, network conditions, application behavior, and configuration choices. This document states what BGP-X protects, what it does not protect, and under what conditions protection degrades.

---

## 2. Assets to Protect

| Asset | Description |
|---|---|
| Source identity | Who is communicating |
| Destination identity | Who they are communicating with |
| Communication content | What is being communicated |
| Communication metadata | When, how much, how often |
| Participation | The fact that a party uses BGP-X at all |
| Domain name | What website or service is being accessed (via ECH) |

---

## 3. Adversary Classes

### Class A: Passive ISP (Client-Side)

**Capabilities**: observes all traffic from client's internet connection.

**BGP-X protection**: ISP sees UDP traffic to the BGP-X entry node. Cannot determine destination, content, or path. May determine that the client uses BGP-X. Cannot determine which service the client accessed.

### Class B: Active ISP (Client-Side MITM)

**Capabilities**: modifies packets in transit between client and entry node.

**BGP-X protection**: Poly1305 authentication tags detect any modification. MITM attack causes decryption failure and silent drop — no session established. Authentication tags verified at every hop.

### Class C: Single Malicious Relay Node

**Capabilities**: adversary controls one relay node in the path.

**BGP-X protection**: relay nodes know only predecessor and successor — not source, not destination, not path length. Cannot link client to service.

### Class D: Multiple Colluding Relay Nodes (Minority)

**Capabilities**: adversary controls multiple relay nodes but not entry+exit simultaneously.

**BGP-X protection**: diversity constraints (ASN, country, operator) prevent a single adversary from controlling multiple positions on one path. If diversity constraints are satisfied and adversary controls some nodes, they cannot reconstruct the full path.

### Class E: Entry + Exit Simultaneously

**Capabilities**: adversary controls both the entry node (knows client IP) and exit node (knows destination).

**BGP-X protection**: pool/domain diversity and operator diversity constraints reduce the probability that the same adversary controls both positions. This is an acknowledged residual risk. N-hop paths with stronger diversity further reduce probability.

### Class F: Global Passive Adversary (Internet Only)

**Capabilities**: observes all internet traffic worldwide.

**BGP-X protection**: content encrypted end-to-end. Timing correlation is possible. Cover traffic (COVER uses session_key; externally identical to RELAY) and KEEPALIVE jitter (±5 seconds, MANDATORY) disrupt timing signals. This is the primary residual risk for single-domain clearnet paths.

### Class G: Malicious Exit Node

**Capabilities**: controls the clearnet exit; observes destination.

**BGP-X protection**: exit node sees destination but NOT source identity. ECH hides domain name from network-layer observers. HTTPS hides content. Exit policy limits destinations the node will serve.

### Class H: Pool Curator Compromise

**Capabilities**: adversary compromises a pool curator key; can sign fake member records.

**BGP-X protection**: individual node verification provides fallback — each node's own advertisement is independently verified. Emergency key rotation with 2-hour window. Cross-pool diversity limits damage to one path segment.

### Class I: Compromised Client Device

**Capabilities**: adversary has full access to the client device.

**BGP-X protection**: none. BGP-X cannot protect against a compromised device.

### Class J: Application-Layer Leakage

**Capabilities**: application leaks identifying information (cookies, JavaScript, etc.).

**BGP-X protection**: none. BGP-X provides network-level privacy; application behavior is out of scope.

### Class K: Rogue Mesh Beacon

**Capabilities**: adversary broadcasts MESH_BEACON with fake NodeID claiming to be a legitimate node.

**BGP-X protection**: MESH_BEACON is Ed25519 signed. Signature verification mandatory before DHT routing table update. Forge attempt fails without the legitimate node's private key.

### Class L: Physical Radio Interception

**Capabilities**: adversary intercepts mesh radio frames.

**BGP-X protection**: all BGP-X packets are ChaCha20-Poly1305 encrypted with session keys. Radio interceptor sees ciphertext only. Cannot decrypt without session key. Content is protected.

### Class M: DHT Poisoning (Standard)

**Capabilities**: adversary floods DHT with fake node advertisements.

**BGP-X protection**: all DHT records are Ed25519 signed. NodeID = BLAKE3(public_key) — fake NodeID requires breaking hash function. Individual node verification required before path selection.

### Class N: Pluggable Transport Fingerprinting

**Capabilities**: adversary identifies BGP-X traffic patterns via DPI even with PT enabled.

**BGP-X protection**: PT obfuscates patterns. BGP-X's underlying security is unchanged if PT is bypassed. Adversary who identifies BGP-X traffic still cannot decrypt it or link endpoints.

### Class P: Cross-Domain Traffic Correlation

**Capabilities**: adversary controls a domain bridge node AND observes traffic in both connected domains; attempts to correlate clearnet and mesh traffic.

**BGP-X protection**:
- Bridge node sees only its immediate predecessor in each domain, not the full path
- path_id used for return routing does not encode path composition
- Cover traffic uses session_key (externally identical) in both domains
- KEEPALIVE jitter (±5 seconds, MANDATORY) in all domains
- Bridge processing latency (radio propagation time) adds noise
- Operator diversity limits single adversary's bridge presence — no single operator can control all bridges for a domain pair

**Residual risk**: adversary controlling the ONLY bridge node for a domain pair and observing both sides can attempt timing correlation.

### Class Q: Rogue Mesh Island

**Capabilities**: adversary creates a controlled mesh island with a signed island advertisement and attractive bridge nodes; attempts to attract traffic.

**BGP-X protection**:
- Island advertisements are Ed25519 signed (by island curator or bridge nodes)
- Individual node verification required for every relay selected in any segment
- Reputation system penalizes misbehaving nodes
- Pool membership in island pool requires curator signature and individual node verification

**Residual risk**: adversary who operates a real, functioning controlled island can attract traffic — standard relay adversary model applies within.

### Class R: Mesh Island Isolation Attack

**Capabilities**: adversary disrupts all bridge nodes connecting an island to other domains (DDoS, BGP hijack of WAN IPs).

**BGP-X protection**:
- Multiple independent bridge nodes per island from different operators and ASNs reduces single-point-of-failure
- LoRa fallback transport when primary WiFi bridge fails
- Island continues to operate internally when isolated

**Residual risk**: if all bridge nodes are simultaneously disabled, the island loses cross-domain connectivity. This is an availability failure, not a privacy failure.

### Class S: Cross-Domain Global Passive Adversary

**Capabilities**: global passive adversary simultaneously observes all clearnet internet traffic AND all mesh radio transmissions in all islands.

**BGP-X protection**:
- Cover traffic (session_key used; externally identical to RELAY) in all domains
- KEEPALIVE jitter in all domains
- Different fundamental latency profiles between clearnet and mesh (harder to correlate)
- Multiple cross-domain path options: adversary cannot know which bridge was used without observing all bridges

**Residual risk**: with truly global observation capability across multiple physical domains simultaneously, timing correlation is possible. This is acknowledged as a fundamental limitation of all low-latency anonymity networks. BGP-X reduces but cannot eliminate this risk.

---

## 4. Protection Matrix

| Asset | A: Passive ISP | C: Single Relay | E: Entry+Exit | F: Global Passive | P: Cross-Domain | S: Multi-Domain Global |
|---|---|---|---|---|---|---|
| Source identity | ✅ | ✅ | ⚠️ | ✅ | ✅ | ⚠️ |
| Destination | ✅ | ✅ | ⚠️ | ✅ | ✅ | ⚠️ |
| Content | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Metadata | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Participation | ⚠️ visible | N/A | N/A | ⚠️ visible | N/A | ❌ visible |
| Domain name | ✅ (ECH) | ✅ | ✅ | ✅ | ✅ | ✅ |

✅ = Protected. ⚠️ = Residual risk mitigated. ❌ = Not protected.

---

## 5. Sybil Attack Resistance

Cross-domain diversity enforcement limits Sybil attacks across domain boundaries:
- Sybil nodes in clearnet cannot also dominate mesh segments (cross-domain operator diversity enforced)
- A Sybil operator controlling many bridge nodes is limited to one bridge position per path
- Domain-bridge pool curator verification adds independent verification layer

---

## 6. What BGP-X Is NOT Designed to Defeat

- Compromise of the client's device
- Application-layer identity leakage (JavaScript, cookies, login sessions)
- Cross-domain timing correlation by adversaries monitoring multiple network domains simultaneously
- Complete unavailability of bridge nodes for a required domain pair (availability failure)
- Physical disruption of mesh radio infrastructure
- Legal coercion of node operators

---

## 7. Trust Assumptions

- Client device is not compromised
- ChaCha20-Poly1305, X25519, Ed25519, BLAKE3, HKDF-SHA256 are not broken
- The random number generator is not predictable
- At least one honest node exists in each position for diversity constraints to provide protection
- Domain bridge node advertisements are accurate (verified cryptographically; misbehavior degrades reputation)
- Bridge node operators for a given domain pair are not all colluding (enforced by diversity constraints)
