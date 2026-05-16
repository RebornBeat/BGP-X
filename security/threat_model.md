# BGP-X Threat Model

**Version**: 0.1.0-draft

---

## 1. Design Philosophy

BGP-X is an engineering system, not a magic shield. Its privacy properties are bounded by cryptographic assumptions, network conditions, application behavior, and configuration choices. This document states precisely what BGP-X protects, what it does not protect, and under what conditions protection degrades.

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

## 4. Protection Matrix

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

## 5. Sybil Attack Resistance

### Standard Sybil Mitigation

A Sybil attacker creates many nodes to influence network behavior.

**BGP-X mitigations**:
1. ASN/country/operator diversity constraints limit path presence of any single operator
2. Reputation system: new nodes start at 60.0; must earn higher scores slowly
3. Weighted random selection: Sybil nodes compete probabilistically
4. Pool curator verification: curated pools provide additional filter

### Sybil via Pool

Adversary convinces curator to include malicious nodes.

**Mitigation**:
- Individual node verification catches poor-reputation nodes
- Reputation system degrades behavior over time
- Cross-pool diversity limits path impact

### Cross-Domain Sybil Resistance

Cross-domain diversity enforcement limits Sybil attacks across domain boundaries:
- Sybil nodes in clearnet cannot also dominate mesh segments (cross-domain operator diversity enforced)
- A Sybil operator controlling many bridge nodes is limited to one bridge position per path
- Domain-bridge pool curator verification adds independent verification layer

**Residual risk**: A very large Sybil operation (thousands of nodes with legitimate uptime) could influence path selection significantly. This is an area of ongoing research in anonymity network design.

---

## 6. Timing Analysis

Timing correlation is the primary unsolved challenge for all low-latency anonymous networks.

### Basic Timing Attack

An adversary observing traffic entering the overlay and traffic exiting the overlay correlates them if packets flow at the same rate and timing pattern.

### BGP-X Mitigations

1. **Multi-hop routing**: Each hop adds variable queuing delay, disrupting simple correlation
2. **Variable path length**: More hops = more timing disruption per hop
3. **Geographic diversity**: Physical distance adds unpredictable delay
4. **Stream multiplexing**: Multiple streams create ambiguity — a packet exiting at a given time could belong to any stream
5. **Cover traffic**: Synthetic traffic masks timing signature of real traffic
6. **KEEPALIVE randomization**: ±5 seconds MANDATORY — prevents keepalive timing fingerprinting
7. **Pool-based multi-segment paths**: Multiple trust boundaries add timing uncertainty
8. **Cross-domain latency variance**: Clearnet vs. mesh have fundamentally different latency profiles

### Cross-Domain Timing

Cross-domain paths traverse fundamentally different physical layers:
- Clearnet: fiber/cellular, 10-200ms RTT typical
- WiFi mesh: 1-50ms per hop
- LoRa mesh: 500ms-5s per hop

This physical diversity makes correlation across domain boundaries more difficult than single-domain correlation.

### Residual Risk

Sufficiently sophisticated timing analysis (Murdoch-Danezis style, or newer deep learning approaches) may still correlate traffic through low-latency overlay networks. This is a known open problem in anonymous communications research. BGP-X reduces but does not eliminate this risk.

---

## 7. Pool-Specific Threats

### Pool Capture

Adversary gradually introduces malicious pool members.

**Mitigation**:
- Individual node verification slows this
- Reputation system penalizes poor behavior
- Cross-pool diversity limits impact

### Pool Curator Compromise

Adversary gains curator private key.

**Mitigation**:
- Can sign fake advertisements
- Individual node verification provides fallback
- Key rotation with 2-hour emergency window
- 24-hour grace period for old-key-signed members

### Pool Flood (Sybil via Pool)

Adversary operates many real nodes and convinces curator to include them.

**Mitigation**:
- Reputation starts at 60.0
- Weighted random selection limits dominance
- Cross-pool diversity distributes risk

