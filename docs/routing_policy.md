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
