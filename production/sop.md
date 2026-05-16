# BGP-X Production Standard Operating Procedures

**Version**: 0.1.0-draft
**License**: MIT (Code), CERN-OHL-S v2 (Hardware), CC BY 4.0 (Documentation)

---

## 1. Overview

This document defines the standard operating procedures (SOPs) for BGP-X production node operators. Procedures cover provisioning, monitoring, incident response, maintenance, and cross-domain-specific operations.

All operators MUST be familiar with this document before operating any BGP-X production node type. This specification applies to all hardware tiers: BGP-X Router v1, BGP-X Node v1, BGP-X Gateway v1, and BGP-X Client Nodes.

---

## 2. Node Classification

| Tier | Node Type | SLA | Uptime Target |
|---|---|---|---|
| 1 | Bootstrap node | Maximum | 99.9% |
| 2 | Public clearnet relay | High | 99.5% |
| 2 | Public exit/gateway node | High | 99.5% |
| 3 | Domain bridge node | High | 99.5% |
| 3 | Mesh island gateway | High | 99.0% |
| 4 | Community relay | Standard | 99.0% |
| 5 | Volunteer node | Best-effort | 95.0% |

Tier 3 nodes carry additional operational responsibility: bridge node downtime causes cross-domain connectivity loss for all users routing through that bridge.

---

## 3. Pre-Deployment Checklist

### 3.1 All Node Types

- [ ] Hardware verified against `/hardware/hardware_spec.md` minimum specifications for target tier
- [ ] Operating system: Ubuntu 24.04 LTS or Debian 12 (server edition) for Linux nodes; OpenWrt 23.05+ for embedded
- [ ] Kernel: 5.15+ with appropriate network stack tuning
- [ ] BGP-X daemon version pinned to current stable release
- [ ] BGP-X daemon hash verified against published signature
- [ ] Static IP address configured and confirmed (or stable dynamic DNS for non-infrastructure nodes)
- [ ] UDP/7474 inbound and outbound confirmed open (clearnet domain)
- [ ] Ed25519 node keypair generated and backed up securely (NEVER in git)
- [ ] `config.toml` reviewed against security defaults
- [ ] Firewall configured (see Section 5)
- [ ] System time synchronized via NTP (chrony preferred; max offset <500ms required)
- [ ] Swap disabled or encrypted (key material must not be swapped to disk)
- [ ] Automatic security updates enabled
- [ ] Remote access secured (SSH key-only, no password auth, no root SSH)
- [ ] Monitoring agent installed and verified reporting
- [ ] Logrotate configured for BGP-X logs (30-day retention, no identifiers logged)

### 3.2 Exit/Gateway Nodes (Tier 2)

- [ ] Exit policy reviewed and signed
- [ ] Exit policy uploaded to unified DHT and verified accessible
- [ ] DNS resolver configured (DoH with DNSSEC + ECS stripping)
- [ ] ECH certificate verification tested with test connection
- [ ] Connection limit configured per `config.toml`
- [ ] Bandwidth cap configured per agreed allocation

### 3.3 Domain Bridge Nodes (Tier 3)

- [ ] All routing domains configured in `config.toml` `[[routing_domains]]` sections
- [ ] `bridge_capable = true` verified in node advertisement via `bgpx-cli node show`
- [ ] Each domain's transport confirmed reachable:
  - [ ] Clearnet: UDP/7474 inbound confirmed
  - [ ] WiFi mesh (if applicable): 802.11s interface confirmed up
  - [ ] LoRa (if applicable): serial interface `/dev/ttyUSB0` (or equivalent) confirmed active
  - [ ] Satellite (if applicable): modem interface confirmed up (satellite internet = clearnet domain)
- [ ] DOMAIN_ADVERTISE record confirmed in unified DHT: `bgpx-cli domains bridges`
- [ ] Cross-domain path_id table initialized: `bgpx-cli node stats | grep cross_domain`
- [ ] Bridge pair advertisement TTL: `bgpx-cli domains bridges | grep ttl` (expect 24h)
- [ ] If also serving as mesh island gateway: MESH_ISLAND_ADVERTISE confirmed in DHT

### 3.4 Mesh Island Gateway Nodes

