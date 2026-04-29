# BGP-X Control Client Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X control client (`bgpx-cli`) is the command-line interface to the daemon's Control API. It connects via the Control Socket and provides all daemon management capabilities.

bgpx-cli is NOT a routing stack. It is a management client only. All routing logic runs in the daemon.

---

## 2. Connection

```bash
# Default socket path
bgpx-cli [command]

# Custom socket path
bgpx-cli --socket /var/run/bgpx/control.sock [command]

# Remote daemon via SSH
ssh user@router "bgpx-cli status"
```

---

## 3. Command Reference

### Node Lifecycle

```bash
bgpx-cli status              # Node status and statistics
bgpx-cli shutdown [--timeout 30] [--no-withdraw]
bgpx-cli reload-config
```

### Path Management

```bash
bgpx-cli paths list
bgpx-cli paths build --segments "default:3,private-exits:1"
bgpx-cli paths build --domain-segments "clearnet:2,bridge:clearnet→mesh:island-1,mesh:island-1:2"
bgpx-cli paths destroy --index 0
bgpx-cli paths test --destination "example.com:443"
```

### Routing Policy

```bash
bgpx-cli policy list
bgpx-cli policy add --file /etc/bgpx/rules/rule.toml --position 5
bgpx-cli policy remove --rule-id rule-abc123
bgpx-cli policy reorder --rule-id rule-abc123 --position 2
bgpx-cli policy test --destination "example.com" --protocol tcp --port 443
bgpx-cli policy reload
bgpx-cli policy set-default --action bgpx
bgpx-cli policy stats
```

### Pool Management

```bash
bgpx-cli pools list [--domain "mesh:island-1"]
bgpx-cli pools get --pool-id hex...
bgpx-cli pools add --file pool.json
bgpx-cli pools remove --pool-id hex...
bgpx-cli pools verify --pool-id hex...
bgpx-cli pools refresh --pool-id hex...
```

### Node Database

```bash
bgpx-cli nodes list [--domain "mesh:island-1"] [--bridge-capable] [--bridge-to "clearnet"]
bgpx-cli nodes get --node-id hex...
bgpx-cli nodes blacklist --node-id hex... --reason "suspicious"
bgpx-cli nodes unblacklist --node-id hex...
bgpx-cli nodes list-blacklisted
```

### Domain Management

```bash
bgpx-cli domains list
bgpx-cli domains bridges [--from clearnet] [--to "mesh:island-1"] [--min-reputation 70]
bgpx-cli domains island-status --island-id island-name
bgpx-cli domains list-islands [--online-only]
bgpx-cli domains test --from clearnet --to "mesh:island-1"
bgpx-cli domains discover-bridges --from clearnet --to "mesh:island-1"
```

### Advertisement Management

```bash
bgpx-cli advert get
bgpx-cli advert republish
bgpx-cli advert update-metrics --bandwidth 200 --latency 8
bgpx-cli advert withdraw
```

### Session Management

```bash
bgpx-cli sessions list
bgpx-cli sessions rehandshake --all
```

### Statistics

```bash
bgpx-cli stats [--window 3600]
bgpx-cli stats reset
```

### Events

```bash
bgpx-cli events watch [--filter domain_bridge,mesh_island,cross_domain_path]
bgpx-cli events watch --filter all
```

### Configuration

```bash
bgpx-cli config get [--key path.to.key]
bgpx-cli config set --key sessions.max_sessions --value 5000
```

---

## 4. Output Formats

```bash
# Default: human-readable table
bgpx-cli nodes list

# JSON output
bgpx-cli --format json nodes list

# Quiet (exit code only)
bgpx-cli --quiet paths build --segments "default:4"
```

---

## 5. Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic error |
| 2 | Daemon not running or unreachable |
| 3 | Configuration error |
| 4 | Operation failed (see stderr) |
| 5 | Permission denied |

---

## 6. Shell Completion

```bash
# Generate shell completion scripts
bgpx-cli completions bash > /etc/bash_completion.d/bgpx-cli
bgpx-cli completions zsh > ~/.zsh/completions/_bgpx-cli
bgpx-cli completions fish > ~/.config/fish/completions/bgpx-cli.fish
```
