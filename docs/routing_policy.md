# BGP-X Routing Policy Engine

**Version**: 0.1.0-draft

This document specifies the BGP-X routing policy engine — the component that decides, for each packet or flow from a LAN device, whether it routes through the BGP-X overlay or via standard internet routing.

---

## 1. Overview

The routing policy engine is what makes dual-stack operation possible. On a router running both standard IP networking and BGP-X, every LAN packet passes through the policy engine for classification:

- **BGP-X overlay**: packet enters bgpx0 TUN interface; daemon encrypts and routes through overlay
- **Standard routing**: packet goes directly to WAN interface; unprotected standard internet
- **Block**: packet is dropped

Rules are evaluated in order. **First match wins.** If no rule matches, the default action applies.

The routing policy engine has no knowledge of what happens after classification. It does not see inside BGP-X paths, does not know which relay nodes are used, and does not log what it classifies.

---

## 2. Policy Rule Types

### 2.1 Device Rules

Match on the source device identity:

```toml
[[rules.device]]
# Match by source IP (single or subnet)
device_ip = "192.168.1.50"
action = "standard"
reason = "gaming_console"

[[rules.device]]
device_ip = "192.168.1.0/24"
action = "bgpx"

[[rules.device]]
# Match by MAC address
device_mac = "aa:bb:cc:dd:ee:ff"
action = "bgpx"
```

### 2.2 Application Rules (Transparent Proxy Mode)

Match on the originating process (requires iptables UID/GID matching):

```toml
[[rules.application]]
process_name = "firefox"
action = "bgpx"

[[rules.application]]
process_uid = 1001
action = "bgpx"

[[rules.application]]
process_name = "steam"
action = "standard"
```

### 2.3 Destination Rules

Match on the destination address:

```toml
[[rules.destination]]
# CIDR blocks
destination_cidr = ["192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12"]
action = "bypass"
reason = "local_network"

[[rules.destination]]
# Specific IP
destination_ip = "8.8.8.8"
action = "standard"

[[rules.destination]]
# Domain name (requires DNS interception or DoH proxy)
destination_domain = ["*.sensitive-site.com", "whistleblower.org"]
action = "bgpx"
path = { hops = 6, require_ech = true }
```

### 2.4 Protocol / Port Rules

Match on protocol or destination port:

```toml
[[rules.port]]
destination_port = [25]
protocol = "tcp"
action = "block"
reason = "no_outbound_smtp"

[[rules.port]]
destination_port = [7474]
protocol = "udp"
action = "bypass"
reason = "bgpx_overlay_traffic_cannot_self_tunnel"

[[rules.port]]
destination_port = [443]
action = "bgpx"
```

### 2.5 Pool-Specific Path Rules

For flows going through BGP-X, specify exact path constraints inline:

```toml
[[rules.destination]]
destination_domain = ["*.sensitive-site.com"]
action = "bgpx"
path = {
    segments = [
        { pool = "bgpx-default", hops = 2 },
        { pool = "trusted-relays", hops = 1 },
        { pool = "my-private-exit", hops = 1, exit = true }
    ],
    require_ech = true,
    exit_logging_policy = "none",
    exit_jurisdiction_blacklist = ["US", "GB", "AU", "CA", "NZ"]
}

[[rules.destination]]
destination_domain = ["streaming-service.com"]
action = "bgpx"
path = {
    hops = 4,
    exit_jurisdiction_whitelist = ["US"]  # Need US IP for geo-restricted content
}
```

### 2.6 Domain-Aware Routing Rules

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
match.destination_domain = ["*.mesh.lima-district-1.bgpx"]
action = "bgpx"

[rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:lima-district-1" },
    { type = "segment", domain = "mesh:lima-district-1", hops = 2 }
]
```

**URI Scheme Correction**: Use `.mesh.<island_id>.bgpx` format for mesh island services, not `bgpx-mesh://` URI schemes. The `.bgpx` domain system provides self-authenticating addresses that work across all routing domains.

### 2.7 Default Rule (Required)

The default rule applies to all traffic not matched by any previous rule. It must be the last rule:

```toml
[[rules.default]]
action = "bgpx"
# Everything else: through the overlay
```

Or for dual-stack where most traffic goes standard:

```toml
[[rules.default]]
action = "standard"
# Everything else: direct internet, only specific rules go through overlay
```

---

## 3. Rule Structure

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

## 4. Rule Evaluation Algorithm

Rules are evaluated in the order they appear in the configuration file. The first rule that matches is applied. Subsequent rules are not evaluated.

```
function route_packet(packet):
    for rule in ordered_rules:
        if rule.matches(packet):
            return rule.action

    return DEFAULT_ACTION
```

### 4.1 Match Evaluation Order (within a rule)