- [ ] Island ID confirmed unique: `bgpx-cli islands list --global`
- [ ] Island domain ID derived correctly from island_id: verify with `bgpx-cli domains show <island_id>`
- [ ] MESH_ISLAND_ADVERTISE published to unified DHT: `bgpx-cli islands list --dht`
- [ ] At least one other bridge node for this island confirmed operational (avoid single-bridge failure)
- [ ] Intra-island DHT bootstrapped: `bgpx-cli node dht-stats` shows ≥5 routing table entries
- [ ] Store-and-forward configuration reviewed for offline behavior

---

## 4. Initial Provisioning

### 4.1 Key Generation

```bash
# Generate Ed25519 node keypair
bgpx-keygen --output /etc/bgpx/node.key

# Verify keypair and derive NodeID
bgpx-cli node identity

# Back up private key (REQUIRED before going online)
# Store in: hardware security module, encrypted offline backup, or operator vault
# NEVER commit to version control
```

### 4.2 Configuration Validation

```bash
# Validate config.toml syntax and semantics
bgpx-node --config /etc/bgpx/config.toml --validate-only

# Test DHT connectivity with bootstrap nodes
bgpx-node --config /etc/bgpx/config.toml --dht-test-only

# For bridge nodes: test each domain transport
bgpx-node --config /etc/bgpx/config.toml --test-domains
```

### 4.3 First Advertisement

After daemon starts successfully:

```bash
# Verify advertisement published to DHT
bgpx-cli node show --advertisement

# For bridge nodes: verify domain advertisement
bgpx-cli domains bridges --self

# For mesh island gateways: verify island advertisement
bgpx-cli islands show --self

# Verify node is reachable by others (requires network connectivity)
bgpx-cli node ping <your_node_id>
```

---

## 5. Firewall Configuration

### 5.1 Standard Relay Node

```bash
# Allow BGP-X UDP
ufw allow 7474/udp comment "BGP-X relay"

# Allow SSH (restrict to management IP if possible)
ufw allow from <management_ip> to any port 22

# Deny all other inbound
ufw default deny incoming
ufw default allow outgoing
ufw enable
```

### 5.2 Domain Bridge Node (Additional Rules)

```bash
# BGP-X clearnet domain (same as standard)
ufw allow 7474/udp comment "BGP-X clearnet domain"

# WiFi mesh (if applicable) — no additional firewall rules
# (handled at interface level via 802.11s)

# LoRa (if applicable) — no additional firewall rules
# (handled at serial device level, not network)

# Note: BGP-X mesh transport communication is at L2 radio layer
# Firewall rules apply only to IP-based clearnet endpoints
```

### 5.3 Exit/Gateway Node (Additional Rules)

```bash
# Allow outbound to internet (for exit function — already default-allow)
# Restrict inbound strictly — exit nodes accept only BGP-X relay traffic on 7474

# No web server, no SSH from internet, no management ports exposed
# All management via VPN or jump host
```

---

## 6. Monitoring

### 6.1 Key Metrics — All Nodes

| Metric | Warning Threshold | Critical Threshold | Check Interval |
|---|---|---|---|
| Daemon process up | N/A | Not running | 30 seconds |
| UDP/7474 socket responding | N/A | Not responding | 1 minute |
| Session count | <5 active | 0 active (tier 1/2) | 5 minutes |
| Session count | >9,500 | >9,800 | 5 minutes |
| Packet decryption failure rate | >1% | >5% | 1 minute |
| Path table size | >80,000 entries | >90,000 entries | 5 minutes |
| NTP offset | >100ms | >500ms | 10 minutes |
| CPU usage | >80% sustained 5min | >95% sustained 1min | 1 minute |
| Memory usage | >80% | >90% | 1 minute |
| Disk usage | >80% | >90% | 5 minutes |

### 6.2 Key Metrics — Domain Bridge Nodes

| Metric | Warning Threshold | Critical Threshold | Check Interval |
|---|---|---|---|
| Cross-domain path table size | >80,000 | >90,000 | 5 minutes |
| Bridge transitions/min | 0 for >10min (if serving active paths) | 0 for >30min | 5 minutes |
| Bridge pair availability (each pair) | N/A | Unavailable >5min | 2 minutes |
| Target domain transport up | N/A | Not responding | 2 minutes |
| DOMAIN_ADVERTISE DHT record age | >20 hours | >23 hours | 30 minutes |

### 6.3 Key Metrics — Mesh Island Gateway

