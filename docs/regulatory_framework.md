# BGP-X Regulatory Framework

This document covers the regulatory considerations for BGP-X deployments, particularly for radio-based mesh networks, elevated relay deployment, and operator legal considerations.

**Jurisdiction**: BGP-X is a worldwide project. This document provides general regulatory context, but users and operators are responsible for understanding and complying with the specific laws of their jurisdictions. BGP-X does not impose jurisdictional restrictions on the network; it provides tools for users to comply with their local requirements.

---

## 1. Radio Spectrum Regulations

### 1.1 LoRa / ISM Band (Primary Mesh Transport)

LoRa operates in the Industrial, Scientific, and Medical (ISM) bands. These are license-free in most jurisdictions, but subject to power and duty cycle limits.

| Region | Frequency | Max Power | Duty Cycle | License |
|---|---|---|---|---|
| European Union | 868 MHz | 25 mW ERP | 1% (some sub-bands) | License-free |
| United States | 915 MHz | 30 dBm (1W) | No limit (but FHSS rules apply) | License-free (Part 15) |
| Canada | 915 MHz | 30 dBm | Part 15 rules | License-free |
| Australia | 915 MHz | 30 dBm | No limit | License-free |
| Japan | 920 MHz | 20 mW | Regulations apply | License-free |
| General (433 MHz) | 433.05-434.79 MHz | 10 mW | 10% | License-free in most regions |

**BGP-X compliance**: BGP-X LoRa nodes (BGP-X Router v1, Node v1, Client Node) operate within these limits by default. Configuration allows adjusting power and duty cycle per regional requirements.

**EU duty cycle**: 1% duty cycle limits LoRa nodes to ~36 seconds of transmit time per hour. BGP-X is designed for this constraint:

- **DHT sync and control messages**: Primary LoRa use case; fits within duty cycle
- **Latency-tolerant applications**: LoRa CAN support interactive applications (browsing, messaging, API calls) when designed for high latency (1-5 seconds per round-trip)
- **Duty cycle management**: BGP-X firmware enforces duty cycle via token bucket; streams may enter PAUSED state when budget exhausted
- **Not suitable for**: High-throughput applications (video streaming, large file transfer) due to inherent bandwidth limits, not just duty cycle

BGP-X Browser and BGP-X-native applications implement latency-tolerant design patterns (batched resource loading, progressive rendering, aggressive caching) specifically to function well over LoRa paths within regulatory constraints.

### 1.2 WiFi 802.11s (Primary Mesh Transport for Dense Deployment)

WiFi operates in the 2.4 GHz and 5 GHz ISM/U-NII bands.

| Frequency | Jurisdiction | License | Notes |
|---|---|---|---|
| 2.4 GHz (802.11b/g/n/ax) | Global | License-free (limits vary) | Universal availability |
| 5 GHz (802.11a/n/ac/ax) | Most regions | License-free with restrictions | Some channels require DFS |
| 6 GHz (802.11ax WiFi 6E) | US, EU, others | License-free (low power indoor) | Newer, faster |

**BGP-X compliance**: 802.11s mesh operates on standard WiFi channels. No special licensing required.

### 1.3 Bluetooth BLE

Bluetooth 5.0 mesh operates at 2.4 GHz, globally license-free, 100mW maximum power. No regulatory concerns for BGP-X Bluetooth mesh use.

---

## 2. Fixed Antenna Installation (Recommended for Elevated Deployment)

For permanent elevated relay installation (masts, rooftops, towers):

### 2.1 Typical Regulatory Path

Fixed antenna installations are regulated by planning/building codes, not aviation authorities. This is the simplest path for permanent elevated coverage.

| Country | Regulatory Body | Process |
|---|---|---|
| USA | Local zoning + FCC (for licensed spectrum, not needed for ISM) | Building permit for mast if above threshold height |
| EU | National planning authorities | Varies by country; many exempt small antennas |
| UK | Permitted Development Rights | Small antennas often exempt from planning permission |

**BGP-X recommendation**: Fixed masts at rooftop or utility pole height (10-30m) are the most practical permanent elevated relay deployment. Regulatory burden is minimal for ISM-band antennas at these heights.

