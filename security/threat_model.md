# BGP-X Threat Model

**Version**: 0.1.0-draft

This document enumerates specific attack vectors against BGP-X, provides analysis of each attack's feasibility, specifies the mitigations built into the protocol and implementation, and states precisely what BGP-X protects, what it does not protect, and under what conditions protection degrades.

---

## 1. Design Philosophy

BGP-X is an engineering system, not a magic shield. Its privacy properties are bounded by cryptographic assumptions, network conditions, application behavior, and configuration choices.

BGP-X is also a **worldwide system**. Users constrain themselves to their jurisdiction through their configuration choices — the protocol does not impose jurisdiction-specific restrictions. Geographic plausibility scoring is **OPTIONAL**; nodes are not required to declare jurisdiction. Satellite-connected nodes are automatically exempt. The system functions identically globally.

---

## 2. Assets to Protect

| Asset | Description |
|---|---|
| Source identity | Who is communicating (real IP address or mesh transport address) |
| Destination identity | Who they are communicating with |
| Communication content | What is being communicated |
| Communication metadata | When, how much, how often, timing patterns |
| Participation | The fact that a party uses BGP-X at all |
| Domain name | What website or service is being accessed (via ECH) |
| Path composition | Which nodes, domains, and pools constitute a path |
| Cross-domain traversal | Whether and how traffic crosses routing domains |

---

## 3. Adversary Classes

### Class A: Passive ISP (Client-Side)

**Who**: The client's Internet Service Provider, or any network operator on the path between the client and the BGP-X entry node.

**Capabilities**:
- Observe all traffic entering and leaving the client's connection
- See source and destination IPs of all packets
- See volume and timing of all packets
- Cannot decrypt encrypted payloads

**Goals**:
- Identify what services the client accesses
- Build behavioral profiles
- Comply with surveillance orders (passive logging)

**BGP-X protection**:
- ISP sees only: encrypted UDP to entry node IP
- ISP sees: that the client uses BGP-X (UDP/7474)
- ISP cannot see: destinations, content, path structure, cross-domain destinations

**Residual risk**: ISP knows participation in BGP-X network. Pluggable transport mitigates this by obfuscating BGP-X traffic patterns.

---

### Class B: Active ISP (Client-Side MITM)

**Who**: Same as Class A, but capable of modifying or injecting traffic.

**Capabilities**:
- All of Class A
- Inject or modify packets
- Perform SSL stripping attacks (not relevant if clients use HTTPS)
- Block BGP-X traffic

**Goals**:
- Break BGP-X connections
- Force clients onto non-private paths
- Inject malicious content
- Deny service

**BGP-X protection**:
- Poly1305 authentication tags detect modification
- Modified packets dropped silently at entry node
- Injection attacks fail without valid session keys
- BGP-X cannot prevent traffic blocking (ISP can block port 7474)
- Pluggable transport extension mitigates blocking by obfuscating patterns

---

### Class C: Single Malicious Relay Node

**Who**: An adversary who operates one node in the BGP-X network — entry, relay, or exit.

**Capabilities**:
- Observe all traffic passing through their node
- Know the IP address of predecessor and successor in any path
- Attempt to inject or drop packets
- Measure traffic timing and volume

**Goals**:
- Link source to destination
- Deanonymize specific users
- Degrade network performance

**BGP-X protection**:
- Entry node: knows client IP, NOT destination (encrypted in inner layers)
- Exit node: knows destination, NOT client IP
- Middle relay: knows neither source nor destination
- Single relay cannot link source to destination
- Packet injection fails (no session key for other hops)
- Packet dropping detected via KEEPALIVE timeout → path rebuild
- path_id in every layer enables return routing without revealing path composition

---

### Class D: Multiple Colluding Relay Nodes (Minority)

**Who**: An adversary who operates multiple BGP-X nodes — up to but not including all nodes in a specific path.

**Capabilities**:
- All of Class C for each controlled node
- Correlate observations across their nodes
- Know the path segment between their nodes
- Measure timing correlation between their positions

**Goals**:
- Link source to destination by controlling multiple positions
- Identify users by observing partial path segments

**BGP-X protection**:
- Path diversity constraints (ASN, country, operator) prevent a single entity from controlling multiple hops
- Pool-based trust isolation limits cross-pool collusion
- If adversary controls nodes A and C in 4-hop path [A, B, C, D]:
  - They know: client → A, C → D
  - They do NOT know: what is between B and C
  - Full link still requires both endpoints (which they lack)
- The adversary CANNOT link source to destination unless controlling BOTH entry AND exit simultaneously

**Residual risk**:
- If diversity constraints are satisfied with falsified advertisement data (ASN spoofing), a single operator could place two nodes in a path
- The reputation system detects ASN inconsistencies over time
- Individual node verification required regardless of pool membership

