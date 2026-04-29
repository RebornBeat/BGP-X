# BGP-X Operator Liability and Legal Considerations

**Version**: 0.1.0-draft

---

## Important Notice

This document provides general educational information about legal considerations for BGP-X node operators. **It is not legal advice.** Laws vary by jurisdiction and change over time. Operators should consult qualified legal counsel in their jurisdiction before operating any BGP-X node, particularly exit nodes, gateway nodes, and domain bridge nodes.

---

## 1. Technical Architecture and Liability Relevance

### What BGP-X Nodes Process

BGP-X relay nodes process encrypted packets. The key technical facts:

- **Relay nodes see**: encrypted ciphertext; their own decryption reveals only the next hop address and a path identifier. They never see source IPs of the original sender, destination addresses, or content.
- **Exit/gateway nodes see**: the clearnet destination and content after their decryption layer (the final hop to clearnet). They do not know the source IP of the original sender.
- **Domain bridge nodes see**: their predecessor address in the source domain and the first relay address in the target domain. They do not know the original sender's IP or the final destination. In cross-domain paths, bridge nodes forward inner onion layers without decryption — they cannot read the payload or determine the service being accessed.
- **No node logs** any session-level identifying information (prohibited under BGP-X SOPs). Technical enforcement prevents access logs.

### Analogy to Existing Infrastructure

BGP-X relay nodes function analogously to internet exchange points and transit providers: they route encrypted traffic without knowledge of its content or endpoints. Exit/gateway nodes are functionally analogous to internet service providers providing clearnet access to anonymous users.

---

## 2. Relay Node Operators

### General Position

Relay nodes process only encrypted ciphertext. They have no technical ability to determine what traffic they relay, where it originates, or its destination. This position is similar to transit ISP operators who carry traffic without knowledge of its content.

### Jurisdictional Variation

- **European Union**: The EU E-Commerce Directive (and its successor the Digital Services Act) provides conduit liability exemptions for operators who do not initiate transmissions, do not select transmission recipients, and do not modify transmitted content. BGP-X relay nodes technically satisfy these conditions.

- **United States**: Section 230 of the Communications Decency Act and common carrier considerations may be relevant. Operators should be aware of the Computer Fraud and Abuse Act with respect to unauthorized access through their nodes.

- **Other jurisdictions**: Legal frameworks vary widely. Some jurisdictions have no clear intermediary liability framework. Operators in such jurisdictions carry greater legal uncertainty.

### Recommended Practices

- Maintain documentation of the technical architecture explaining why logs cannot exist
- Register as a legal entity if operating a high-traffic node
- Consult ISP/transit liability precedents in your jurisdiction
- Review applicable data retention laws before deploying

---

## 3. Exit and Gateway Node Operators

### Heightened Exposure

Exit node operators carry greater legal exposure than relay operators because:

1. Their IP address is the visible origin of connections to internet services
2. Law enforcement investigations may identify their server IP as the source of traffic
3. Abuse complaints from website operators may be directed to their ASN/hosting provider

### DMCA and Copyright (United States)

Exit node operators whose IP appears in DMCA takedown complaints should have a registered DMCA agent and an established process for responding to notices. Because they cannot identify the user who generated the traffic, they typically cannot comply with content removal requests and should explain this in their response process.

### Law Enforcement Requests

Operators should prepare a standard response explaining the BGP-X architecture:

> "This server operates as an exit node in the BGP-X privacy overlay network. By design, no connection logs, session logs, or identifying information about users whose traffic exited through this server are retained or technically collectible. The architecture uses onion encryption that prevents the exit node operator from knowing the identity of any user."

Operators should consult legal counsel regarding their jurisdiction's obligations when receiving law enforcement requests.

### Recommended Practices for Exit Operators

- Publish a clear exit policy documenting what traffic the node does and does not forward
- Establish an abuse contact address
- Choose a hosting provider in a jurisdiction with strong intermediary liability protections
- Consider reverse DNS pointing to a domain explaining the node's function
- Prepare a standard response to abuse complaints and legal requests
- Maintain only the minimum necessary operational records (no session logs, no connection logs)
- Keep records of when exit policy was in effect (for demonstrating no prohibited content forwarding)

