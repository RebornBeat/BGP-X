# BGP-X Exit Node Operations Guide

**Version**: 0.1.0-draft

This document is the operational guide for BGP-X exit node (gateway) operators. It covers legal considerations, configuration best practices, monitoring, and incident response.

---

## 1. What It Means to Run an Exit Node

Running a BGP-X exit node means that your node sends traffic to clearnet destinations on behalf of BGP-X users. From the perspective of the destination server and any network observer between your node and the destination, the traffic appears to originate from your server's IP address.

This has significant implications:

- **Legal exposure**: Destinations may receive traffic that violates their terms of service. Law enforcement may associate your IP address with that traffic.
- **Abuse complaints**: Your IP may appear in abuse reports (spam, scraping, unauthorized access attempts).
- **Network provider terms**: Your hosting provider may prohibit running exit nodes or forwarding certain traffic.

Running an exit node is a meaningful act of participation in privacy infrastructure. Do not do it without understanding these implications.

---

## 2. Legal Considerations

### 2.1 General principles

BGP-X exit nodes are legally analogous to Tor exit nodes and VPN providers. In most jurisdictions, a network intermediary that:

- Does not select or modify the traffic it carries
- Does not know the content it is forwarding
- Operates under a published, consistent policy
- Responds appropriately to valid legal requests

...has significant but not unlimited legal protection as a neutral carrier or conduit.

This is NOT legal advice. Consult a lawyer in your jurisdiction before running an exit node.

### 2.2 Jurisdiction selection

Consider running your exit node in a jurisdiction with:

- Strong network neutrality protections
- Clear "safe harbor" or "mere conduit" provisions for ISPs/carriers
- A legal tradition of protecting privacy
- No mandatory data retention laws (or retention laws that apply only to ISPs, not operators like you)

Commonly chosen jurisdictions for privacy infrastructure: Iceland, Switzerland, Romania, Germany (with careful legal structuring), Netherlands.

Jurisdictions to research carefully before choosing: United States, United Kingdom, Australia (Five Eyes), any jurisdiction with mandatory data retention laws.

### 2.3 Responding to legal requests

If you receive a legal request (subpoena, court order, law enforcement inquiry) related to your exit node:

1. Do not panic and do not immediately comply without legal advice
2. Consult a lawyer with expertise in internet law in your jurisdiction
3. Verify the request is legally valid (proper jurisdiction, proper process)
4. If you operate a "none" logging policy, explain truthfully that you have no responsive records
5. Consider publishing a warrant canary if your jurisdiction permits

The BGP-X project does not provide legal support for exit node operators. Operators are responsible for their own legal compliance.

---

## 3. Recommended Exit Policy Configuration

### 3.1 Minimal risk exit policy

This policy minimizes abuse potential while still providing useful exit capability:

```toml
[exit_policy]
allow_protocols = ["tcp"]
allow_ports = [80, 443]  # HTTP and HTTPS only
deny_destinations = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
    "100.64.0.0/10",    # Shared address space
    "192.0.0.0/24",     # IETF protocol assignments
    "198.51.100.0/24",  # TEST-NET-2
    "203.0.113.0/24",   # TEST-NET-3
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "DE"
```

Rationale: ports 80 and 443 cover web browsing (the primary use case). Restricting to TCP only eliminates UDP-based DDoS amplification vectors.

### 3.2 Full-service exit policy

For operators willing to accept more risk in exchange for more utility:

```toml
[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = []  # All ports (represented as [0] in the JSON)
deny_destinations = [
    # RFC 1918 and other non-routable
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
    # SMTP (port 25 still denied via port restriction below)
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "IS"  # Iceland
```

Note: Even with all ports open, consider separately restricting port 25 (SMTP) to prevent spam relay. This is commonly enforced by hosting providers regardless.

---

## 4. Abuse Handling

### 4.1 Expected abuse categories

| Category | Frequency | Severity | Response |
|---|---|---|---|
| DMCA/copyright notices | Common | Low | Forward notice; no logs available |
| Port scanning complaints | Occasional | Low | Explain exit node nature |
| Spam complaints | Rare (if port 25 blocked) | Medium | Verify port 25 is blocked |
| Hack attempt reports | Rare | Medium | Explain; review exit policy |
| Law enforcement inquiry | Rare | High | Consult lawyer immediately |

### 4.2 Abuse contact template

Publish the following (or similar) at your abuse contact address:

```
This IP address is a BGP-X exit node.

BGP-X is a privacy-preserving overlay routing network similar to Tor.
Traffic originating from this IP address was forwarded on behalf of
BGP-X users. We do not log, store, or have access to information
about the source of this traffic or its content.

This node operates under the following exit policy:
[link to your published exit policy]

If you have received a legal request, please direct it to:
[legal contact or attorney]

For technical abuse concerns:
[abuse@yourdomain.com]
```

### 4.3 Automated abuse response

Consider deploying an automated abuse response system that:

- Accepts abuse reports at your abuse@ address
- Auto-responds with your exit node explanation
- Logs the complaint category (not content) for your records
- Escalates reports that appear to be valid legal requests

---

## 5. Monitoring an Exit Node

### 5.1 Key metrics to monitor

| Metric | Alert threshold | Description |
|---|---|---|
| Active outbound connections | > 1,800 (90% of limit) | Approaching connection limit |
| Exit policy denial rate | > 20% of streams | Possible misconfiguration |
| DNS resolution failures | > 5% of resolutions | DNS resolver issue |
| Bandwidth utilization | > 90% of cap | Approaching bandwidth limit |
| Session count | > 9,000 (90% of limit) | Approaching session limit |

### 5.2 Health check endpoint

The exit node exposes a health check via the Control API:

```bash
bgpx-cli status
# State: ACTIVE
# Active sessions: 312
# Active streams: 891
# Active outbound connections: 445
# Bytes relayed today: 128 GB
# Exit policy denials today: 23 (protocol:5, port:8, destination:10)
# Uptime: 30d 14h 22m
```

### 5.3 IP reputation monitoring

Monitor your exit node's IP address in public IP reputation services:

- AbuseIPDB
- Spamhaus
- Barracuda Reputation Block List

If your IP appears in blocklists, review your exit policy and consider restricting high-risk ports.