| Metric | Warning Threshold | Critical Threshold | Check Interval |
|---|---|---|---|
| Mesh island reachable | N/A | Not reachable >5min | 2 minutes |
| MESH_ISLAND_ADVERTISE DHT age | >20 hours | >23 hours | 30 minutes |
| Mesh relay count in island | <3 | <2 | 15 minutes |
| Bridge node count for island | 1 only | 0 (self down) | 5 minutes |
| WiFi/LoRa interface up (mesh) | N/A | Interface down | 1 minute |
| Internet connectivity (WAN) | Degraded | Down | 1 minute |

### 6.4 Monitoring Alerting Rules

**Page immediately (PagerDuty/equivalent)**:
- Daemon not running
- UDP/7474 not responding
- NTP offset >500ms
- Critical path table overflow
- Bridge pair completely unavailable >5min (Tier 3)
- Mesh island offline >5min AND no other bridge operational (Tier 3)

**Slack/email alert**:
- Warning thresholds exceeded
- DOMAIN_ADVERTISE approaching expiry
- MESH_ISLAND_ADVERTISE approaching expiry
- Decryption failure rate elevated

### 6.5 Automated Health Check Script

```bash
#!/bin/bash
# /usr/local/bin/bgpx-healthcheck.sh
# Run daily via cron: 0 7 * * * root /usr/local/bin/bgpx-healthcheck.sh

CONTROL_SOCKET="/var/run/bgpx/control.sock"
LOG_FILE="/var/log/bgpx/healthcheck.log"
ALERT_EMAIL="ops@yourdomain.com"

check_daemon() {
    if ! systemctl is-active --quiet bgpx-node; then
        echo "CRITICAL: bgpx-node daemon not running"
        systemctl start bgpx-node && echo "Restarted daemon"
        return 1
    fi
}

check_dht() {
    # Unified DHT check (single instance for all domains)
    DHT_SIZE=$(bgpx-cli dht stats | grep routing_table_size | awk '{print $2}')
    if [ "${DHT_SIZE:-0}" -lt 10 ]; then
        echo "WARNING: DHT routing table small (${DHT_SIZE} entries)"
        return 1
    fi
}

check_advertisement() {
    ADV_EXPIRES=$(bgpx-cli advertisement show | grep expires_at | cut -d'"' -f2)
    EXPIRES_EPOCH=$(date -d "${ADV_EXPIRES}" +%s 2>/dev/null)
    NOW=$(date +%s)
    HOURS_LEFT=$(( (EXPIRES_EPOCH - NOW) / 3600 ))

    if [ "${HOURS_LEFT}" -lt 6 ]; then
        echo "WARNING: Advertisement expires in ${HOURS_LEFT}h — republishing"
        bgpx-cli advertisement republish
    fi
}

check_pool_health() {
    bgpx-cli pools list 2>/dev/null | while read pool_id; do
        ACTIVE=$(bgpx-cli pools get --pool-id "$pool_id" | grep active_count | awk '{print $2}')
        if [ "${ACTIVE:-0}" -lt 3 ]; then
            echo "WARNING: Pool ${pool_id} has only ${ACTIVE} active members"
        fi
    done
}

check_ntp() {
    OFFSET=$(chronyc tracking | grep "RMS offset" | awk '{print $NF}')
    echo "NTP offset: ${OFFSET}"
}

# Run all checks
{
    date
    check_daemon
    check_dht
    check_advertisement
    check_pool_health
    check_ntp
    bgpx-cli stats --window 24h
} >> "${LOG_FILE}" 2>&1
```

---

## 7. Routine Maintenance

### 7.1 Daily

```bash
# Check daemon status
systemctl status bgpx-node

# Check active sessions (count only — no identifiers)
bgpx-cli node stats | grep session_count

# For bridge nodes: check bridge pair health
bgpx-cli domains bridges --health

# For mesh island gateways: check island reachability
bgpx-cli islands health <island_id>

# Review logs for errors (no identifiers should appear)
journalctl -u bgpx-node --since yesterday | grep -E "ERROR|WARN"
```

### 7.2 Weekly