---

## 8. Hardware-Specific Threats

### BGP-X Router v1 / Node v1

| Threat | Mitigation |
|---|---|
| Physical theft | Encrypted eMMC; TPM-bound keys |
| USB port abuse | Disable unused ports in firmware |
| Firmware tampering | Secure boot; signed updates |
| Radio interception | Standard ChaCha20-Poly1305 encryption |
| Power analysis | Constant-time crypto; hardware RNG |

### BGP-X Gateway v1

| Threat | Mitigation |
|---|---|
| Physical access (datacenter) | Encrypted storage; TPM |
| Insider threat | Multi-administrator; audit logs |
| High-value target | Enhanced monitoring; dedicated security |

### BGP-X Client Node (Tier 2)

| Threat | Mitigation |
|---|---|
| Device theft | Keys in secure element (ATECC608) if present |
| Malware on host | Client firmware does not trust host; verify signatures |
| Radio interception | Standard encryption |

### BGP-X Adapter (Tier 3)

| Threat | Mitigation |
|---|---|
| Device theft | No key storage on device |
| Firmware tampering | Signed firmware |
| USB MITM | No sensitive operations on dongle |

---

## 9. Geographic Plausibility — OPTIONAL Feature

Geographic plausibility scoring is an OPTIONAL reputation signal. It is NOT required for BGP-X operation.

### When Geo Plausibility Applies

- IF a node declares a jurisdiction in its advertisement: geo plausibility scoring applies
- IF a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

Nodes are NOT required to declare a jurisdiction. Declaring jurisdiction is an opt-in privacy/convenience tradeoff.

### Exemptions

The following node types are EXEMPT from geo plausibility scoring (always return neutral score 0.5):
- Satellite-class clearnet nodes (`latency_class = satellite-*`)
- Mesh nodes without declared jurisdiction
- Nodes in domains without internet RTT calibration (pure mesh islands without clearnet bridge)

### Behavior in Path Construction

A node with poor geo plausibility score:
- Receives lower selection probability (scoring penalty)
- Can still be selected in a path (not hard excluded)
- Persistent implausibility may lead to reputation penalties

Geo plausibility is a reputation signal, not a mandatory filtering criterion.

---

## 10. Trust Assumptions

1. **Cryptographic primitives are secure**: X25519, Ed25519, ChaCha20-Poly1305, BLAKE3, HKDF-SHA256 are computationally infeasible to break with current non-quantum hardware.

2. **CSPRNG is secure**: Operating system random number generator produces unpredictable values.

3. **Client device is trusted**: Device, OS, and BGP-X software are not compromised.

4. **Diversity constraints are satisfied**: Node advertisements accurately report ASN, country, operator identity.

5. **Pool curators are honest**: Individual node verification is the fallback if not.

6. **BGP-X software is unmodified**: Binary transparency (planned extension).

7. **Domain bridge advertisements are accurate**: Misbehavior degrades reputation.

8. **Bridge node operators for a given domain pair are not all colluding**: Enforced by diversity constraints.

---

## 11. What BGP-X Is NOT Designed to Defeat

- **Compromised client device**: If the device is compromised, all bets are off
- **Application fingerprinting**: Browser JavaScript, canvas, fonts, etc.
- **Long-term behavioral deanonymization**: Consistent access patterns may identify users
- **Legal coercion of the user themselves**: Users can be compelled to reveal identity
- **Quantum computer attacks**: X25519/Ed25519 are not post-quantum secure (PQ planned for future version)
- **Physical coercion of node operators**: Operators can be compelled to log or modify behavior
- **Physical disruption of infrastructure**: BGP-X provides privacy, not physical protection
- **Cross-domain timing correlation by adversaries monitoring multiple domains simultaneously**: Known residual risk, mitigated by cover traffic and latency variance
- **Complete unavailability of bridge nodes for a required domain pair**: Availability failure, not privacy failure