---

## 4. Domain Bridge Node Operators

Domain bridge nodes occupy a position similar to relay nodes with respect to traffic they process: they forward encrypted onion layers between routing domains without decrypting inner content. The bridge node's DOMAIN_BRIDGE hop decryption reveals only the next relay node ID in the target domain — not source IP, not destination, not payload.

**Cross-domain routing**: When a bridge node facilitates traffic between a clearnet client and a mesh island service, or between two mesh islands, it processes encrypted packets in both domains. In neither direction can the operator determine the source or destination of the traffic, or its content.

**Operator responsibility for bridge node advertising**: Operators advertising bridge node capability (`bridge_capable = true`) are responsible for ensuring their bridge infrastructure is available as advertised. Repeatedly advertising availability while being unavailable causes harm to users who construct paths relying on those bridges and may expose the operator to reputation system penalties and community sanctions.

**Specific considerations for domain bridge operators**:
- The bridge node facilitates access between routing domains. This increases the range of traffic types that may traverse the node, including traffic from mesh island communities that may be in jurisdictions with different legal norms.
- Bridge operators should understand that they technically enable connectivity between communities and should consider whether local law creates any obligations related to that facilitation.
- The encrypted nature of all traffic means the operator has no technical ability to monitor or filter it (beyond the coarse exit policy on clearnet-exit segments, which applies only at exit nodes, not bridge nodes).

---

## 5. Mesh Island Gateway Operators

Mesh island gateway operators are bridge node operators who also serve as the internet access point for a mesh community.

**Additional considerations**:

- Users of the mesh island whose internet traffic exits via the gateway have legal exposure channeled through the gateway operator's infrastructure.
- Gateway operators should consider whether their jurisdiction's ISP regulations apply when they are providing internet access to third parties.
- If the mesh island is operated in a context of political sensitivity or civil unrest, gateway operators should understand potential government pressure to disable or surveil the gateway.

**Recommended practice**: mesh island gateways SHOULD be operated with at least two independent operators in different jurisdictions, so that no single operator's legal pressure can isolate the community.

---

## 6. Data Retention Laws

Many jurisdictions impose data retention obligations on internet service providers and electronic communications operators.

BGP-X node operators should determine:

1. Whether their jurisdiction's data retention laws apply to their operation
2. If applicable, whether the law requires retention of session-level identifying information that BGP-X's architecture prohibits retaining
3. If there is a conflict between data retention obligations and BGP-X's technical design, they should consult legal counsel regarding whether an exemption or technical infeasibility defense is available

BGP-X SOP prohibits logging session identifiers, source IPs, and destinations regardless of any external requirement. Operators who face mandatory retention obligations that conflict with this SOP must resolve the conflict before deploying.

---

## 7. Export Control

BGP-X implements ChaCha20-Poly1305 encryption with 256-bit keys. This may be subject to export control regulations under the Wassenaar Arrangement and implementing national regulations (U.S. EAR, EU Dual-Use Regulation, etc.).

Key considerations:
- Most jurisdictions have broad exemptions for mass-market encryption software
- Open-source status and public availability of source code affects classification
- Operators distributing BGP-X binaries to users in other countries should review applicable export control regulations

**Note**: This is not a comprehensive export control analysis. Operators concerned about export control should consult legal counsel specializing in export compliance.

---

## 8. No Operator Coordination

BGP-X is decentralized. There is no BGP-X operating company that:
- Controls the network
- Can surveil user traffic
- Holds logs
- Receives legal process on behalf of operators
- Indemnifies operators

Each operator is independently responsible for their node's operation and legal compliance in their jurisdiction.

---

## 9. Changes to This Document

Legal landscapes change. This document reflects general considerations at the time of writing. Operators should review this document periodically and consult current legal counsel for up-to-date advice.
