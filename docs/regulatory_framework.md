# BGP-X Regulatory Framework

This document covers the regulatory considerations for BGP-X deployments, particularly for radio-based mesh networks and elevated relay deployment.

---

## Radio Spectrum Regulations

### LoRa / ISM Band (Primary Mesh Transport)

LoRa operates in the Industrial, Scientific, and Medical (ISM) bands. These are license-free in most jurisdictions, but subject to power and duty cycle limits.

| Region | Frequency | Max Power | Duty Cycle | License |
|---|---|---|---|---|
| European Union | 868 MHz | 25 mW ERP | 1% (some sub-bands) | License-free |
| United States | 915 MHz | 30 dBm (1W) | No limit (but FHSS rules apply) | License-free (Part 15) |
| Canada | 915 MHz | 30 dBm | Part 15 rules | License-free |
| Australia | 915 MHz | 30 dBm | No limit | License-free |
| Japan | 920 MHz | 20 mW | Regulations apply | License-free |
| General (433 MHz) | 433.05-434.79 MHz | 10 mW | 10% | License-free in most regions |

**BGP-X compliance**: BGP-X LoRa nodes operate within these limits by default. Configuration allows adjusting power and duty cycle per regional requirements.

**EU duty cycle**: 1% duty cycle limits LoRa nodes to ~36 seconds of transmit time per hour. BGP-X is designed for this — LoRa is used for DHT sync, key exchange, and small control messages, not continuous data streams.

### WiFi 802.11s (Primary Mesh Transport for Dense Deployment)

WiFi operates in the 2.4 GHz and 5 GHz ISM/U-NII bands.

| Frequency | Jurisdiction | License | Notes |
|---|---|---|---|
| 2.4 GHz (802.11b/g/n/ax) | Global | License-free (limits vary) | Universal availability |
| 5 GHz (802.11a/n/ac/ax) | Most regions | License-free with restrictions | Some channels require DFS |
| 6 GHz (802.11ax WiFi 6E) | US, EU, others | License-free (low power indoor) | Newer, faster |

**BGP-X compliance**: 802.11s mesh operates on standard WiFi channels. No special licensing required.

### Bluetooth BLE

Bluetooth 5.0 mesh operates at 2.4 GHz, globally license-free, 100mW maximum power. No regulatory concerns for BGP-X Bluetooth mesh use.

---

## Fixed Antenna Installation (Recommended for Elevated Deployment)

For permanent elevated relay installation (masts, rooftops, towers):

### Typical Regulatory Path

Fixed antenna installations are regulated by planning/building codes, not aviation authorities. This is the simplest path for permanent elevated coverage.

| Country | Regulatory Body | Process |
|---|---|---|
| USA | Local zoning + FCC (for licensed spectrum, not needed for ISM) | Building permit for mast if above threshold height |
| EU | National planning authorities | Varies by country; many exempt small antennas |
| UK | Permitted Development Rights | Small antennas often exempt from planning permission |

**BGP-X recommendation**: Fixed masts at rooftop or utility pole height (10-30m) are the most practical permanent elevated relay deployment. Regulatory burden is minimal for ISM-band antennas at these heights.

---

## Tethered Aerostat Regulations

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

## Drone / UAV Regulations

Drones for BGP-X relay deployment face stricter regulation than tethered aerostats.

### Altitude Limits

| Country | Standard Limit | Notes |
|---|---|---|
| USA (FAA Part 107) | 400 ft (122m) AGL | Waivers available up to ~800 ft for justified operations |
| European Union (EASA) | 120m AGL | 150m with authorization for justified operations |
| UK (CAA) | 120m AGL | |
| Canada (Transport Canada) | 122m AGL | |
| Australia (CASA) | 120m AGL | |

### Over Private Property

In the USA, the FAA regulates airspace — not specifically who you fly over on private property. State laws vary. In the EU and UK, regulations are more restrictive near residential areas.

### BGP-X Drone Relay Assessment

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

## Maritime Operations

BGP-X maritime nodes operate in unlicensed ISM bands (LoRa, WiFi). No special maritime radio license is required for ISM-band operation.

BGP-X does NOT use licensed maritime frequencies (VHF, HF marine bands). Those require operator licenses.

**In international waters**: Generally more flexible ISM regulations. Check flag state requirements.

**In coastal areas**: Follow local national regulations.

---

## Satellite Terminal Regulations

### Starlink

Starlink requires a user agreement and service registration with SpaceX. Available in most countries. Some countries restrict or prohibit satellite internet terminal operation.

**Regulatory check**: Confirm Starlink is licensed in your jurisdiction before deploying a BGP-X satellite gateway.

### Iridium, Inmarsat

Licensed service; operator provides regulatory compliance within coverage area.

---

## Cryptographic Software Export

BGP-X implements strong cryptography (AES-256 equivalent security, ChaCha20-Poly1305). Export regulations apply in some jurisdictions.

### United States (EAR)

Encryption software generally falls under ECCN 5D002. Mass-market encryption software is generally eligible for License Exception ENC after a one-time review. Open source cryptography generally has more permissive rules.

**BGP-X**: as open source software, check current EAR rules. BGP-X contributors should verify compliance for their jurisdiction before contributing to or distributing the software.

### European Union

EU generally does not restrict the export of publicly available cryptographic software.

**Recommendation**: consult a lawyer familiar with export control in your jurisdiction if you have concerns about deploying BGP-X in a specific country or exporting the software.

---

## Privacy and Data Regulations

BGP-X nodes — particularly exit nodes — may be subject to data retention regulations in their jurisdiction.

**BGP-X design**: the default logging policy is "none" — no traffic data is logged. Exit nodes cannot produce records of connections they never stored.

**Operator responsibility**: verify the legal requirements for node operation in your jurisdiction. See `/gateway/exit_node.md` for legal considerations for exit node operators.

**Data retention laws**: some jurisdictions (EU member states under national implementation of electronic communications law, others) require ISPs and network operators to retain traffic metadata. Whether BGP-X node operators qualify as "ISPs" or "network operators" for these purposes varies by jurisdiction and legal interpretation. Consult a lawyer.