---

### Class E: Entry + Exit Simultaneously Controlled

**Who**: An adversary who controls both the entry node AND the exit node of a specific client's path.

**Capabilities**:
- Entry node: knows client IP, timing of outbound traffic
- Exit node: knows destination, timing of outbound traffic
- Can correlate the two observations to link client to destination

**Goals**:
- Deanonymize the client completely
- Identify which destination corresponds to which client

**BGP-X protection**:
- Operator diversity enforcement prevents same entity at both ends
- ASN diversity prevents same network at both ends
- Country diversity prevents same jurisdiction at both ends
- Pool-based paths use different pools for different segments
- Cover traffic disrupts timing correlation

**Residual risk**: A well-resourced adversary (state-level, large-scale infrastructure) who controls many nodes across many ASNs and countries could appear at both ends despite diversity constraints. This is a known fundamental limitation of multi-hop overlay networks. Use longer paths (N-hop unlimited) with stronger diversity for higher-value traffic.

---

### Class F: Global Passive Adversary (Internet Only)

**Who**: An adversary with visibility into all internet traffic globally — e.g., a nation-state intelligence agency with access to major internet exchange points.

**Capabilities**:
- Observe all traffic on the public internet
- See all BGP-X packets entering overlay (client → entry)
- See all packets exiting overlay (exit → destination)
- Perform timing correlation between entry and exit
- Build comprehensive surveillance of BGP-X users

**Goals**:
- Link any client to any destination
- Map the BGP-X network topology
- Identify participants

**BGP-X protection**:
- End-to-end encryption prevents content observation
- Multi-hop routing prevents simple source-destination IP correlation
- Cover traffic (same session_key as RELAY, externally identical) disrupts timing
- KEEPALIVE jitter (±5 seconds, MANDATORY) prevents keepalive timing fingerprinting
- Stream multiplexing creates timing ambiguity

**Residual risk**: A true global passive adversary can attempt traffic correlation attacks. BGP-X is a low-latency network — low latency and strong anonymity are fundamentally in tension. Cover traffic significantly raises the cost but cannot eliminate correlation. This is the primary residual risk, shared with all practical low-latency anonymity networks including Tor.

---

### Class G: Malicious Exit Node

**Who**: An adversary who operates a BGP-X exit node.

**Capabilities**:
- Knows the destination (IP or domain)
- Observes TLS handshake (unless ECH)
- Can perform TLS interception if client accepts invalid certificates
- Can inject content into unencrypted protocols
- Can block specific destinations

**Goals**:
- Observe destinations
- Perform content injection
- Block access to specific services
- Build destination profiles

**BGP-X protection**:
- Exit sees destination but NOT source identity
- ECH hides domain name from SNI when destination supports it
- HTTPS hides content
- Exit policy limits destinations
- Pool-based exit selection allows avoiding untrusted exits

**Residual risk**: Exit sees destination IP. ECH hides domain from network observers but exit sees TLS handshake. A malicious exit could attempt certificate substitution (client should verify certificates properly).

---

### Class H: Pool Curator Compromise

**Who**: An adversary who compromises a pool curator's Ed25519 private key.

**Capabilities**:
- Sign fake pool advertisements
- Include malicious nodes in pool
- Exclude legitimate nodes
- Rotate keys (legitimately or maliciously)

**Goals**:
- Influence path selection
- Place adversary nodes in trusted pools
- Degrade network trust model

**BGP-X protection**:
- Individual node advertisement verification required (curator cannot bypass this)
- Cross-pool diversity limits damage to one segment
- Reputation system penalizes nodes included by malicious curators
- Key rotation mechanism with dual-signature for scheduled rotation
- 2-hour emergency window for confirmed compromise
- 24-hour grace period for old-key-signed members

**Residual risk**: A compromised curator can include adversary nodes in the pool until detected. Individual node verification prevents pure pool-based attack. Curator can exclude nodes but cannot force their inclusion in paths without node-level acceptance.

---

### Class I: Compromised Client Device

**Who**: An adversary with access to the client's device — malware, physical access, or OS-level compromise.

**Capabilities**:
- Read memory including session keys
- Observe traffic before encryption
- Modify the BGP-X client software
- Log all activity before BGP-X processes it

**Goals**:
- Extract session keys
- Observe application-layer data
- Disable protections silently
- Correlate with network observations

**BGP-X protection**: None. BGP-X cannot protect against a compromised device. This is out of scope by design.

---

### Class J: Application-Layer Identity Leakage

**Who**: Not a network adversary — the application itself revealing identity.

**Examples**:
- Logging into a service with a real username over BGP-X
- Browser fingerprinting (JavaScript, canvas, fonts)
- Cookies from non-BGP-X sessions
- WebRTC leaking real IP address
- Application telemetry and analytics