The **BGP-X Node v1 outdoor enclosure (IP67)** and **BGP-X Router v1 outdoor carrier** are designed for this deployment profile.

---

## 3. Tethered Aerostat Regulations

Tethered balloons and aerostats have separate, generally more permissive regulations than free-flying drones.

| Country | Regulatory Body | Key Rules |
|---|---|---|
| USA | FAA (14 CFR Part 101) | Under 300 ft AGL: notify FSDO, no aviation authority required. Under 150 ft with <4 lb free lift: essentially unregulated. |
| European Union | EASA + national authorities | Tethered aerostats typically under national (not EASA UAS) rules; generally notify for >50m |
| UK | CAA | Tethered balloons have separate rules from UAS; generally lighter regulation at low altitudes |
| Canada | Transport Canada | Tethered kites/balloons: separate section of CARs |

**BGP-X recommendation**: A tethered aerostat at 60-150m AGL, notifying the relevant aviation authority as required, provides approximately 30-45km LoRa coverage radius. This is appropriate for temporary wide-area coverage during events, disaster response, or community trials.

For permanent deployment, fixed masts are more practical (no helium maintenance, no weather constraints).

---

## 4. Drone / UAV Regulations

Drones for BGP-X relay deployment face stricter regulation than tethered aerostats.

### 4.1 Altitude Limits

| Country | Standard Limit | Notes |
|---|---|---|
| USA (FAA Part 107) | 400 ft (122m) AGL | Waivers available up to ~800 ft for justified operations |
| European Union (EASA) | 120m AGL | 150m with authorization for justified operations |
| UK (CAA) | 120m AGL | |
| Canada (Transport Canada) | 122m AGL | |
| Australia (CASA) | 120m AGL | |

### 4.2 Over Private Property

In the USA, the FAA regulates airspace — not specifically who you fly over on private property. State laws vary. In the EU and UK, regulations are more restrictive near residential areas.

### 4.3 BGP-X Drone Relay Assessment

**Honest assessment**: Drones are not practical for persistent relay infrastructure.

- Flight time: 30-60 minutes requires continuous battery swaps
- Regulatory complexity increases with persistent operation
- Mechanical failure risk for infrastructure
- Weather sensitivity

**Appropriate drone use**:
- Emergency deployment (disaster, rescue communications)
- Temporary event coverage
- Site survey for optimal fixed node placement

**Not appropriate**:
- Permanent mesh coverage (use fixed masts or aerostats)
- Unattended continuous operation

---

## 5. Maritime Operations

BGP-X maritime nodes operate in unlicensed ISM bands (LoRa, WiFi). No special maritime radio license is required for ISM-band operation.

BGP-X does NOT use licensed maritime frequencies (VHF, HF marine bands). Those require operator licenses.

**In international waters**: Generally more flexible ISM regulations. Check flag state requirements.

**In coastal areas**: Follow local national regulations.

**BGP-X hardware**: The **BGP-X Router v1 Maritime enclosure (IP68)** is designed for vessel mounting, saltwater corrosion resistance, and integration with marine power systems.

---

## 6. Satellite Terminal Regulations

### 6.1 Starlink

Starlink requires a user agreement and service registration with SpaceX. Available in most countries. Some countries restrict or prohibit satellite internet terminal operation.

**Regulatory check**: Confirm Starlink is licensed in your jurisdiction before deploying a BGP-X satellite gateway.

### 6.2 Iridium, Inmarsat

Licensed service; operator provides regulatory compliance within coverage area.

### 6.3 Satellite as Clearnet in BGP-X Architecture

**Important clarification**: Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) provide **clearnet** connectivity over BGP-routed infrastructure. From BGP-X's perspective:

- **Domain**: Clearnet (type 0x00000001) — same as fiber, cellular, or cable
- **Latency class**: Satellite-specific (LEO: 20-60ms, GEO: 600ms+)
- **Regulatory treatment**: Same as any other internet WAN from a BGP-X protocol perspective. The physical medium (satellite radio vs. fiber) is invisible to BGP-X routing.

**Domain type 0x00000005 (bgpx-satellite)** is RESERVED for a future BGP-X-native satellite network where satellites themselves run BGP-X relay software and communicate via inter-satellite links. This is NOT currently active. Any node advertising domain type 0x00000005 in current deployments MUST be rejected as unverifiable.