When a rule has multiple match criteria, ALL criteria must match (AND logic):

```toml
[[rules.destination]]
destination_domain = ["*.example.com"]
destination_port = [443]
protocol = "tcp"
action = "bgpx"
# Matches only if: domain ends in .example.com AND port 443 AND TCP
```

### 4.2 Multiple Values in a Field (OR Logic)

Multiple values within a single field are OR logic:

```toml
[[rules.destination]]
destination_cidr = ["10.0.0.0/8", "192.168.0.0/16"]
action = "bypass"
# Matches if: destination is in 10.0.0.0/8 OR in 192.168.0.0/16
```

---

## 5. Configuration File Format

Location: `/etc/bgpx/routing_policy.toml` (router install) or `~/.config/bgpx/routing_policy.toml` (user install)

### 5.1 Complete Format

```toml
# BGP-X Routing Policy Configuration
# Rules are evaluated in order — first match wins

[defaults]
# Default action for unmatched traffic
default_action = "bgpx"

# Default BGP-X path config for rules that don't specify path
default_path = { hops = 4, pool = "bgpx-default" }

# DNS routing
# overlay: resolve DNS for BGP-X traffic through overlay
# system: use system resolver (may leak DNS queries)
bgpx_dns = "overlay"
standard_dns = "system"

[rules]

# LAN and loopback always bypass (prevent routing recursion)
[[rules.bypass]]
destination_cidr = [
    "192.168.0.0/16",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fe80::/10",
    "fc00::/7"
]
action = "bypass"
reason = "local_network"

# BGP-X overlay traffic must bypass (prevent routing loop)
[[rules.bypass]]
destination_port = [7474]
protocol = "udp"
action = "bypass"
reason = "bgpx_overlay"

# High-security destinations: maximum privacy
[[rules.destination]]
destination_domain = [
    "*.whistleblower.example.com",
    "*.sensitive-banking.com"
]
action = "bgpx"
path = {
    hops = 6,
    segments = [
        { pool = "bgpx-default", hops = 3 },
        { pool = "my-private-exit", hops = 1, exit = true }
    ],
    require_ech = true,
    exit_logging_policy = "none",
    exit_jurisdiction_blacklist = ["US", "GB", "AU", "CA", "NZ"]
}

# Streaming: need US exit IP
[[rules.destination]]
destination_domain = ["*.netflix.com", "*.hulu.com", "*.hbomax.com"]
action = "bgpx"
path = {
    hops = 3,
    exit_jurisdiction_whitelist = ["US"]
}

# Gaming devices: bypass for low latency
[[rules.device]]
device_ip = ["192.168.1.100", "192.168.1.101"]
action = "standard"
reason = "gaming_devices"

# Smart TV: bypass (geo-locked content)
[[rules.device]]
device_mac = "aa:bb:cc:dd:ee:ff"
action = "standard"
reason = "smart_tv"

# Work laptop: always through overlay
[[rules.device]]
device_ip = "192.168.1.50"
action = "bgpx"
reason = "work_laptop"

# Block outbound SMTP (prevent spam relay)
[[rules.port]]
destination_port = [25]
protocol = "tcp"
action = "block"

# Default: everything else through BGP-X
[[rules.default]]
action = "bgpx"
```

---

## 6. DNS Routing Policy

DNS queries need special handling in dual-stack mode:

### 6.1 For BGP-X Traffic

DNS queries for destinations routed through BGP-X should also go through BGP-X, otherwise the DNS query leaks the destination to the ISP.

Configuration: `bgpx_dns = "overlay"` — BGP-X daemon runs a local DNS proxy on port 5300. dnsmasq routes BGP-X-destined queries through this proxy, which forwards via the overlay.

### 6.2 For Standard Traffic

DNS queries for traffic going directly to WAN can use the system resolver (ISP DNS) or a custom DNS-over-HTTPS provider.

Configuration: `standard_dns = "system"` or `standard_dns = "https://dns.quad9.net/dns-query"`

### 6.3 Domain-Based Routing

When `bgpx_dns = "overlay"`, the daemon can intercept DNS responses to apply domain-based routing rules. A DNS query for `example.com` that resolves to an IP in the bypass list causes that IP to be treated as bypassed.

### 6.4 Cross-Domain DNS Routing

Domain-based routing rules use DNS interception at the policy layer. The destination domain name is matched against patterns. When using cross-domain rules, domain names from mesh island services use the `.mesh.<island_id>.bgpx` format.

---

## 7. Dynamic Rule Updates

Routing policy rules can be updated without restarting the daemon:

```bash
# Add a new rule
bgpx-cli policy add \
    --destination sensitive-site.com \
    --action bgpx \
    --path-hops 6 \
    --require-ech

# Remove a rule by ID
bgpx-cli policy remove --rule-id 7

# Reload policy from file
bgpx-cli policy reload

# Test which rule matches a destination
bgpx-cli policy test --destination example.com --device 192.168.1.50

# List all rules in priority order
bgpx-cli policy list
```

---

## 8. Bypass Rules — Preventing Routing Loops

Bypass rules are critical for preventing routing recursion and self-tunneling. Every configuration must include these bypass rules:

### 8.1 Local Network Bypass

```toml
[[rules.bypass]]
destination_cidr = [
    "192.168.0.0/16",
    "10.0.0.0/8",
    "172.16.0.0/12",
    "127.0.0.0/8",
    "169.254.0.0/16"
]
action = "bypass"
reason = "local_network"
```

Without this rule, LAN traffic would be routed into the BGP-X overlay, potentially creating routing loops or preventing local network access.

### 8.2 BGP-X Overlay Traffic Bypass

```toml
[[rules.bypass]]
destination_port = [7474]
protocol = "udp"
action = "bypass"
reason = "bgpx_overlay"
```

Without this rule, BGP-X overlay traffic would be routed into itself, creating a routing loop. This is a mandatory rule for any BGP-X router.

---

## 9. Common Configurations

### 9.1 Configuration 1: All Traffic Through BGP-X

```toml
[defaults]
default_action = "bgpx"

[[rules.bypass]]
destination_cidr = ["192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12", "127.0.0.0/8"]
action = "bypass"

[[rules.bypass]]
destination_port = [7474]
protocol = "udp"
action = "bypass"

[[rules.default]]
action = "bgpx"
```

### 9.2 Configuration 2: Everything Through BGP-X Except Gaming

```toml
[defaults]
default_action = "bgpx"

[[rules.bypass]]
destination_cidr = ["192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12", "127.0.0.0/8"]
action = "bypass"

[[rules.bypass]]
destination_port = [7474]
protocol = "udp"
action = "bypass"

[[rules.device]]
device_ip = ["192.168.1.100", "192.168.1.101"]
action = "standard"
reason = "gaming_consoles"

[[rules.default]]
action = "bgpx"
```

### 9.3 Configuration 3: Work Security Policy

```toml
[defaults]
default_action = "standard"

[[rules.bypass]]
destination_cidr = ["192.168.0.0/16", "10.0.0.0/8"]
action = "bypass"

[[rules.bypass]]
destination_port = [7474]
protocol = "udp"
action = "bypass"

# Work laptop: all traffic through high-security path
[[rules.device]]
device_ip = "192.168.1.50"
action = "bgpx"
path = {
    hops = 5,
    exit_logging_policy = "none",
    exit_jurisdiction_blacklist = ["US", "GB", "AU", "CA", "NZ"],
    require_ech = true
}

# Family devices: standard internet
[[rules.default]]
action = "standard"
```

### 9.4 Configuration 4: Per-Destination with Different Pools

```toml
[[rules.destination]]
destination_domain = ["*.sensitive.com"]
action = "bgpx"
path = {
    segments = [
        { pool = "bgpx-default", hops = 2 },
        { pool = "my-private-exit", hops = 1, exit = true }
    ],
    require_ech = true
}

[[rules.destination]]
destination_domain = ["*.streaming.com"]
action = "bgpx"
path = { hops = 3, exit_jurisdiction_whitelist = ["US"] }

[[rules.default]]
action = "bgpx"
```

### 9.5 Configuration 5: Cross-Domain via Mesh Island

```toml
[[rules]]
id = "via-mesh-island"
priority = 5
match.destination_domain = ["*.ultra-sensitive.org"]
action = "bgpx"
reason = "Cross-domain via trusted mesh island"

[rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", pool = "bgpx-default", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 2 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "eu-nolog-exits", hops = 1, exit = true }
]
require_ech = true
```

### 9.6 Configuration 6: Satellite WAN Gateway

```toml
[[rules]]
id = "satellite-gateway-traffic"
priority = 10
match.source_device_ip = ["192.168.1.200"]  # Satellite gateway node
action = "bgpx"
reason = "Satellite WAN gateway — all traffic via overlay"

path_constraints = {
    hops = 3,
    pool = "satellite-aware-relays"
}
```

---

## 10. N-Hop Unlimited in Policy

Routing policy rules can specify any number of hops in any number of domain segments. The policy engine passes configuration to the daemon without modification. The daemon constructs the path without any hop count limit.

Example with extended hops and multiple domains:

```toml
[rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", hops = 5 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:island-a" },
    { type = "segment", domain = "mesh:island-a", hops = 4 },
    { type = "bridge", from_domain = "mesh:island-a", to_domain = "mesh:island-b" },
    { type = "segment", domain = "mesh:island-b", hops = 3 },
    { type = "bridge", from_domain = "mesh:island-b", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", hops = 2, exit = true }
]
# Total: 14 hops across 3 domains — valid
```