**BGP-X protection**: None. BGP-X operates at the network layer. It cannot prevent applications from leaking identity.

---

### Class K: Rogue Mesh Beacon

**Who**: An adversary broadcasting MESH_BEACON with fake NodeID claiming to be a legitimate mesh node.

**Capabilities**:
- Advertise non-existent services
- Attract traffic to adversary-controlled nodes
- Attempt to poison mesh DHT

**Goals**:
- Attract traffic for surveillance
- Disrupt mesh connectivity
- Perform MITM at mesh layer

**BGP-X protection**:
- MESH_BEACON is Ed25519 signed
- Signature verification mandatory before DHT routing table update
- Forge attempt fails without legitimate node's private key
- NodeID = BLAKE3(public_key) — cannot fake without breaking hash

---

### Class L: Physical Radio Interception (Mesh)

**Who**: An adversary with radio equipment capable of receiving BGP-X mesh transmissions (WiFi 802.11s, LoRa, BLE).

**Capabilities**:
- Record all mesh radio frames
- Store for later analysis
- Attempt timing correlation across radio domain

**Goals**:
- Observe mesh traffic patterns
- Correlate with internet observations
- Identify mesh participants

**BGP-X protection**:
- All BGP-X packets encrypted with ChaCha20-Poly1305 using session keys
- Radio capture yields only ciphertext
- Cannot decrypt without session key
- Cover traffic uses same session_key — externally identical to real traffic

**Residual risk**: Adversary can confirm participation in BGP-X mesh. Timing correlation between radio capture and internet observation is possible — see Class S.

---

### Class M: Rogue Mesh Node

**Who**: An adversary who introduces an unauthorized node into a mesh network.

**Capabilities**:
- Operate a node with valid signed advertisement
- Participate in mesh routing
- Observe traffic passing through

**Goals**:
- Observe mesh traffic
- Attempt to link participants
- Degrade mesh performance

**BGP-X protection**:
- All node advertisements require valid Ed25519 signatures
- Unsigned or invalid signatures rejected
- Reputation system penalizes anomalous behavior
- Individual node verification required regardless of pool membership
- Geographic plausibility scoring detects inconsistent location claims

**Residual risk**: An adversary who operates a legitimate node can observe traffic passing through it — same as Class C (single malicious relay).

---

### Class N: Gateway Compromise

**Who**: An adversary who controls a mesh island gateway node.

**Capabilities**:
- See destinations of all clearnet traffic from mesh island
- Block or disrupt mesh-to-internet access
- Perform MITM at clearnet exit point
- Log all cross-domain traffic

**Goals**:
- Observe mesh user destinations
- Deny service
- Perform correlation attacks

**BGP-X protection**:
- Gateway sees destination but NOT source (same as any exit node)
- Multiple gateways in different jurisdictions provide redundancy
- Pool selection enables avoiding compromised gateways
- Path rebuild detects unresponsive gateways

**Residual risk**: A compromised gateway knows all destinations for mesh users routed through it. Multiple independent gateways mitigate. Use pool diversity to avoid single gateway for sensitive traffic.

---

### Class O: Mesh Network Partitioning

**Who**: An adversary who physically disrupts mesh connectivity.

**Capabilities**:
- Jam radio frequencies
- Destroy equipment
- Block line-of-sight paths
- Cut power

**Goals**:
- Isolate mesh island
- Deny cross-domain connectivity
- Force traffic through compromised infrastructure

**BGP-X protection**:
- Multiple transport types provide redundancy (LoRa backup for WiFi failures)
- Gateway provides alternative path for critical communications
- BGP-X does NOT guarantee physical connectivity

**Residual risk**: BGP-X provides privacy for whatever connectivity exists. It cannot prevent physical disruption. This is an availability failure, not a privacy failure.

---

### Class P: Cross-Domain Traffic Correlation

**Who**: An adversary who controls a domain bridge node AND observes traffic in both connected domains.

**Capabilities**:
- Observe timing of traffic entering bridge from one domain
- Observe timing of traffic leaving bridge into other domain
- Attempt correlation across domain boundary
- Measure latency between observations

**Goals**:
- Link cross-domain traffic flows
- Identify which mesh traffic corresponds to which clearnet traffic

**BGP-X protection**:
- Bridge node sees only immediate predecessor in each domain, not full path
- path_id used for return routing does not encode path composition
- Cover traffic uses session_key (externally identical) in both domains
- KEEPALIVE jitter (±5 seconds, MANDATORY) in all domains
- Bridge processing latency (radio propagation time) adds noise
- Operator diversity limits single adversary's bridge presence

**Residual risk**: An adversary controlling the ONLY bridge node for a domain pair and observing both sides can attempt timing correlation. Multiple independent bridge nodes from different operators mitigate this.

