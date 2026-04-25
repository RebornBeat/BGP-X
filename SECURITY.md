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
- All cryptographic decisions are documented in `/security/crypto_spec.md`

---

## Cryptographic Suite (Summary)

| Function | Algorithm |
|---|---|
| Asymmetric signatures | Ed25519 |
| Key exchange | X25519 (ECDH) |
| Symmetric encryption | ChaCha20-Poly1305 |
| Hashing | BLAKE3 |
| KDF | HKDF-SHA256 |

Any proposed change to the cryptographic suite requires a full security analysis and specification update before implementation.

---

## Known Limitations

See `/security/threat_model.md` for a complete description of what BGP-X protects against and what it does not.

The primary known limitation is resistance to global passive adversaries performing traffic correlation. This is a fundamental challenge for all low-latency anonymity networks and is documented in detail in the threat model.
