# BGP-X Exit Node Operations Guide

**Version**: 0.1.0-draft

This document is the operational guide for BGP-X exit node (gateway) operators. It covers legal considerations, configuration best practices, monitoring, abuse handling, cross-domain operations, and incident response.

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

### 2.1 General Principles

BGP-X exit nodes are legally analogous to Tor exit nodes and VPN providers. In most jurisdictions, a network intermediary that:

- Does not select or modify the traffic it carries
- Does not know the content it is forwarding
- Operates under a published, consistent policy
- Responds appropriately to valid legal requests

...has significant but not unlimited legal protection as a neutral carrier or conduit.

**This is NOT legal advice. Consult a lawyer in your jurisdiction before running an exit node.**

### 2.2 Jurisdiction Selection

Consider running your exit node in a jurisdiction with:

- Strong network neutrality protections
- Clear "safe harbor" or "mere conduit" provisions for ISPs/carriers
- A legal tradition of protecting privacy
- No mandatory data retention laws (or retention laws that apply only to ISPs, not operators like you)

Commonly chosen jurisdictions for privacy infrastructure: Iceland, Switzerland, Romania, Germany (with careful legal structuring), Netherlands.

Jurisdictions to research carefully before choosing: United States, United Kingdom, Australia (Five Eyes), any jurisdiction with mandatory data retention laws.

### 2.3 Responding to Legal Requests

If you receive a legal request (subpoena, court order, law enforcement inquiry) related to your exit node:

1. Do not panic and do not immediately comply without legal advice
2. Consult a lawyer with expertise in internet law in your jurisdiction
3. Verify the request is legally valid (proper jurisdiction, proper process)
4. If you operate a "none" logging policy, explain truthfully that you have no responsive records
5. Consider publishing a warrant canary if your jurisdiction permits

The BGP-X project does not provide legal support for exit node operators. Operators are responsible for their own legal compliance.

---

## 3. What the Exit Node Sees

An exit node is the final hop for clearnet traffic. The exit node knows certain information but not the full path:

| Information | Visible to Exit Node |
|---|---|
| Destination IP or domain | ✅ Yes |
| Application layer content (if HTTPS) | ❌ No (encrypted by TLS) |
| Source (client IP) | ❌ No |
| Number of hops before it | ❌ No |
| Whether path is cross-domain | ❌ No |
| Domain name (if ECH available) | ❌ No (hidden by ECH) |
| Domain name (if ECH unavailable) | ⚠️ Yes (from TLS SNI) |

**Important**: Exit nodes do not know whether the path traversed mesh islands before reaching them. A path that traverses multiple mesh islands before reaching the exit node looks identical to the exit node as a single-domain clearnet path. The exit node only knows it is the final hop before clearnet.

---

## 4. Exit Policy Enforcement

All exit node operators MUST publish a signed exit policy. The exit policy is enforced technically, not by trust.

When a connection is denied by exit policy: `STREAM_CLOSE` with `ERR_EXIT_POLICY_DENIED`. The exit node does NOT reveal why the connection was denied or what the deny list contains.

### 4.1 Minimal Risk Exit Policy

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
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

Rationale: Ports 80 and 443 cover web browsing (the primary use case). Restricting to TCP only eliminates UDP-based DDoS amplification vectors.

### 4.2 Full-Service Exit Policy

For operators willing to accept more risk in exchange for more utility:

```toml
[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = []  # All ports (represented as empty array meaning no restriction)
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
    # SMTP (port 25 still denied via separate port restriction below)
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "IS"  # Iceland
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

Note: Even with all ports open, consider separately restricting port 25 (SMTP) to prevent spam relay. This is commonly enforced by hosting providers regardless.

---

## 5. DNS Resolution

All DNS resolution at exit nodes MUST use DoH (DNS over HTTPS) with DNSSEC validation:

```toml
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
```

ECS (EDNS Client Subnet) MUST be stripped from all DNS queries. Revealing the client subnet to DNS resolvers would leak approximate client location.

DoH is required because:
- Plain DNS (UDP/53) is visible to network observers
- DoT (DNS over TLS) is acceptable but DoH is preferred for its HTTPS disguise
- The exit node's DNS queries should not reveal BGP-X traffic patterns

---

## 6. ECH (Encrypted Client Hello)

When `ech_capable = true` and the destination supports ECH:

1. Exit node queries DNS for destination's ECH configuration (HTTPS DNS record)
2. Constructs TLS ClientHello with ECH (using destination's ECH public key)
3. TLS ServerName Indication (SNI) contains a cover name, not the real domain
4. Real destination domain is encrypted inside ECH and only decryptable by the destination server

ECH hides the domain name from network-level observers between the exit node and destination. This includes the exit node's own ISP and any passive monitoring.

**ECH does not apply if destination does not publish ECH configuration.** In that case, the domain name is visible in TLS SNI to the exit node and all observers between exit and destination.

### 6.1 ECH Configuration and Testing

Verify ECH is working:

```bash
# Test ECH for a specific destination
bgpx-cli exit test-ech --destination example.com