---

### Class Q: Rogue Mesh Island

**Who**: An adversary who creates a controlled mesh island with signed island advertisement and attractive bridge nodes.

**Capabilities**:
- Operate entire mesh island infrastructure
- Control all nodes in island
- Observe all intra-island traffic
- Correlate traffic entering and leaving island

**Goals**:
- Attract traffic for surveillance
- Perform MITM at island level
- Correlate cross-domain flows

**BGP-X protection**:
- Island advertisements are Ed25519 signed (by island curator or bridge nodes)
- Individual node verification required for every relay selected
- Reputation system penalizes misbehaving nodes
- Pool membership in island pool requires curator signature and individual node verification

**Residual risk**: An adversary operating a real, functioning controlled island can attract traffic. Standard relay adversary model (Class C/D) applies within the island. Users should evaluate island reputation and bridge node diversity.

---

### Class R: Mesh Island Isolation Attack

**Who**: An adversary who disrupts all bridge nodes connecting an island to other domains.

**Capabilities**:
- DDoS bridge node WAN IPs
- BGP hijack bridge node addresses
- Physically disable bridge nodes
- Block bridge node radio links

**Goals**:
- Isolate mesh island
- Deny cross-domain connectivity
- Force users onto alternative infrastructure

**BGP-X protection**:
- Multiple independent bridge nodes per island from different operators and ASNs
- LoRa fallback transport when primary WiFi bridge fails
- Island continues to operate internally when isolated

**Residual risk**: If all bridge nodes are simultaneously disabled, the island loses cross-domain connectivity. This is an availability failure, not a privacy failure. Intra-island traffic remains protected.

---

### Class S: Cross-Domain Global Passive Adversary

**Who**: A global passive adversary with simultaneous observation of all clearnet internet traffic AND all mesh radio transmissions in all islands.

**Capabilities**:
- Observe all internet BGP-X packets
- Observe all mesh radio traffic
- Correlate timing across physical domains
- Identify cross-domain traffic patterns

**Goals**:
- Link cross-domain flows
- Identify participants across all domains
- Comprehensive surveillance

**BGP-X protection**:
- Cover traffic (session_key used; externally identical to RELAY) in all domains
- KEEPALIVE jitter in all domains
- Different fundamental latency profiles between clearnet and mesh make correlation harder
- Multiple cross-domain path options: adversary cannot know which bridge was used without observing all bridges
- Onion encryption prevents content observation regardless of domain

**Residual risk**: With truly global observation capability across multiple physical domains simultaneously, timing correlation is possible. This is acknowledged as a fundamental limitation of all low-latency anonymity networks. BGP-X reduces but cannot eliminate this risk.

---

### Class T: DHT Poisoning

**Who**: An adversary who floods the DHT with fake or malicious records.

**Capabilities**:
- Insert fake node advertisements
- Insert fake pool advertisements
- Insert fake domain bridge records
- Insert fake mesh island advertisements

**Goals**:
- Influence path selection
- Redirect traffic to adversary nodes
- Degrade discovery

**BGP-X protection**:
- All DHT records are Ed25519 signed
- NodeID = BLAKE3(public_key) — fake NodeID requires breaking hash function
- Individual node verification required before path selection
- Storage nodes MUST verify signatures before accepting DHT_PUT

**Residual risk**: An adversary can attempt Sybil attack via many valid-looking advertisements. Reputation system and diversity constraints limit impact. No external dependency for node verification.

---

### Class U: Pluggable Transport Fingerprinting

**Who**: An adversary who identifies BGP-X traffic patterns via DPI even when PT is enabled.

**Capabilities**:
- Deep packet inspection
- Machine learning traffic classification
- Correlation with known BGP-X patterns
- Attempt to bypass PT obfuscation

**Goals**:
- Identify BGP-X participants
- Block BGP-X traffic despite PT

**BGP-X protection**:
- PT obfuscates traffic patterns
- BGP-X's underlying security is unchanged if PT is bypassed
- Adversary who identifies BGP-X traffic still cannot decrypt it or link endpoints

**Residual risk**: A sufficiently sophisticated adversary may identify BGP-X participation through PT. This reveals participation but not content or associations.

---

### Class V: Routing Policy Bypass (LAN Device)

**Who**: A LAN device that sends traffic directly to internet without going through BGP-X overlay.

**Capabilities**:
- Bypass BGP-X routing policy
- Access clearnet directly
- Leak IP address

**Goals**:
- Avoid privacy protection
- Bypass content filtering

**BGP-X protection**:
- BGP-X only firmware: eliminates all bypass paths
- Dual-stack mode: bypass is a feature, not a vulnerability (some traffic intentionally bypasses)
- Per-device rules can enforce overlay for specific devices
- Firewall rules can prevent direct UDP connections to BGP-X node IPs

