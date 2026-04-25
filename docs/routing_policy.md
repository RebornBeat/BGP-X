# BGP-X Routing Policy Engine

This document specifies the BGP-X routing policy engine — the component that decides, for each packet or flow from a LAN device, whether it routes through the BGP-X overlay or via standard internet routing.

---

## Overview

The routing policy engine is what makes dual-stack operation possible. On a router running both standard IP networking and BGP-X, every LAN packet passes through the policy engine for classification:

- **BGP-X overlay**: packet enters bgpx0 TUN interface; daemon encrypts and routes through overlay
- **Standard routing**: packet goes directly to WAN interface; unprotected standard internet
- **Block**: packet is dropped

Rules are evaluated in order. **First match wins.** If no rule matches, the default action applies.

The routing policy engine has no knowledge of what happens after classification. It does not see inside BGP-X paths, does not know which relay nodes are used, and does not log what it classifies.

---

## Policy Rule Types

### Device Rules

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

### Application Rules (Transparent Proxy Mode)

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

### Destination Rules

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

### Protocol / Port Rules

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

### Pool-Specific Path Rules

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

### Default Rule (Required)

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

## Rule Evaluation Algorithm

Rules are evaluated in the order they appear in the configuration file. The first rule that matches is applied. Subsequent rules are not evaluated.

```
function route_packet(packet):
    for rule in ordered_rules:
        if rule.matches(packet):
            return rule.action

    return DEFAULT_ACTION
```

### Match Evaluation Order (within a rule)

When a rule has multiple match criteria, ALL criteria must match (AND logic):

```toml
[[rules.destination]]
destination_domain = ["*.example.com"]
destination_port = [443]
protocol = "tcp"
action = "bgpx"
# Matches only if: domain ends in .example.com AND port 443 AND TCP
```

### Multiple Values in a Field (OR Logic)

Multiple values within a single field are OR logic:

```toml
[[rules.destination]]
destination_cidr = ["10.0.0.0/8", "192.168.0.0/16"]
action = "bypass"
# Matches if: destination is in 10.0.0.0/8 OR in 192.168.0.0/16
```

---

## Configuration File Format

Location: `/etc/bgpx/routing_policy.toml` (router install) or `~/.config/bgpx/routing_policy.toml` (user install)

### Complete Format

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

## DNS Routing Policy

DNS queries need special handling in dual-stack mode:

### For BGP-X Traffic

DNS queries for destinations routed through BGP-X should also go through BGP-X, otherwise the DNS query leaks the destination to the ISP.

Configuration: `bgpx_dns = "overlay"` — BGP-X daemon runs a local DNS proxy on port 5300. dnsmasq routes BGP-X-destined queries through this proxy, which forwards via the overlay.

### For Standard Traffic

DNS queries for traffic going directly to WAN can use the system resolver (ISP DNS) or a custom DNS-over-HTTPS provider.

Configuration: `standard_dns = "system"` or `standard_dns = "https://dns.quad9.net/dns-query"`

### Domain-Based Routing

When `bgpx_dns = "overlay"`, the daemon can intercept DNS responses to apply domain-based routing rules. A DNS query for `example.com` that resolves to an IP in the bypass list causes that IP to be treated as bypassed.

---

## Dynamic Rule Updates

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

## Common Configurations

### Configuration 1: All Traffic Through BGP-X

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

### Configuration 2: Everything Through BGP-X Except Gaming

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

### Configuration 3: Work Security Policy

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

### Configuration 4: Per-Destination with Different Pools

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

---

## Performance Considerations

The routing policy engine evaluates rules for every packet or flow. For high-throughput deployments:

- Keep rule count reasonable (under 50 for optimal performance)
- Put most-matched rules first (device bypass rules before per-destination rules)
- Use CIDR blocks instead of individual IPs where possible
- Domain-based rules require DNS query interception — add latency to first connection only (subsequent connections use cached classification)

Rule evaluation overhead: <100ns for typical rule sets on modern hardware.

---

## Audit and Verification

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
