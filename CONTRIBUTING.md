# Contributing to BGP-X

Thank you for your interest in contributing to BGP-X.

BGP-X is a serious infrastructure project. Contributions are held to a high standard because the system is intended to protect people under real-world adversarial conditions. Mistakes in a networking or cryptography project are not just bugs — they can be privacy failures.

---

## Before You Contribute

Read the following documents before submitting any contribution:

1. [ARCHITECTURE.md](./ARCHITECTURE.md) — understand the full system design
2. [/protocol/protocol_spec.md](./protocol/protocol_spec.md) — understand the protocol
3. [/security/threat_model.md](./security/threat_model.md) — understand the adversary model

If you propose a change that violates a design principle or weakens a security property, it will be rejected regardless of implementation quality.

---

## Types of Contributions

### Documentation

- Corrections to typos, grammar, or factual errors
- Clarification of ambiguous specifications
- Additional worked examples
- Translation (see Translation section below)

### Specification Changes

Specification changes require:

1. Opening an issue describing the proposed change and its motivation
2. A design discussion period (minimum 7 days for protocol-level changes)
3. Sign-off from at least 2 maintainers
4. Updated documentation with the change

### Implementation (once codebase exists)

- Bug fixes (must include regression test)
- Performance improvements (must include benchmark)
- New features (must have an open accepted specification issue first)

---

## Contribution Process

1. Fork the repository
2. Create a branch: `git checkout -b type/short-description`
   - Types: `docs/`, `spec/`, `fix/`, `feat/`
3. Make your changes
4. Commit with a clear message (see Commit Format below)
5. Push and open a Pull Request

---

## Commit Format

BGP-X uses conventional commits:

```
type(scope): short description

Body (optional): longer description of what and why.

Refs: #issue-number
```

Types:
- `docs` — documentation only
- `spec` — specification change
- `fix` — bug or spec correction
- `feat` — new feature or capability
- `security` — security-related change
- `chore` — maintenance, tooling

---

## What Will Be Rejected

- Changes that introduce a single point of trust or central authority
- Changes that weaken anonymity properties
- Changes that break backward compatibility without a versioning migration path
- "Just trust us" changes without cryptographic enforcement
- Code without tests (once implementation phase begins)
- Protocol changes without updated specification

---

## Cryptography Contributions

If your contribution involves cryptographic primitives, algorithms, or protocol design:

- You must cite prior art and security analysis
- Changes will not be accepted based on performance alone if they reduce security margins
- No home-grown cryptographic primitives will be accepted

---

## Security Vulnerabilities

Do NOT open a public issue for security vulnerabilities. See [SECURITY.md](./SECURITY.md) for responsible disclosure.

---

## Code of Conduct

All contributors are expected to follow the [Code of Conduct](./CODE_OF_CONDUCT.md).

---

## Translation

BGP-X documentation is written in English as the canonical source. Translations are welcome but must:

- Be clearly marked as translations
- Include the original document's version/date
- Not be relied upon for security-critical interpretation (English is authoritative)
