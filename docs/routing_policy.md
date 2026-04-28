# BGP-X Routing Policy Guide

**Version**: 0.1.0-draft

---

## 1. Overview

The routing policy engine evaluates each packet or flow and decides whether to route via the BGP-X overlay, bypass to standard routing, or block. Rules are evaluated in priority order.

---

## 2. Rule Structure

```toml
[[rules]]
id = "rule-unique-id"
priority = 10               # Lower = higher priority; 0 is highest
description = "Human description"

[rules.match]
# Match criteria — all specified must match (AND logic)
destination_cidr = ["10.0.0.0/8"]        # CIDR ranges
destination_domain = ["*.example.com"]    # Glob patterns
source_device_ip = ["192.168.1.50"]      # Source device on LAN
protocol = "tcp"                          # tcp, udp, or omit for any
destination_port = [80, 443]              # Port list

action = "bgpx"          # "bgpx", "standard", or "block"
reason = "Human reason for audit log"

[rules.path_constraints]
# Applied only when action = "bgpx"
hops = 4
exit_jurisdiction_blacklist = ["US", "GB"]
exit_jurisdiction_whitelist = []
exit_logging_policy = "none"
require_ech = false
min_reputation_score = 70.0
pool = "my-curated-pool"
```

---

## 3. Domain-Aware Routing Rules

Routing rules can specify cross-domain paths for specific destinations:

```toml
# Route via mesh island for extra isolation
[[rules]]
id = "high-sec-via-mesh"
priority = 5
match.destination_domain = ["*.sensitive.com"]
action = "bgpx"
reason = "High-security via mesh island"

[rules.path_constraints.domain_segments]
# clearnet → mesh island → clearnet exit
segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 2 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "private-exits", hops = 1, exit = true }
]
require_ech = true

# Route to mesh island service directly
[[rules]]
id = "mesh-service-direct"
priority = 3
match.destination_domain = ["bgpx-mesh://lima-district-1/*"]
action = "bgpx"

[rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:lima-district-1" },
    { type = "segment", domain = "mesh:lima-district-1", hops = 2 }
]
```

---

## 4. Complete Policy Example

```toml
# /etc/bgpx/routing_policy.toml

default_action = "bgpx"

# Block local network routing through overlay
[[rules]]
id = "bypass-local"
priority = 0
match.destination_cidr = [
    "192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12",
    "127.0.0.0/8", "169.254.0.0/16"
]
action = "standard"
reason = "Local network bypass"

# Block known malware C2 ranges
[[rules]]
id = "block-malware"
priority = 1
match.destination_cidr = ["198.51.100.0/24"]
action = "block"
reason = "Known malware C2"

# High-security traffic via mesh
[[rules]]
id = "high-sec-mesh"
priority = 5
match.destination_domain = ["*.journalsite.org", "*.privacytool.net"]
action = "bgpx"
path_constraints.domain_segments = [
    { type = "segment", domain = "clearnet", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 3 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "eu-nolog-exits", hops = 1, exit = true }
]
path_constraints.require_ech = true
reason = "High-security routing for sensitive destinations"

# Default: all traffic via BGP-X overlay
[[rules]]
id = "default-bgpx"
priority = 100
match.destination_domain = ["*"]
action = "bgpx"
path_constraints.exit_logging_policy = "none"
path_constraints.exit_jurisdiction_blacklist = ["US"]
reason = "Default BGP-X routing"
```

---

## 5. N-Hop Unlimited in Policy

Routing policy rules can specify any number of hops in any number of domain segments. The policy engine passes configuration to the daemon without modification. The daemon constructs the path without any hop count limit.

---

## 6. DNS Routing with Cross-Domain

Domain-based routing rules use DNS interception at the policy layer. The destination domain name is matched against patterns. When using cross-domain rules, domain names from mesh island services use the `bgpx-mesh://` URI scheme.

---

## 7. Policy Testing

```bash
# Test what rule matches a destination
bgpx-cli policy test --destination "sensitive.example.com" --protocol tcp --port 443
# Output: matched rule "high-sec-mesh", action "bgpx", domain_segments configured

# Test from specific device
bgpx-cli policy test --source-ip 192.168.1.50 --destination "10.0.0.1" --protocol tcp

# List all rules
bgpx-cli policy list
```

---

## 8. Kill Switch

When BGP-X is active, optionally block all non-BGP-X traffic:

```toml
[routing_policy]
kill_switch = true              # Block all non-BGP-X WAN traffic
kill_switch_allow_dns = true    # Allow DNS to configured resolver
kill_switch_allow_local = true  # Allow LAN traffic
```

Kill switch prevents traffic leakage if the BGP-X daemon stops unexpectedly.