**Residual risk**: In dual-stack mode, misconfigured devices or applications can bypass the overlay. Use BGP-X only firmware for maximum enforcement.

---

### Class W: SDK Socket Exposure

**Who**: An unauthenticated LAN device that accesses the SDK socket TCP endpoint.

**Capabilities**:
- Open streams
- Register services
- Influence path selection via API

**Goals**:
- Use BGP-X infrastructure
- Open streams to arbitrary destinations
- Enumerate services

**BGP-X protection**:
- Default: Unix socket (local access only, file permission 0600)
- TCP endpoint: requires authentication token if enabled
- Recommendation: never expose control or SDK sockets to WAN

**Residual risk**: An attacker with LAN access and SDK socket access can open streams through the overlay. This is a LAN-level trust boundary issue.

---

### Class X: Satellite Link Observation

**Who**: An adversary who observes traffic on a satellite link (e.g., can capture satellite downlink).

**Capabilities**:
- Observe encrypted BGP-X packets over satellite
- Attempt correlation with other observations

**Goals**:
- Confirm BGP-X participation via satellite
- Correlate timing

**BGP-X protection**:
- Same ChaCha20-Poly1305 encryption as all other transports
- Satellite is clearnet domain (0x00000001) — same protection as any clearnet hop
- Cover traffic and jitter apply

**Residual risk**: An adversary who can observe satellite traffic can confirm BGP-X participation. Correlation with other observations is possible. Satellite users should consider their adversary model carefully.

---

## 4. Protocol-Level Attack Vectors

### 4.1 Replay Attack

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

### 4.2 Packet Injection

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

### 4.3 Session ID Guessing

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

### 4.4 Handshake MITM

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

### 4.5 Version Downgrade Attack

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

### 4.6 Path Manipulation by Malicious Node

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

### 4.7 path_id Guessing

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

## 5. Cross-Domain Specific Attack Vectors

### 5.1 Domain Bridge MITM

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

### 5.2 Fake Mesh Island

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

### 5.3 Bridge Node Sybil Attack

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

### 5.4 Domain Isolation Attack (Availability)

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

### 5.5 Cross-Domain Timing Correlation

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

### 5.6 Rogue DOMAIN_ADVERTISE Record

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

### 5.7 Mesh Island DHT Poisoning

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

### 5.8 Satellite Timing Correlation

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

## 6. Node Discovery Attack Vectors

### 6.1 DHT Poisoning

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

### 6.2 Malicious Bootstrap Node

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

### 6.3 Eclipse Attack

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

## 7. Traffic Analysis Attack Vectors

### 7.1 Timing Correlation

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

### 7.2 Volume Correlation

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

### 7.3 Website Fingerprinting

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

## 8. Pool-Specific Attack Vectors

### 8.1 Pool Poisoning

**Description**: An adversary gains curator identity control; inserts malicious nodes.

**Impact**: Malicious nodes appear in trusted pool.

**Mitigation**:
- Individual node verification: adversary cannot forge individual node Ed25519 signatures
- Reputation system: inserted malicious nodes start at 60.0 and must be caught misbehaving
- Cross-pool diversity: even with poisoned pool in one segment, other segments use different pool
- Pool advertisements signed by curator key; key rotation mechanism available

**Residual risk**: Compromised curator can insert their real (operational) nodes until detected.

---

### 8.2 Pool Sybil

**Description**: An adversary creates many real nodes and convinces curator to include them.

**Impact**: Adversary gains proportional presence in pool.

**Mitigation**:
- Each node starts at 60.0 reputation and must earn higher scores
- Weighted random selection: Sybil nodes don't dominate
- Cross-pool operator diversity limits per-operator presence in a path
- Pool size thresholds: pools below minimum members trigger warning

---

### 8.3 Pool Curator Key Compromise

**Description**: An adversary steals pool curator's private key.

**Impact**: Adversary can sign new pool advertisements with malicious members.

**Mitigation**:
- Key rotation mechanism with dual-signature rotation records
- Emergency rotation with 2-hour acceptance window for compromise_confirmed
- Individual node signatures (curator can add malicious nodes but cannot forge individual signatures)
- Clients detect anomalous pool-level behavior and avoid pool
- 24-hour grace period for member re-verification after rotation

---

### 8.4 Pool Curator Coercion

**Description**: Legal coercion of pool curator to include malicious nodes.

**Impact**: Coerced curator adds adversary nodes to pool.

**Mitigation**:
- Individual node verification still applies (coerced curator cannot bypass)
- Curator can publish pool with honest nodes; coerced curator included "honest-looking" adversary nodes
- Cross-pool diversity limits impact to one segment
- Reputation system catches misbehaving nodes over time
- Pool membership does NOT require jurisdiction declaration