The daemon logs an informational message when total hops exceeds 15 (high latency expected) but does NOT reject or limit the path. N-hop unlimited is enforced at the protocol level.

---

## 11. Performance Considerations

The routing policy engine evaluates rules for every packet or flow. For high-throughput deployments:

- Keep rule count reasonable (under 50 for optimal performance)
- Put most-matched rules first (device bypass rules before per-destination rules)
- Use CIDR blocks instead of individual IPs where possible
- Domain-based rules require DNS query interception — add latency to first connection only (subsequent connections use cached classification)

Rule evaluation overhead: <100ns for typical rule sets on modern hardware.

---

## 12. Audit and Verification

Verify routing policy is working correctly:

```bash
# Test which rule matches a destination/device combination
bgpx-cli policy test --destination 8.8.8.8 --device 192.168.1.50

# View routing statistics (aggregate counts per rule, no identifiers)
bgpx-cli policy stats

# Audit for unexpected bypass (aggregate only)
bgpx-cli routing audit --window 1h

# Verify DNS routing
bgpx-cli dns test --query example.com --expected overlay
```

**Important**: routing policy statistics count packets per rule category. They do NOT log destination addresses, source IPs, or any identifying information.

---

## 13. Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an OPTIONAL feature in BGP-X:

- If a node declares a jurisdiction: geo plausibility scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply
- Nodes are NOT required to declare jurisdiction
- Declaring jurisdiction is an opt-in privacy/convenience tradeoff

Routing policy rules can specify geo-related constraints:

```toml
[rules.path_constraints]
exit_jurisdiction_blacklist = ["US", "GB"]  # Avoid these jurisdictions
exit_jurisdiction_whitelist = ["CH", "IS"]  # Prefer these jurisdictions
```

These constraints apply to exit node selection, not to intermediate relays.

---

## 14. Kill Switch

When BGP-X is active, optionally block all non-BGP-X traffic:

```toml
[routing_policy]
kill_switch = true              # Block all non-BGP-X WAN traffic
kill_switch_allow_dns = true    # Allow DNS to configured resolver
kill_switch_allow_local = true  # Allow LAN traffic
```

Kill switch prevents traffic leakage if the BGP-X daemon stops unexpectedly. With `kill_switch = true`:
- All traffic not explicitly matched to a bypass rule is blocked if the daemon is not running
- DNS queries are allowed if `kill_switch_allow_dns = true`
- LAN traffic is allowed if `kill_switch_allow_local = true`

---

## 15. HTTP/2 for .bgpx Services

BGP-X native services (.bgpx addresses) use HTTP/2 over BGP-X streams. This is transparent to routing policy — the policy engine only decides whether traffic goes through BGP-X or not. The HTTP version is handled by the daemon.

For routing policy purposes:
- Traffic to `.bgpx` domains is classified as BGP-X traffic
- The daemon handles HTTP/2 multiplexing automatically
- No special routing policy configuration needed for HTTP version

---

## 16. Hardware Considerations

When configuring routing policy on BGP-X Router v1, BGP-X Node v1, or BGP-X Gateway v1:

- **Router v1**: Full policy engine with cross-domain support
- **Node v1**: Full policy engine; domain bridge mode requires both WAN and mesh radio configured
- **Gateway v1**: Optimized for high-throughput; can handle complex rule sets at line rate
- **Client Node (Tier 2)**: No routing policy — endpoint only, not routing for others
- **Adapter (Tier 3)**: No routing policy — host computer handles all decisions

Domain bridge nodes require both clearnet (WAN) and mesh radio (WiFi 802.11s or LoRa) to be operational for cross-domain routing policies to function. If either interface is down, cross-domain rules may fail to construct paths.

---

## 17. Domain Bridge and Mesh Gateway Routing

Routing policies for cross-domain paths require domain bridge nodes or mesh gateways to be available:

```toml
# Cross-domain rule - requires domain bridge node
[[rules]]
id = "cross-domain-example"
match.destination_domain = ["*.remote-mesh-service.bgpx"]
action = "bgpx"

[rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:remote-island" },
    { type = "segment", domain = "mesh:remote-island", hops = 2 }
]
```

If no domain bridge node exists between clearnet and `mesh:remote-island`, this rule will fail to construct paths. The daemon logs an error and falls back to the default action.

For high-availability cross-domain routing:
- Deploy at least 2 domain bridge nodes from different operators
- Use pool-based bridge selection for redundancy
- Monitor bridge availability via `bgpx-cli domains bridges --health`