# Expected output:
# ECH config found: YES
# ECH negotiation result: SUCCESS
# SNI visible to observer: (none — ECH used)

# View cached ECH configs
bgpx-cli exit ech-cache

# Force re-fetch ECH configs
bgpx-cli exit clear-ech-cache
```

### 6.2 ECH Caching

Exit nodes cache ECH configurations with a 1-hour TTL. When the cache expires, the exit node re-queries DNS for the ECH configuration. This reduces DNS query overhead for frequently accessed destinations.

---

## 7. Exit Policy Versioning

When exit policy changes:

1. Increment `exit_policy_version`
2. Set `exit_policy_previous_version` to old version number
3. Re-publish advertisement immediately
4. Old policy honored for existing streams until they close
5. New policy applied to new streams immediately

Clients checking exit policy before stream establishment always see current policy. If a client has a cached policy and sees `exit_policy_version` has changed, it re-fetches the full advertisement.

---

## 8. Abuse Handling

### 8.1 Expected Abuse Categories

| Category | Frequency | Severity | Response |
|---|---|---|---|
| DMCA/copyright notices | Common | Low | Forward notice; no logs available |
| Port scanning complaints | Occasional | Low | Explain exit node nature |
| Spam complaints | Rare (if port 25 blocked) | Medium | Verify port 25 is blocked |
| Hack attempt reports | Rare | Medium | Explain; review exit policy |
| Law enforcement inquiry | Rare | High | Consult lawyer immediately |

### 8.2 Abuse Contact Template

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

### 8.3 Automated Abuse Response

Consider deploying an automated abuse response system that:

- Accepts abuse reports at your abuse@ address
- Auto-responds with your exit node explanation
- Logs the complaint category (not content) for your records
- Escalates reports that appear to be valid legal requests

---

## 9. Monitoring

### 9.1 Key Metrics to Monitor

| Metric | Alert threshold | Description |
|---|---|---|
| Active outbound connections | > 1,800 (90% of limit) | Approaching connection limit |
| Exit policy denial rate | > 20% of streams | Possible misconfiguration |
| DNS resolution failures | > 5% of resolutions | DNS resolver issue |
| Bandwidth utilization | > 90% of cap | Approaching bandwidth limit |
| Session count | > 9,000 (90% of limit) | Approaching session limit |
| ECH fallback rate | High (informational) | Many destinations without ECH (expected) |

### 9.2 Health Check Endpoint

The exit node exposes a health check via the Control API:

```bash
bgpx-cli stats
# State: ACTIVE
# Active sessions: 312
# Active streams: 891
# Active outbound connections: 445
# Bytes relayed today: 128 GB
# Exit policy denials today: 23 (protocol:5, port:8, destination:10)
# ECH connections: 156 (ECH fallback: 34)
# Uptime: 30d 14h 22m
```

### 9.3 IP Reputation Monitoring

Monitor your exit node's IP address in public IP reputation services:

- AbuseIPDB
- Spamhaus
- Barracuda Reputation Block List

If your IP appears in blocklists, review your exit policy and consider restricting high-risk ports.

---

## 10. Pool Participation for Exit Nodes

Register with curated exit pools to increase visibility to privacy-conscious users:

1. Contact pool curator
2. Provide node_id and exit policy details
3. Meet pool requirements (uptime, no-log policy, ECH capable, etc.)
4. Curator adds node_id to pool and re-signs pool advertisement

Your node can be in multiple pools simultaneously.

```toml
[advertisement]
pool_memberships = ["pool-id-for-eu-exits", "pool-id-for-no-log-exits"]
```

Pool membership signals trust attributes to clients:
- **No-log policy**: Verified by pool curator
- **ECH capable**: Destination domain privacy
- **Jurisdiction**: Legal environment transparency
- **Uptime**: Reliability commitment

---

## 11. Mesh Gateway Operations

If also operating as a gateway for a mesh network:

```bash
# Check gateway bridge status
bgpx-cli gateway status

# Check DHT cross-domain sync
bgpx-cli gateway dht-sync-status