---

## 9. Denial of Service Attack Vectors

### 9.1 Session Flooding

**Description**: An attacker sends many HANDSHAKE_INIT messages to exhaust the target node's session table.

**Impact**: Legitimate clients cannot establish sessions.

**Mitigation**:
- HANDSHAKE_INIT requires decryption with the node's public key — computationally costly for attacker
- Session table has configurable maximum (default 10,000)
- Rate limiting on HANDSHAKE_INIT per source address
- Silent rejection when full (no amplifiable error response)

**Status**: Partially mitigated. High-volume DDoS from many source IPs remains possible.

---

### 9.2 Packet Amplification

**Description**: A small request triggers a large response.

**Impact**: DDoS amplification.

**Analysis**: BGP-X responses approximately match request size. DHT responses bounded. No significant amplification factor.

**Status**: Mitigated.

---

### 9.3 Gateway Overload

**Description**: Opens many streams through single exit gateway.

**Impact**: Exit node becomes unavailable.

**Mitigation**:
- Per-session connection limits
- Total outbound connection cap
- Gateway reputation degrades if overloaded; selected less frequently
- Multiple exit gateways in pool

---

### 9.4 Cross-Domain Path Table Overflow

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

## 10. Implementation-Level Attack Vectors

### 10.1 Buffer Overflow in Packet Parsing

**Description**: Maliciously crafted packet triggers memory safety vulnerability in packet parsing.

**Impact**: Potential remote code execution on the relay node.

**Mitigation**:
- Reference implementation is written in Rust (memory-safe by default)
- No unsafe Rust in the packet parsing path (enforced by code review policy)
- Packet length fields are bounds-checked before any access
- Fuzzing of the packet parsing pipeline using cargo-fuzz

**Status**: Strongly mitigated by Rust memory safety.

---

### 10.2 Timing Side-Channel in Cryptographic Operations

**Description**: An attacker measures the time taken to process packets and extracts key material from timing differences.

**Impact**: Potential session key recovery.

**Mitigation**:
- ChaCha20-Poly1305 implementation uses constant-time operations
- Session ID lookup uses constant-time comparison
- Authentication tag comparison is constant-time (no early exit on mismatch)
- Cryptographic library selection criteria require constant-time implementations

**Status**: Mitigated (dependent on correctness of cryptographic library implementations).

---

### 10.3 Key Material in Swap Space

**Description**: The OS writes session keys to swap space, allowing an adversary with physical access to the swap partition to recover keys.

**Impact**: Recovery of past session keys.

**Mitigation**:
- Session keys are stored in `mlock`-protected memory on Linux
- `mlock` prevents the OS from swapping the memory page to disk
- On process exit, zeroize is called on all key material before freeing
- Guard pages around key memory

**Status**: Mitigated on Linux. Platform support for memory locking varies.

---

## 11. Implementation-Specific Attack Vectors

### 11.1 PT Bypass

**Description**: Adversary identifies BGP-X traffic despite pluggable transport.

**Impact**: ISP can block or identify BGP-X participation.

**Mitigation**: obfs4-style obfuscation designed for DPI resistance; Elligator2 key encoding; random-length padding; timing jitter.

**Residual risk**: Sufficiently sophisticated DPI with ML may detect patterns.

---

### 11.2 ECH Oracle at Exit Node

**Description**: Timing or error-based oracle attack on ECH implementation at exit node reveals domain name.

**Impact**: Destination domain revealed despite ECH.

**Mitigation**: ECH from audited TLS library. No timing branches on decryption success/failure. Error responses standardized (no domain-specific error detail). Silent retry without ECH before stream failure.

**Residual risk**: Timing difference between ECH and standard TLS handshakes may be measurable at network level.

---

### 11.3 Routing Policy Bypass (LAN)

**Description**: LAN device sends traffic directly to internet, bypassing overlay.

**Impact**: Traffic bypasses privacy protections.

**Mitigation**: BGP-X only mode eliminates all bypass; per-device rules enforce overlay; kill switch blocks non-BGP-X traffic; firewall rules can prevent direct UDP on 7474.

---

### 11.4 Radio Interception (Mesh Domain)

**Description**: Adversary captures mesh radio transmissions.

**Impact**: Adversary obtains encrypted traffic.

**Mitigation**: All BGP-X packets ChaCha20-Poly1305 encrypted with session keys. Radio interceptor sees ciphertext only. Traffic analysis mitigated by cover traffic and KEEPALIVE jitter.

**Status**: Content protected. Timing analysis partially mitigated.

---

### 11.5 Rogue Mesh Beacon

**Description**: Adversary broadcasts fake MESH_BEACON with fake NodeID to poison DHT routing table.

**Impact**: DHT routing table contaminated with adversary nodes.