---

## 7. Cryptographic Software Export

BGP-X implements strong cryptography (AES-256 equivalent security, ChaCha20-Poly1305). Export regulations apply in some jurisdictions.

### 7.1 United States (EAR)

Encryption software generally falls under ECCN 5D002. Mass-market encryption software is generally eligible for License Exception ENC after a one-time review. Open source cryptography generally has more permissive rules.

**BGP-X**: as open source software, check current EAR rules. BGP-X contributors should verify compliance for their jurisdiction before contributing to or distributing the software.

### 7.2 European Union

EU generally does not restrict the export of publicly available cryptographic software.

**Recommendation**: consult a lawyer familiar with export control in your jurisdiction if you have concerns about deploying BGP-X in a specific country or exporting the software.

---

## 8. Privacy and Data Regulations

BGP-X nodes — particularly exit nodes and mesh island gateways — may be subject to data retention regulations in their jurisdiction.

**BGP-X design**: the default logging policy is "none" — no traffic data is logged. Exit nodes and gateways cannot produce records of connections they never stored.

**Operator responsibility**: verify the legal requirements for node operation in your jurisdiction. See `/gateway/exit_node.md` for legal considerations for exit node operators.

**Data retention laws**: some jurisdictions (EU member states under national implementation of electronic communications law, others) require ISPs and network operators to retain traffic metadata. Whether BGP-X node operators qualify as "ISPs" or "network operators" for these purposes varies by jurisdiction and legal interpretation. Consult a lawyer.

### 8.1 Mesh Island Users and Data Retention

Mesh island users whose traffic exits via a gateway node are subject to the gateway operator's logging policy (which should be "none"). However, mesh-only traffic (between nodes within the island) never traverses the internet and is not subject to internet data retention requirements.

**Key distinction**:
- **Intra-island traffic**: LoRa/WiFi mesh transport; never touches internet; not subject to ISP-style data retention
- **Cross-domain traffic via gateway**: Traffic exiting to internet through gateway; gateway operator's jurisdiction and logging policy apply

Mesh island users should understand:
- Their mesh island traffic is private within the island (onion encryption end-to-end)
- If they use the island's gateway to access the internet, the gateway is their "ISP" from a regulatory perspective
- The gateway's logging policy determines what records exist

---

## 9. Mesh Island Gateway Operations

A mesh island gateway node bridges traffic between a local mesh network and the internet. Operators of mesh island gateways should understand their potential legal status.

### 9.1 When a Gateway Operator May Be Considered an ISP

In some jurisdictions, providing internet access to others (even freely) may classify the operator as an ISP or telecommunications service provider. This depends on:

- **Scale**: Number of users served
- **Commercial vs. non-commercial**: For-profit operation more likely regulated
- **Jurisdiction**: Varies significantly by country

**BGP-X design**: Gateway nodes log NO traffic data by default. If an operator is legally required to produce traffic records, they truthfully cannot — the records do not exist.

### 9.2 Recommendations for Gateway Operators

1. **Understand local law**: Consult a lawyer in your jurisdiction
2. **Document non-commercial nature** (if applicable): Community mesh gateways are typically non-profit
3. **Transparency**: Publish your node's capabilities and logging policy (which is "none")
4. **Exit policy**: If providing clearnet exit, publish your exit policy

### 9.3 EU-Specific Considerations

EU member states implement the Electronic Communications Code differently. Some require notification or registration for even small-scale network operators. Check your national implementation.

### 9.4 US-Specific Considerations

In the United States, operating as a common carrier requires FCC registration. Most small community mesh gateways do not qualify as common carriers, but providing internet access to others could trigger obligations in some circumstances. The FCC's jurisdiction is over interstate communication — purely local mesh operations may fall under different frameworks.

---

## 10. Domain Bridge Operator Considerations

A domain bridge node connects two routing domains (e.g., clearnet ↔ LoRa mesh). From a regulatory perspective, the bridge node is the **only internet-visible endpoint** for the entire mesh island.

### 10.1 Key Considerations