```bash
# Verify node advertisement in DHT (freshness check)
bgpx-cli node show --advertisement --dht-freshness

# For bridge nodes: verify domain bridge advertisements
bgpx-cli domains bridges --self --dht-freshness

# For mesh island gateways: verify island advertisement
bgpx-cli islands show --self --dht-freshness

# Verify no cross-domain path table overflow occurred
bgpx-cli node stats | grep cross_domain_path_table_size

# Check reputation score (should remain above 0.5)
bgpx-cli reputation show --self

# Review security updates
unattended-upgrades --dry-run
apt list --upgradable 2>/dev/null | grep bgpx
```

### 7.3 Monthly

```bash
# Verify Ed25519 keypair backup is accessible and valid
bgpx-keytool verify --key /etc/bgpx/node.key

# Review exit policy (exit nodes): confirm still matches operational intent
bgpx-cli exit-policy show

# For bridge nodes: review bridge latency vs. advertised values
bgpx-cli domains bridges --latency-report

# Audit log retention policy: ensure no prohibited data in logs
grep -r "session_id\|node_id\|path_id\|ip_addr" /var/log/bgpx/ | wc -l
# Expected: 0

# Security patch review: check CVE feeds for dependencies
cargo audit --file /usr/share/bgpx/Cargo.lock
```

---

## 8. Incident Response

### 8.1 Severity Levels

| P0 | Complete service disruption; security breach; active deanonymization |
| P1 | Significant degradation; bridge node completely offline; island isolated |
| P2 | Partial degradation; elevated error rates; bridge pair latency exceeded 3x |
| P3 | Minor issues; monitoring gap; configuration drift |

### 8.2 P0 Response Procedure

```
T+0:    Page on-call operator
T+5:    Assess situation; classify P0 or downgrade
T+10:   Isolate node from network if active exploitation suspected:
          systemctl stop bgpx-node
          ufw deny 7474/udp
T+15:   Preserve forensic state:
          journalctl -u bgpx-node > /tmp/incident-$(date +%s).log
          cp /var/log/bgpx/ /tmp/incident-logs/
T+30:   Notify BGP-X security team via SECURITY.md contact
T+60:   Determine if key material may be compromised
T+120:  If key compromise suspected: rotate keypair
          bgpx-keygen --output /etc/bgpx/node.key.new
          # Update config, restart daemon, publish withdrawal for old key
```

**If active deanonymization suspected**: STOP ALL TRAFFIC IMMEDIATELY. Issue NODE_WITHDRAW for old key. Do not resume service until root cause confirmed and resolved.

### 8.3 P1 Response — Domain Bridge Node Offline

```
T+0:    Alert fires: bridge pair unavailable >5min
T+2:    Verify both domain transport endpoints:
          bgpx-cli domains bridges --health
          # Check clearnet UDP: nc -zu <your_ip> 7474
          # Check mesh transport: ip link show <mesh_interface>
          # Check LoRa serial: ls -la /dev/ttyUSB0
T+5:    If clearnet down: check WAN interface, ISP status
T+5:    If mesh transport down:
          # WiFi: systemctl restart hostapd; iw <interface> mesh join <mesh_id>
          # LoRa: check serial connection; reload LoRa driver
T+10:   Once transport restored:
          systemctl restart bgpx-node
          bgpx-cli domains bridges --health
T+15:   Verify DOMAIN_ADVERTISE re-published:
          bgpx-cli domains bridges --self --dht-freshness
T+20:   Verify cross-domain paths routing correctly (test from external client)
```

### 8.4 P1 Response — Mesh Island Isolated (All Bridge Nodes Offline)

```
T+0:    Alert: all bridge nodes for island offline (or mesh island unreachable)
T+2:    Verify island DHT advertisement freshness:
          bgpx-cli islands show <island_id> --dht-freshness
T+5:    Check other bridge operators (if multi-operator island):
          Contact operators for other bridge nodes via out-of-band channel
T+10:   Assess whether isolation is temporary (WAN outage) or structural
T+15:   If WAN outage: monitor for recovery; intra-island traffic continues normally
T+30:   If WAN restored: verify MESH_ISLAND_ADVERTISE re-published to unified DHT
          bgpx-cli islands publish <island_id>
T+45:   Verify clearnet clients can discover island again:
          bgpx-cli islands show <island_id> --dht-global
```

**Emergency island isolation** (mesh island experiencing internal security incident or rogue node activity):