**Mitigation**: MESH_BEACON is Ed25519 signed. Signature verification mandatory before DHT update. NodeID = BLAKE3(public_key) — forge attempt fails without private key.

**Status**: Mitigated.

---

### 11.6 Gateway Censorship

**Description**: Gateway operator refuses to forward certain traffic.

**Impact**: Certain destinations blocked.

**Mitigation**: Multiple gateways in different jurisdictions; pool-based gateway selection allows avoiding censoring gateways; exit policy versioning enables detecting policy changes; reputation system penalizes unresponsive gateways.

---

### 11.7 Domain Bridge Timing Oracle

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

## 12. Social and Operational Attack Vectors

### 12.1 Malicious Operator

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

### 12.2 Legal Coercion of Node Operators

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

### 12.3 Pool Curator Coercion

**Description**: Legal coercion of pool curator to include malicious nodes or reveal pool membership information.

**Impact**: Pool becomes untrustworthy.

**Mitigation**:
- Individual node verification still applies
- Pool membership is public (curator cannot reveal what is already public)
- Cross-pool diversity limits impact
- Curator key rotation enables replacing compromised curator

---

## 13. Protection Matrix

| Asset | A: ISP | C: Relay | E: Entry+Exit | F: Global Passive | P: Cross-Domain | S: Multi-Domain Global | L: Radio | N: Gateway |
|---|---|---|---|---|---|---|---|---|
| Source identity | ✅ | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ✅ | ✅ |
| Destination | ✅ | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ✅ | ⚠️ |
| Content | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Metadata | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Participation | ⚠️ visible | N/A | N/A | ⚠️ visible | N/A | ❌ visible | ⚠️ visible | N/A |
| Domain name | ✅ (ECH) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ (exit sees) |
| Path composition | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cross-domain traversal | N/A | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ | ✅ |

✅ = Protected. ⚠️ = Residual risk mitigated. ❌ = Not protected.

---

## 14. Sybil Attack Resistance

### 14.1 Standard Sybil Mitigation

A Sybil attacker creates many nodes to influence network behavior.

**BGP-X mitigations**:
1. ASN/country/operator diversity constraints limit path presence of any single operator
2. Reputation system: new nodes start at 60.0; must earn higher scores slowly
3. Weighted random selection: Sybil nodes compete probabilistically
4. Pool curator verification: curated pools provide additional filter

### 14.2 Sybil via Pool

Adversary convinces curator to include malicious nodes.

**Mitigation**:
- Individual node verification catches poor-reputation nodes
- Reputation system degrades behavior over time
- Cross-pool diversity limits path impact

### 14.3 Cross-Domain Sybil Resistance

Cross-domain diversity enforcement limits Sybil attacks across domain boundaries:
- Sybil nodes in clearnet cannot also dominate mesh segments (cross-domain operator diversity enforced)
- A Sybil operator controlling many bridge nodes is limited to one bridge position per path
- Domain-bridge pool curator verification adds independent verification layer

**Residual risk**: A very large Sybil operation (thousands of nodes with legitimate uptime) could influence path selection significantly. This is an area of ongoing research in anonymity network design.

---

## 15. Timing Analysis

Timing correlation is the primary unsolved challenge for all low-latency anonymous networks.

### 15.1 Basic Timing Attack

An adversary observing traffic entering the overlay and traffic exiting the overlay correlates them if packets flow at the same rate and timing pattern.

### 15.2 BGP-X Mitigations

1. **Multi-hop routing**: Each hop adds variable queuing delay, disrupting simple correlation
2. **Variable path length**: More hops = more timing disruption per hop
3. **Geographic diversity**: Physical distance adds unpredictable delay
4. **Stream multiplexing**: Multiple streams create ambiguity — a packet exiting at a given time could belong to any stream
5. **Cover traffic**: Synthetic traffic masks timing signature of real traffic
6. **KEEPALIVE randomization**: ±5 seconds MANDATORY — prevents keepalive timing fingerprinting
7. **Pool-based multi-segment paths**: Multiple trust boundaries add timing uncertainty
8. **Cross-domain latency variance**: Clearnet vs. mesh have fundamentally different latency profiles

### 15.3 Cross-Domain Timing

Cross-domain paths traverse fundamentally different physical layers:
- Clearnet: fiber/cellular, 10-200ms RTT typical
- WiFi mesh: 1-50ms per hop
- LoRa mesh: 500ms-5s per hop

This physical diversity makes correlation across domain boundaries more difficult than single-domain correlation.

### 15.4 Residual Risk

Sufficiently sophisticated timing analysis (Murdoch-Danezis style, or newer deep learning approaches) may still correlate traffic through low-latency overlay networks. This is a known open problem in anonymous communications research. BGP-X reduces but does not eliminate this risk.

