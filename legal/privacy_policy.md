# BGP-X Privacy Policy

**Version**: 0.1.0-draft

---

## Applicability

This policy applies to:
- The BGP-X protocol specification
- The reference implementation (bgpx-node daemon, bgpx-cli, bgpx-sdk)
- Official BGP-X infrastructure (bootstrap nodes operated by the BGP-X project)

Individual node operators who deploy BGP-X software operate under their own privacy policies and legal obligations. This policy does not govern third-party node operators.

---

## 1. Core Privacy Principle

BGP-X is designed to make user surveillance technically impossible at the network layer. This policy documents what is and what is not collected, by design.

---

## 2. What the BGP-X Protocol Does Not Collect

The BGP-X protocol is designed so that no single node in the network can observe the complete picture of any user's activity. Specifically:

**Entry nodes (first relay hop)**:
- Can see: the connecting client's IP address
- Cannot see: the destination of the traffic; the content; the complete path

**Relay nodes (middle hops)**:
- Can see: the previous relay's address; the next relay's address
- Cannot see: the originating client's IP; the destination; the content

**Exit/gateway nodes (last hop to clearnet)**:
- Can see: the clearnet destination; the traffic content (after their decryption layer)
- Cannot see: the originating client's IP address; the complete path

**Domain bridge nodes**:
- Can see: their immediate predecessor in the source domain; the first relay in the target domain
- Cannot see: originating client IP; final destination; content (inner onion layers remain encrypted); full path composition; which routing domains the complete path traverses; cross-domain traversal details

**No node logs**:
- Source IP addresses of incoming connections
- Session identifiers
- path_id values
- Routing paths or path compositions
- Destination addresses (except exit nodes during active connection only — not retained)
- Stream content at any hop
- Timing metadata that could be associated with specific sessions
- Cross-domain traversal details (which routing domains a session traversed, which mesh islands were involved, which bridge nodes were used)
- Mesh island identifiers associated with specific sessions or users

---

## 3. What Official BGP-X Infrastructure Collects

### Bootstrap Nodes

Official BGP-X bootstrap nodes collect:
- **DHT participation**: connecting node's IP address and NodeID (required for DHT routing)
- **DHT records**: signed node advertisements, domain bridge records, mesh island records
- **Operational logs**: daemon start/stop events, DHT routing table size, error events

Bootstrap nodes do NOT collect:
- Information about which routes, services, or destinations any user accesses
- Session content
- Relay path compositions
- Any cross-domain traversal associations

DHT records are published by node operators (not users) and contain only information operators choose to advertise about themselves.

### Official Repository and Documentation

The BGP-X project website and documentation may collect:
- Standard web server access logs (IP, URL, timestamp)
- These are retained for no more than 30 days and used only for operational purposes

### Issue Tracker and Contributions

Contributors to the BGP-X project via the public repository may have their contributions, usernames, and associated metadata retained as part of the public version control record.

---

## 4. Third-Party Node Operators

When users connect to BGP-X through independently operated nodes, those operators' privacy practices govern what (if anything) is collected at those nodes. BGP-X SOP (Standard Operating Procedures) prohibits logging identifying information. However, the BGP-X project cannot technically prevent non-compliant operators from logging, and cannot guarantee all operators comply.

**User trust model**: Users should not need to trust any individual node with their complete activity. The multi-hop architecture limits what any single node operator can observe. No single operator sees both source IP and destination.

---

## 5. What BGP-X Cannot Protect Against

BGP-X protects network-layer metadata. It does not protect against:

- **Application-layer identification**: websites tracking you via cookies, browser fingerprinting, or account login
- **Compromised device**: malware on your device can capture traffic before it enters the BGP-X overlay
- **Global passive adversary**: an adversary who observes all internet traffic simultaneously can potentially perform timing correlation attacks against any low-latency anonymity system
- **Endpoint compromise**: if the server you connect to is compromised, your content may be visible there (though not your identity)
- **Traffic confirmation attacks**: with observation of both the first and last hop of a path, statistical traffic analysis may be possible

---

## 6. Cryptographic Privacy Guarantees

BGP-X provides the following cryptographic guarantees:

- **Confidentiality**: traffic is encrypted with ChaCha20-Poly1305 (256-bit key) at every hop; each hop uses a distinct session key
- **Authentication**: all hop session keys are derived via X25519 ECDH with the hop's Ed25519 static key; modification by intermediaries is cryptographically detectable
- **Forward secrecy**: ephemeral X25519 keys are used per session and zeroized after key derivation; compromise of long-term static keys does not compromise past sessions
- **Indistinguishable cover traffic**: COVER packets use the same session_key as RELAY packets; external observers cannot distinguish COVER from RELAY traffic in any routing domain
- **Domain-agnostic cryptography**: identical cryptographic suite applied in all routing domains (clearnet, mesh, satellite); no domain-specific key derivation or cipher variation

---

## 7. Data Minimization by Design

BGP-X applies data minimization at the protocol level:

- path_id is 8 bytes of random data with no relationship to user identity, IP address, or destination
- Session IDs are 16 bytes of random data; they are rotated per session
- DHT records contain only information operators choose to publish about their nodes
- No persistent user identifiers exist at the protocol level
- Mesh island records contain only information island operators choose to publish (island ID, bridge nodes, service IDs — no user associations)

---

## 8. User Rights

For BGP-X official infrastructure (bootstrap nodes, repository, documentation):

- **Access**: users may request what information is held about them by official infrastructure
- **Deletion**: users may request deletion of personally identifiable information from official infrastructure records (where technically feasible; DHT records published by the user's own node cannot be deleted by the project)
- **Portability**: operational logs pertaining to a user's node can be provided on request

For third-party node operators: contact the operator directly.

---

## 9. Changes to This Policy

Significant changes to this privacy policy will be announced via the BGP-X release notes and operator mailing list. The policy version is reflected in the CHANGELOG.

---

## 10. Contact

Privacy concerns regarding official BGP-X infrastructure: see SECURITY.md for contact information.

For general questions about the privacy architecture, refer to the technical documentation at `/security/threat_model.md` and `/docs/faq.md`.