1. **Exit-like role**: A bridge that provides clearnet exit for mesh island users functions similarly to an exit node. See `/gateway/exit_node.md` for legal considerations.

2. **Mesh-only bridge**: A bridge between two mesh islands (no clearnet) has fewer regulatory implications — no public internet traffic is involved.

3. **Liability**: The bridge operator cannot see the content of traffic (onion encryption) or the identities of users within the mesh island. This provides legal protection but does not eliminate all risk in some jurisdictions.

4. **Jurisdictional diversity**: If operating a bridge for a mesh island in one jurisdiction while physically located in another, both jurisdictions' laws may apply.

### 10.2 Transparency Recommendations

Domain bridge operators should publish:
- Node identity and public key
- Served domains (e.g., "clearnet ↔ mesh:lima-district-1")
- Logging policy ("none" is the default and recommended)
- Operator contact for abuse reports (if applicable)

---

## 11. EU Radio Equipment Directive (RED) Compliance

BGP-X Router v1, Node v1, and Gateway v1 hardware sold in the European Union must comply with the Radio Equipment Directive (2014/53/EU, updated by the Radio Equipment Directive 2022).

### 11.1 Requirements

- CE marking
- Declaration of Conformity
- Testing by notified body for relevant radio standards
- User documentation in local language

### 11.2 BGP-X Hardware Compliance

**BGP-X Router v1 and Node v1** hardware sold in EU will be RED-compliant with CE marking. Community manufacturers building from open reference designs are responsible for their own compliance certification.

**Software-only deployments** (BGP-X OpenWrt package on third-party hardware): Regulatory compliance for the radio is the hardware manufacturer's responsibility, not the BGP-X project.

---

## 12. Jurisdiction Declaration and Geographic Plausibility

BGP-X nodes may optionally declare a jurisdiction in their advertisement. This enables geographic plausibility scoring — a reputation signal based on whether RTT measurements are consistent with the claimed region.

### 12.1 Jurisdiction Declaration is OPTIONAL

- Nodes are NOT required to declare jurisdiction
- If jurisdiction is not declared, geo plausibility scoring does NOT apply
- If jurisdiction IS declared, geo plausibility scoring applies
- Declaring jurisdiction is an opt-in privacy/convenience tradeoff

### 12.2 Regulatory Implications of Jurisdiction Declaration

If a node declares a jurisdiction:

1. **Consistency**: BGP-X measures RTT to peers and validates that latency is plausible for the claimed region
2. **Reputation**: Implausible geo scores generate reputation penalties (not hard blocks)
3. **Transparency**: The declared jurisdiction is visible in the node advertisement (public DHT record)

### 12.3 Why a Node Might NOT Declare Jurisdiction

- Privacy: Don't want to reveal any location information
- Mobile/vehicle nodes: Location changes
- Satellite nodes: IP may be geographically distant from terminal location (geo plausibility exempt for satellite-class nodes)

### 12.4 Why a Node Might Declare Jurisdiction

- Reputation: Strong geo plausibility scores are a positive signal
- Trust: Users may prefer nodes in specific jurisdictions
- Regulatory compliance: Some operators may be required to disclose location

**BGP-X recommendation**: Jurisdiction declaration is a personal/operational choice. The network functions identically with or without it.

---

## 13. Web3 and BGP-X

Web3 (blockchain protocols, DeFi, NFT platforms, cross-chain systems like OpenAccord) is an **application layer** that runs over the internet using standard protocols (HTTPS, WebSocket, libp2p). It is NOT a separate BGP-X routing domain.

From BGP-X's perspective:
- Connecting to an Ethereum RPC endpoint is clearnet (UDP/IP over BGP-routed internet)
- Connecting to a Solana validator is clearnet
- Connecting to an IPFS node via libp2p is clearnet

Web3 traffic benefits from BGP-X privacy (IP addresses hidden from RPC providers, validators, and blockchain nodes) but does not require any new BGP-X routing domain. BGP-X treats blockchain traffic identically to any other clearnet application traffic.

**Web3 on mesh islands**: A mesh island user can access Web3 services via a domain bridge node. The bridge node provides clearnet access to blockchain RPCs. The user's mesh address is hidden; only the bridge's WAN IP is visible to the RPC endpoint.