```bash
# Remove island advertisement from unified DHT:
bgpx-cli islands withdraw <island_id>

# This prevents new cross-domain paths to the island
# Existing paths will time out via KEEPALIVE timeout
# Intra-island traffic continues (cannot be stopped from bridge node)

# After resolving internal issue:
bgpx-cli islands publish <island_id>
```

### 8.5 P2 Response — Bridge Latency Exceeded

If monitoring shows bridge transition latency consistently exceeds 200% of advertised value:

```bash
# Check which bridge pair is degraded
bgpx-cli domains bridges --latency-report

# Check target domain transport quality
bgpx-cli node ping <mesh_relay_node_id> --domain mesh:<island_id>

# If LoRa segment is congested: check duty cycle remaining
bgpx-cli node stats | grep lora_duty_cycle

# Reputation system will record BridgeLatencyExceeded events automatically
# If sustained: update advertised bridge_latency_ms in config and republish
```

### 8.6 Legal Request Response

**Trigger**: Law enforcement inquiry, subpoena, court order, or legal demand

**Immediate actions**:

1. **Do not panic** and do not immediately comply without legal advice
2. **Do not** delete any data (even if operating a no-log policy, deleting data after receiving a legal request may be obstruction)
3. **Document** the request (date, agency, contact, content)
4. **Contact a lawyer** with internet law expertise in your jurisdiction before taking any further action
5. **Notify BGP-X project** if you are comfortable doing so (security@bgpx.network) — this helps the project understand legal pressure patterns

**If operating a no-log policy**:

You can truthfully state: "We do not retain connection records, access logs, or any information about which users accessed which destinations. We are technically unable to produce records we do not have."

Prepare this statement in advance with your lawyer.

---

## 9. Configuration Updates

### 9.1 Live Configuration Changes (No Restart)

These changes apply immediately via Control API:

```bash
# Add routing policy rule
bgpx-cli policy add --rule '{"type":"cross_domain","destination":"192.168.0.0/16","action":"block"}'

# Blacklist a node
bgpx-cli nodes blacklist <node_id> --reason "protocol violation"

# Update bridge availability (e.g., planned mesh maintenance)
bgpx-cli domains bridge-availability --from clearnet --to mesh:<island_id> --available false
# Perform maintenance
bgpx-cli domains bridge-availability --from clearnet --to mesh:<island_id> --available true
```

### 9.2 Configuration Changes Requiring Daemon Restart

- `[[routing_domains]]` changes (adding/removing domains)
- `bridge_capable` flag changes
- Port number changes
- Keypair changes
- Mesh transport interface changes

```bash
# Always validate config before restart
bgpx-node --config /etc/bgpx/config.toml --validate-only

# Graceful restart (waits for active sessions to complete, up to grace_period)
systemctl reload bgpx-node

# Hard restart if graceful not available
systemctl restart bgpx-node

# Verify startup and advertisement
bgpx-cli node show --advertisement
bgpx-cli domains bridges --self  # for bridge nodes
```

---

## 10. Key Rotation Procedures

### 10.1 Scheduled Keypair Rotation

Recommended: annually, or when security audit recommends it.

```bash
# Generate new keypair
bgpx-keygen --output /etc/bgpx/node.key.new

# Update config.toml to reference new key
# (Do NOT restart yet)

# Publish NODE_WITHDRAW for old key (signed with old key)
bgpx-cli node withdraw --old-key /etc/bgpx/node.key

# Wait for withdrawal propagation (minimum 10 minutes)
sleep 600

# Switch to new key
mv /etc/bgpx/node.key /etc/bgpx/node.key.old
mv /etc/bgpx/node.key.new /etc/bgpx/node.key

# Restart daemon
systemctl restart bgpx-node

# Verify new NodeID
bgpx-cli node identity

# Delete old key from disk (retain secure offline backup)
shred -u /etc/bgpx/node.key.old
```

### 10.2 Emergency Keypair Rotation (Suspected Compromise)

```bash
# STOP SERVICE IMMEDIATELY
systemctl stop bgpx-node
ufw deny 7474/udp

# Generate new keypair
bgpx-keygen --output /etc/bgpx/node.key.emergency

# DO NOT re-use old key for withdrawal if key is compromised
# Withdrawal signed with compromised key will be ignored by reputation system anyway

# Update config and restart with new key
# Update config.toml: node_key_path = "/etc/bgpx/node.key.emergency"
mv /etc/bgpx/node.key.emergency /etc/bgpx/node.key

# Re-enable and restart
ufw allow 7474/udp
systemctl start bgpx-node

# Notify security team
# See SECURITY.md for contact information
```

