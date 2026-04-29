# BGP-X Exit Node Specification

**Version**: 0.1.0-draft

---

## 1. Exit Node Role

An exit node (gateway) is the final hop for clearnet traffic. The exit node:
- Knows the destination (IP, domain name, or resolved DNS)
- Does NOT know the source (client IP)
- Does NOT know the path before it
- Enforces exit policy on outbound connections
- Handles ECH for destinations that support it

Exit nodes are a specific type of domain bridge node: they bridge from the BGP-X overlay to the public clearnet internet.

---

## 2. What the Exit Node Sees

| Information | Visible to Exit Node |
|---|---|
| Destination IP or domain | ✅ Yes |
| Application layer content (if HTTPS) | ❌ No (encrypted by TLS) |
| Source (client IP) | ❌ No |
| Number of hops before it | ❌ No |
| Whether path is cross-domain | ❌ No |
| Domain name (if ECH available) | ❌ No (hidden by ECH) |
| Domain name (if ECH unavailable) | ⚠️ Yes (from TLS SNI) |

---

## 3. Exit Policy Enforcement

All exit node operators MUST publish a signed exit policy. The exit policy is enforced technically, not by trust.

```toml
[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = [
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "127.0.0.0/8", "169.254.0.0/16", "100.64.0.0/10",
    "::1/128", "fc00::/7", "fe80::/10"
]
logging_policy = "none"
operator_contact = "ops@example.com"
jurisdiction = "DE"
```

When a connection is denied by exit policy: STREAM_CLOSE with ERR_EXIT_POLICY_DENIED. The exit node does NOT reveal why the connection was denied or what the deny list contains.

---

## 4. DNS Resolution

All DNS resolution at exit nodes MUST use DoH (DNS over HTTPS) with DNSSEC validation:

```toml
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
```

ECS (EDNS Client Subnet) MUST be stripped from all DNS queries. Revealing the client subnet to DNS resolvers would leak approximate client location.

---

## 5. ECH (Encrypted Client Hello)

When `ech_capable = true` and the destination supports ECH:

1. Exit node queries DNS for destination's ECH configuration (HTTPS record)
2. Constructs TLS ClientHello with ECH (using destination's ECH public key)
3. TLS ServerName Indication (SNI) contains a cover name, not the real domain
4. Real destination domain is encrypted inside ECH and only decryptable by the destination server

ECH hides the domain name from network-level observers between the exit node and destination. Does not apply if destination does not publish ECH configuration.

---

## 6. Exit Node Cross-Domain Context

Exit nodes in cross-domain paths work identically to single-domain exit nodes. A path that traverses mesh islands before reaching the exit node looks identical to the exit node as a single-domain path. The exit node does not know the path traversed mesh domains.

---

## 7. Exit Policy Versioning

When exit policy changes:
1. Increment `exit_policy_version`
2. Set `exit_policy_previous_version` to old version number
3. Re-publish advertisement immediately
4. Old policy honored for existing streams until they close
5. New policy applied to new streams immediately

Clients checking exit policy before stream establishment always see current policy.

---

## 8. Operator Responsibilities

Exit node operators are responsible for:
- Publishing accurate exit policy
- Responding to abuse complaints via operator_contact
- Complying with local laws regarding traffic they handle
- NOT logging connection-level data (technical enforcement by logging_policy = "none")
- Running up-to-date BGP-X software

BGP-X provides no legal protection. Operators should consult legal advice for their jurisdiction.