# View mesh peers reachable via gateway
bgpx-cli mesh peers
```

### 11.1 Gateway Reliability

Gateway uptime is more critical than relay uptime — mesh users depend on the gateway for clearnet access. Consider:

- Uninterruptible power supply (UPS)
- Redundant WAN connections (fiber + cellular)
- Automatic failover between WAN links
- Physical security for the gateway node

### 11.2 Cross-Domain Path Handling

As a mesh gateway (domain bridge node), your gateway:

- Bridges between mesh routing domain and clearnet domain
- Stores path_id entries in both single-domain and cross-domain path tables
- Forwards traffic between domains without re-encryption
- Does NOT know the client's mesh address when handling return traffic (path_id routing)

The gateway knows the destination but not the source. The mesh user's privacy is maintained because the gateway only sees:
- The clearnet destination
- That traffic came from a mesh path (not which mesh node)
- The path_id for return routing (not linked to identity)

---

## 12. Cross-Domain Context

Exit nodes in cross-domain paths work identically to single-domain exit nodes. The exit node does not know whether the path traversed mesh islands before reaching it.

**What the exit node sees in cross-domain paths**:
- Same as single-domain: destination, no source
- The path arriving from a domain bridge node (via clearnet relay or directly)
- No information about how many mesh hops preceded the bridge

**What the exit node does NOT see**:
- Whether mesh islands were involved
- The client's mesh address
- The number of mesh hops in the path

This is by design: cross-domain paths maintain the same privacy properties as single-domain paths from the exit node's perspective.

---

## 13. Satellite WAN Considerations

If your exit node uses satellite internet (Starlink, Iridium, Inmarsat, HughesNet, Viasat) as its WAN connection:

- **Domain**: Still clearnet (0x00000001), not a separate satellite domain
- **Latency class**: Configured as `satellite-leo` (20-60ms) or `satellite-geo` (600ms+)
- **Geo plausibility**: Satellite nodes are exempt from geographic plausibility RTT scoring
- **Performance expectations**: Higher RTT affects congestion signals and path quality reports

From BGP-X's protocol perspective, a satellite-connected exit node is a clearnet node with higher latency. The BGP-X overlay operates unchanged.

---

## 14. Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal for exit nodes:

- **If jurisdiction is declared**: geo plausibility RTT scoring applies
- **If jurisdiction is NOT declared**: geo plausibility scoring does NOT apply

Declaring jurisdiction is a personal/operational choice. The network functions identically with or without it.

**Why declare jurisdiction**:
- Users may prefer exit nodes in specific legal environments
- Signals transparency to privacy-conscious users
- Enables pool curation by jurisdiction

**Why NOT declare jurisdiction**:
- Reveals information about node location
- May enable unwanted attention from authorities
- Privacy-conscious operators may prefer anonymity

Satellite-connected exit nodes are **exempt** from geo plausibility scoring regardless of jurisdiction declaration, because satellite terminal IPs may be assigned from distant ground stations.

---

## 15. Operator Responsibilities

Exit node operators are responsible for:

- Publishing accurate exit policy
- Responding to abuse complaints via `operator_contact`
- Complying with local laws regarding traffic they handle
- NOT logging connection-level data (technical enforcement by `logging_policy = "none"`)
- Running up-to-date BGP-X software
- Maintaining adequate uptime (especially for gateways serving mesh communities)
- Understanding cross-domain implications if operating as a domain bridge

**BGP-X provides no legal protection.** Operators should consult legal advice for their jurisdiction.

---

## 16. Recommended Hardware for Exit Nodes

| Hardware | Use Case | Notes |
|---|---|---|
| BGP-X Router v1 (Outdoor) | Small community gateway | Full daemon, integrated radios, IP67 |
| BGP-X Node v1 | Community mesh gateway | Solar-powered option, outdoor enclosure |
| BGP-X Gateway v1 | Provider exit infrastructure | High throughput, SFP+ uplink, 1 GB RAM |
| GL.iNet GL-MT6000 + LoRa USB | Low-cost domain bridge | Third-party hardware, requires adapter |
| x86 server | Datacenter exit node | Maximum performance, high session capacity |

For high-throughput provider deployments, the BGP-X Gateway v1 with MT7988A quad-core processor and 1 GB RAM is recommended. For community mesh gateways, the BGP-X Node v1 with solar power option is ideal.

---

## 17. Checklist for New Exit Node Operators

Before announcing your exit node:

- [ ] Understand legal implications for your jurisdiction
- [ ] Configure minimal-risk or appropriate exit policy
- [ ] Enable DoH DNS with DNSSEC validation
- [ ] Enable ECH capability (`ech_capable = true`)
- [ ] Set `logging_policy = "none"`
- [ ] Configure `operator_contact` and `jurisdiction` (optional)
- [ ] Test exit functionality with known destinations
- [ ] Test ECH functionality
- [ ] Verify no prohibited data is logged
- [ ] Set up monitoring and health checks
- [ ] Prepare abuse response template
- [ ] Join relevant exit pools
- [ ] If operating as mesh gateway: configure domain bridge settings

---

## 18. References

- `/protocol/protocol_spec.md` — Exit policy wire format
- `/gateway/gateway_spec.md` — Gateway protocol specification
- `/hardware/router_spec.md` — BGP-X Router v1 hardware
- `/hardware/gateway_spec.md` — BGP-X Gateway v1 hardware
- `/docs/satellite_architecture.md` — Satellite WAN integration