---

## 11. Log Management

### 11.1 Permitted Log Content

```
# Permitted (operational): level, timestamp, daemon-internal events
2026-01-15T14:23:01Z INFO  bgpx-node: Daemon started, node_id=a3f2b9c1...
2026-01-15T14:23:05Z INFO  bgpx-node: DHT bootstrap complete, routing_table_size=20
2026-01-15T14:23:07Z INFO  bgpx-node: Session table capacity: 0/10000
2026-01-15T14:30:00Z WARN  bgpx-node: High decryption failure rate: 2.1% (threshold: 1%)
2026-01-15T14:45:00Z INFO  bgpx-node: Domain bridge clearnet→mesh:lima online
2026-01-15T16:00:00Z INFO  bgpx-node: DOMAIN_ADVERTISE republished to DHT
2026-01-15T16:00:00Z INFO  bgpx-node: Mesh island lima-district-1 reachable, relay_count=12
```

### 11.2 PROHIBITED Log Content (NEVER Log)

```
# NEVER log:
# - IP addresses of clients or relays
# - Session IDs
# - path_id values
# - Which routing domains a session traversed
# - Cross-domain traversal details (source domain, target domain, path composition)
# - Mesh island identifiers associated with specific sessions
# - Timing of individual connections
# - Destination addresses in clearnet exit context
# - Stream contents at any hop
# - node_ids associated with sessions
```

### 11.3 Log Rotation

```ini
# /etc/logrotate.d/bgpx
/var/log/bgpx/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    postrotate
        systemctl reload bgpx-node
    endscript
}
```

---

## 12. Backup and Recovery

### 12.1 What to Back Up

| File | Criticality | Recovery Without |
|---|---|---|
| `/etc/bgpx/node_private_key` | CRITICAL — NOT recoverable | New NodeID; lose all reputation |
| `/etc/bgpx/node.toml` | High | Reconfiguration required |
| `/etc/bgpx/curator_key` (if curator) | CRITICAL | Pool becomes unmanageable |
| `/var/lib/bgpx/reputation.db` | Medium | Reputation rebuilt from network |
| `/var/lib/bgpx/dht_routing_table.db` | Low | Repopulated from bootstrap |
| `/var/lib/bgpx/cross_domain_path_table.db` | Low | Rebuilt as traffic arrives |

### 12.2 Backup Procedure

```bash
#!/bin/bash
# /usr/local/bin/bgpx-backup.sh — run weekly

BACKUP_DIR="/backup/bgpx/$(date +%Y%m%d)"
mkdir -p "${BACKUP_DIR}"

# Node private key (ENCRYPTED)
gpg --symmetric --cipher-algo AES256 \
    --output "${BACKUP_DIR}/node_key.gpg" \
    /etc/bgpx/node_private_key

# Configuration
cp /etc/bgpx/node.toml "${BACKUP_DIR}/node.toml"

# Reputation database
cp /var/lib/bgpx/reputation.db "${BACKUP_DIR}/reputation.db"

# Cross-domain path table (if bridge node)
cp /var/lib/bgpx/cross_domain_path_table.db "${BACKUP_DIR}/cross_domain_path_table.db"

# Verify
gpg --decrypt "${BACKUP_DIR}/node_key.gpg" | sha256sum
echo "Backup complete: ${BACKUP_DIR}"

# Keep 4 weeks of backups
find /backup/bgpx -maxdepth 1 -type d -mtime +28 -exec rm -rf {} +
```

### 12.3 Recovery from Backup

```bash
# Stop daemon
systemctl stop bgpx-node

# Restore private key
gpg --decrypt /backup/bgpx/YYYYMMDD/node_key.gpg \
    > /etc/bgpx/node_private_key

# Restore configuration
cp /backup/bgpx/YYYYMMDD/node.toml /etc/bgpx/node.toml

# Restore cross-domain path table (if bridge node)
cp /backup/bgpx/YYYYMMDD/cross_domain_path_table.db /var/lib/bgpx/

# Fix permissions
chown bgpx:bgpx /etc/bgpx/node_private_key /etc/bgpx/node.toml
chmod 600 /etc/bgpx/node_private_key
chmod 640 /etc/bgpx/node.toml

# Restart
systemctl start bgpx-node
bgpx-cli status
```