---

## 16. Pool-Specific Threats

### 16.1 Pool Capture

Adversary gradually introduces malicious pool members.

**Mitigation**:
- Individual node verification slows this
- Reputation system penalizes poor behavior
- Cross-pool diversity limits impact

### 16.2 Pool Curator Compromise

Adversary gains curator private key.

**Mitigation**:
- Can sign fake advertisements
- Individual node verification provides fallback
- Key rotation with 2-hour emergency window
- 24-hour grace period for old-key-signed members

### 16.3 Pool Flood (Sybil via Pool)

Adversary operates many real nodes and convinces curator to include them.

**Mitigation**:
- Reputation starts at 60.0
- Weighted random selection limits dominance
- Cross-pool diversity distributes risk

---

## 17. Hardware-Specific Threats

### 17.1 BGP-X Router v1 / Node v1

| Threat | Mitigation |
|---|---|
| Physical theft | Encrypted eMMC; TPM-bound keys |
| USB port abuse | Disable unused ports in firmware |
| Firmware tampering | Secure boot; signed updates |
| Radio interception | Standard ChaCha20-Poly1305 encryption |
| Power analysis | Constant-time crypto; hardware RNG |

### 17.2 BGP-X Gateway v1

| Threat | Mitigation |
|---|---|
| Physical access (datacenter) | Encrypted storage; TPM |
| Insider threat | Multi-administrator; audit logs |
| High-value target | Enhanced monitoring; dedicated security |

### 17.3 BGP-X Client Node (Tier 2)

| Threat | Mitigation |
|---|---|
| Device theft | Keys in secure element (ATECC608) if present |
| Malware on host | Client firmware does not trust host; verify signatures |
| Radio interception | Standard encryption |

### 17.4 BGP-X Adapter (Tier 3)

| Threat | Mitigation |
|---|---|
| Device theft | No key storage on device |
| Firmware tampering | Signed firmware |
| USB MITM | No sensitive operations on dongle |

---

## 18. Geographic Plausibility — OPTIONAL Feature

Geographic plausibility scoring is an OPTIONAL reputation signal. It is NOT required for BGP-X operation.

### 18.1 When Geo Plausibility Applies

- IF a node declares a jurisdiction in its advertisement: geo plausibility scoring applies
- IF a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

Nodes are NOT required to declare a jurisdiction. Declaring jurisdiction is an opt-in privacy/convenience tradeoff.

### 18.2 Exemptions

The following node types are EXEMPT from geo plausibility scoring (always return neutral score 0.5):
- Satellite-class clearnet nodes (`latency_class = satellite-*`)
- Mesh nodes without declared jurisdiction
- Nodes in domains without internet RTT calibration (pure mesh islands without clearnet bridge)

### 18.3 Behavior in Path Construction

A node with poor geo plausibility score:
- Receives lower selection probability (scoring penalty)
- Can still be selected in a path (not hard excluded)
- Persistent implausibility may lead to reputation penalties

Geo plausibility is a reputation signal, not a mandatory filtering criterion.

---

## 19. Trust Assumptions

1. **Cryptographic primitives are secure**: X25519, Ed25519, ChaCha20-Poly1305, BLAKE3, HKDF-SHA256 are computationally infeasible to break with current non-quantum hardware.

2. **CSPRNG is secure**: Operating system random number generator produces unpredictable values.

3. **Client device is trusted**: Device, OS, and BGP-X software are not compromised.

4. **Diversity constraints are satisfied**: Node advertisements accurately report ASN, country, operator identity.

5. **Pool curators are honest**: Individual node verification is the fallback if not.

6. **BGP-X software is unmodified**: Binary transparency (planned extension).

7. **Domain bridge advertisements are accurate**: Misbehavior degrades reputation.

8. **Bridge node operators for a given domain pair are not all colluding**: Enforced by diversity constraints.

---

## 20. What BGP-X Is NOT Designed to Defeat

- **Compromised client device**: If the device is compromised, all bets are off
- **Application fingerprinting**: Browser JavaScript, canvas, fonts, etc.
- **Long-term behavioral deanonymization**: Consistent access patterns may identify users
- **Legal coercion of the user themselves**: Users can be compelled to reveal identity
- **Quantum computer attacks**: X25519/Ed25519 are not post-quantum secure (PQ planned for future version)
- **Physical coercion of node operators**: Operators can be compelled to log or modify behavior
- **Physical disruption of infrastructure**: BGP-X provides privacy, not physical protection
- **Cross-domain timing correlation by adversaries monitoring multiple domains simultaneously**: Known residual risk, mitigated by cover traffic and latency variance
- **Complete unavailability of bridge nodes for a required domain pair**: Availability failure, not privacy failure

---

## 21. Acceptance Criteria for Attack Simulations

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
