# BGP-X Security Policy

## Reporting a Vulnerability

BGP-X is infrastructure software. Vulnerabilities in it can have serious privacy implications for real people.

**Do NOT open a public GitHub issue for any security vulnerability.**

### How to report

Send a detailed description of the vulnerability to the maintainers via encrypted email or a private channel.

Your report should include:

1. **Description** — What is the vulnerability?
2. **Impact** — What can an attacker do? Who is affected?
3. **Reproduction** — Step-by-step instructions to reproduce
4. **Environment** — Version, OS, configuration
5. **Proposed fix** (optional but appreciated)

### What happens next

1. We will acknowledge receipt within 48 hours
2. We will provide an initial assessment within 7 days
3. We will work with you on a coordinated disclosure timeline
4. We will credit you in the security advisory (if desired)

---

## Supported Versions

During the pre-implementation specification phase, all specification documents are considered active.

Once implementation begins, the support matrix will be maintained here.

---

## Security Design Principles

BGP-X security is based on the following non-negotiable principles:

- All security properties must be cryptographically enforced, not policy-dependent
- No single party should be able to link sender to receiver
- The compromise of any individual node must not compromise the anonymity of the full network
- path_id enables return routing without leaking path composition to intermediate relays
- COVER packets use session_key — externally indistinguishable from RELAY packets
- Pool trust does not chain: individual node verification required regardless of pool membership
- Global blacklists are advisory; local enforcement is authoritative

---

## Pool-Specific Security

- **Curator compromise**: pool membership can be manipulated by a compromised curator; individual node signature verification provides defense; cross-pool diversity limits blast radius
- **Pool trust isolation**: trust is not transitive across pools; each segment's nodes are independently verified
- **Private pool membership**: private pool member lists must not be disclosed; pool curator key must be in HSM for production deployments

---

## Routing Policy Security

- Routing policy bypass is possible in dual-stack mode by design (bypass is a feature)
- BGP-X-only mode eliminates all bypass paths for LAN devices
- SDK Socket exposure: if TCP endpoint exposed to LAN, authentication is mandatory; never expose to WAN

---

## Mesh Security

- All mesh transport traffic is encrypted by the BGP-X overlay (ChaCha20-Poly1305)
- Radio interception of LoRa/WiFi mesh traffic cannot yield plaintext
- Rogue mesh nodes: invalid advertisements are rejected; reputation system penalizes anomalous behavior
- Gateway compromise: gateway knows destination but not source; same as exit node compromise at internet level

---

## Cryptographic Suite (Summary)

| Function | Algorithm |
|---|---|
| Asymmetric signatures | Ed25519 |
| Key exchange | X25519 (ECDH) |
| Symmetric encryption | ChaCha20-Poly1305 |
| Hashing | BLAKE3 |
| KDF | HKDF-SHA256 |

No separate cover_key is derived. COVER packets use the same session_key as RELAY packets.

Pool curator keys are Ed25519 keypairs separate from node identity keys.

---

## Prohibited Logging

The following MUST NEVER be logged by any BGP-X component:

- Client IP addresses at relay or exit nodes
- Destination addresses at relay nodes
- path_id values
- Path composition (which nodes appeared in the same path)
- Session identifiers
- Traffic volume per path
- Pool query history
- Application data content

---

## Known Limitations

See `/security/threat_model.md` for a complete description of what BGP-X protects against and what it does not.

The primary known limitation is resistance to global passive adversaries performing traffic correlation. This is a fundamental challenge for all low-latency anonymity networks.

ECH hides domain names from SNI observation but requires destination support via DNS HTTPS record publication.

Geographic plausibility scoring is an RTT-based signal, not cryptographic proof of node location.