---

## 13. Exit Node Specific Procedures

### 13.1 Exit Policy Updates

```bash
# Update exit policy file
vim /etc/bgpx/exit_policy.toml

# Sign new policy version
bgpx-exitpolicy sign --key /etc/bgpx/node.key --policy /etc/bgpx/exit_policy.toml

# Verify policy
bgpx-exitpolicy verify --policy /etc/bgpx/exit_policy.toml

# Reload (no restart needed for policy updates)
bgpx-cli exit-policy reload

# Verify clients see updated version
bgpx-cli exit-policy show --version
```

### 13.2 Responding to Abuse Complaints

BGP-X exit nodes technically exit traffic to the internet on behalf of anonymous users. Operators should understand their legal obligations in their jurisdiction.

```
Standard response to requests for user logs:
"This server operates a BGP-X exit node. No session logs, connection logs,
or identifying information about any user are retained. The technical
architecture prevents the operator from determining the identity of users
whose traffic passed through this server."

Refer any additional questions to legal counsel.
```

Operators MUST NOT reconfigure the daemon to enable prohibited logging under legal pressure. If required to do so: take the node offline and notify the BGP-X security team.

---

## 14. Hardware Failure Procedures

### 14.1 Storage Failure

If storage fails and cannot be recovered:

1. Key material (Ed25519 keypair) is LOST — restore from secure offline backup
2. Configuration — restore from version-controlled config (key paths excluded from git)
3. DHT routing table — rebuilt automatically on next start
4. Session state — all active sessions lost; clients rebuild paths automatically
5. Path tables (single-domain and cross-domain) — rebuilt as new traffic arrives
6. Reputation data — lost; node starts fresh

**Cross-domain path table loss**: clients will experience path rebuild events. This is operationally normal. No privacy impact.

### 14.2 Hardware Replacement (Bridge Node)

When replacing hardware on a bridge node:

```bash
# On new hardware:
# 1. Install BGP-X and restore keypair from secure backup
# 2. Restore config.toml (excluding key paths — already handled)
# 3. Verify ALL domain transports are operational on new hardware:
bgpx-node --config /etc/bgpx/config.toml --test-domains

# 4. Start daemon
systemctl start bgpx-node

# 5. Verify DOMAIN_ADVERTISE published
bgpx-cli domains bridges --self

# 6. Verify MESH_ISLAND_ADVERTISE if applicable
bgpx-cli islands show --self

# 7. Update DNS A record if WAN IP changed
```

---

## 15. Maintenance Windows

### Scheduled maintenance

For planned maintenance (upgrades, hardware changes):

```bash
# 1. Announce on BGP-X operators mailing list (24h advance for non-critical)

# 2. Enter drain mode (stop accepting new sessions, wait for existing to close)
# BGP-X daemon automatically drains on SIGTERM
# The drain timeout is configurable (default 30s)

# 3. Perform maintenance

# 4. Restart and verify

# 5. Update mailing list with completion
```

Maintenance windows longer than 1 hour will cause the node to be deprioritized in path selection due to KEEPALIVE timeouts on existing sessions. This is normal behavior — the reputation system will recover within 24 hours of the node returning to full operation.

---

## 16. Pool Curator Key Compromise Response

If a pool curator key is compromised:

```bash
# 1. STOP using compromised key immediately
# Do not sign any new records with the compromised key

# 2. Generate new keypair
bgpx-keygen --output /etc/bgpx/curator_key_new

# 3. Perform emergency rotation (2-hour window)
bgpx-cli pools rotate-key \
    --pool-id <pool-id> \
    --old-key /etc/bgpx/curator_key_compromised \
    --new-key /etc/bgpx/curator_key_new \
    --reason compromise_confirmed \
    --accept-window 2h

# 4. Re-sign all members IMMEDIATELY
bgpx-cli pools resign-members \
    --pool-id <pool-id> \
    --key /etc/bgpx/curator_key_new

# 5. Re-publish pool advertisement
bgpx-cli pools publish --pool-id <pool-id>

# 6. Destroy compromised key
shred -u /etc/bgpx/curator_key_compromised

# 7. Notify BGP-X project of the compromise
# Email: security@bgpx.network
