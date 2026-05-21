# Hosting a .bgpx Service

**Version**: 0.1.0-draft

---

## 1. What is a .bgpx Service?

A `.bgpx` service is any application accessible via the BGP-X overlay network at a `.bgpx` address. Examples:

- A community news website: `community-news.bgpx`
- A private file sharing service accessible only to island members
- An API for IoT sensor data from a mesh island
- A community chat service
- A weather station dashboard hosted on a remote mesh node
- A mesh island directory service
- A latency-tolerant messaging platform
- A decentralized marketplace for community goods

`.bgpx` services have three key properties:

- **Privacy**: all traffic is end-to-end encrypted via BGP-X onion routing; the operator's server IP is hidden
- **Self-authentication**: the `.bgpx` address is the cryptographic identity — no CA needed
- **Cross-domain accessibility**: services hosted inside mesh islands are reachable from clearnet users via domain bridge nodes

---

## 2. Generating Your Service Identity

Every `.bgpx` service has an Ed25519 keypair. The public key IS the service's `.bgpx` address.

```bash
# Generate service keypair
bgpx-keygen --service --output /etc/bgpx/my-service.key

# Show your .bgpx address (64 hex characters = 32 bytes = Ed25519 public key)
bgpx-keytool show-public --key /etc/bgpx/my-service.key --format bgpx-address
# Output: a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx
```

**Keep this key safe.** Back it up. If lost, your `.bgpx` address is lost permanently — you cannot recover it or transfer it. Store the backup encrypted and offline.

### Key Security Best Practices

1. **Generate on an air-gapped machine** for maximum security
2. **Store in hardware security module (HSM)** if available (especially for high-value services)
3. **Use TPM 2.0** if running on BGP-X Router v1 or Gateway v1 hardware
4. **Create encrypted backups** with strong passphrase
5. **Never store the private key in web-accessible directories**

---

## 3. Registering a Short Name

Instead of sharing a 64-character address, register a human-readable short name:

```bash
bgpx-cli names register my-service --service-key /etc/bgpx/my-service.key

# Output:
# Registered: my-service.bgpx
# Points to: a3f2b9c1....bgpx
# Expires: 2026-10-15 (90 days)
# Renewal due: 2026-09-15 (renew before expiry)
```

### Name Rules

- 3-63 characters
- Lowercase letters, numbers, hyphens only
- No leading or trailing hyphens
- Cannot start with a digit
- First-come, first-served (no dispute resolution in protocol)

### Name Selection Guidance

Choose names that are:
- **Descriptive**: `lima-community-news` is better than `news-42`
- **Memorable**: short, easy to type
- **Stable**: you can't change the name without re-registering
- **Non-trademark-infringing**: you are responsible for your name choice

### Automatic Renewal

The BGP-X daemon renews your name automatically if the service is registered locally:

```toml
# In /etc/bgpx/config.toml:
[[services]]
key_path = "/etc/bgpx/my-service.key"
name = "my-service"
auto_renew_name = true
renew_before_days = 30  # Renew 30 days before expiry
```

### Manual Renewal

```bash
bgpx-cli names renew my-service --service-key /etc/bgpx/my-service.key
```

### Name Withdrawal

If you want to release a name:

```bash
bgpx-cli names withdraw my-service --service-key /etc/bgpx/my-service.key
```

---

## 4. Simple Static Website Hosting

The simplest `.bgpx` service: a static website served directly by `bgpx-serve`:

```bash
# Create your website
mkdir -p /var/www/my-service
echo "<h1>Welcome to my-service.bgpx</h1>" > /var/www/my-service/index.html

# Start serving
bgpx-serve \
    --dir /var/www/my-service \
    --key /etc/bgpx/my-service.key \
    --name my-service \
    --advertise

# Your site is now accessible at:
# bgpx://my-service.bgpx/
# bgpx://a3f2b9c1....bgpx/
```

### bgpx-serve Automatic Features

`bgpx-serve` automatically:

- Handles HTTP/2 framing
- Applies Brotli compression (better than gzip for LoRa)
- Sets appropriate cache headers
- Responds to `X-BGP-X-Latency-Class` path type headers
- Implements server push for CSS and JS files
- Generates ETags for cache validation
- Serves pre-compressed `.br` and `.gz` files if present

### Configuration File

For persistent deployment:

```toml
# /etc/bgpx/services/my-service.toml
[service]
name = "my-service"
key_path = "/etc/bgpx/my-service.key"
type = "static"

[static]
directory = "/var/www/my-service"
auto_index = true  # Serve index.html for directories
cache_ttl = 86400  # 24 hours
compress = "br"    # Brotli compression
```

---

## 5. Web Application Hosting

For a web application (Node.js, Python, Go, Rust, etc.), connect the BGP-X SDK and pass incoming connections to your HTTP/2 server:

### Node.js Example

```javascript
const { Client } = require('@bgpx/sdk');
const http2 = require('http2');

const client = await Client.connectAuto();
const listener = await client.registerService({
    name: 'my-app',
    advertise: true,
    maxConnections: 100,
});

// For each incoming BGP-X connection, create an HTTP/2 session
for await (const [stream, peerInfo] of listener) {
    // Stream is an AsyncRead/AsyncWrite trait implementation
    // Create an HTTP/2 session over it
    const http2Session = http2.createServerSession(stream, {
        settings: { enablePush: true }
    });

    // Handle HTTP/2 requests
    http2Session.on('stream', (req, res) => {
        // Standard HTTP/2 request handling
        res.writeHead(200, { 'content-type': 'text/html' });
        res.end('<h1>Hello from .bgpx</h1>');
    });
}
```

### Python Flask Example

```python
from bgpx_sdk import Client, ServiceConfig
from flask import Flask
import asyncio

app = Flask(__name__)

@app.route('/')
def home():
    return '<h1>Hello from .bgpx</h1>'

async def serve():
    client = await Client.connect_auto()
    listener = await client.register_service(ServiceConfig(
        name="flask-app",
        advertise=True,
    ))

    async for stream, peer_info in listener:
        # Stream is an async HTTP/2 transport
        asyncio.create_task(handle_stream(app, stream))

asyncio.run(serve())
```

### Rust Example (Native)

```rust
use bgpx_sdk::{Client, ServiceConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::connect_auto().await?;
    let listener = client.register_service(ServiceConfig {
        name: "rust-app".to_string(),
        advertise: true,
        ..Default::default()
    }).await?;

    while let Some((mut stream, peer_info)) = listener.accept().await {
        tokio::spawn(async move {
            // Handle HTTP/2 over the stream
            handle_http2(&mut stream).await;
        });
    }

    Ok(())
}
```

---

## 6. LoRa-Optimized Service Design

If your service will be accessible from mesh islands via LoRa paths, design for high latency. **LoRa paths introduce 500ms-5 seconds of latency per round-trip.** Standard browsers designed for sub-100ms web experiences will feel unusably slow. BGP-X Browser and BGP-X-native applications are designed to handle LoRa-class latency gracefully.

### Required HTTP Response Headers

For LoRa-optimized responses:

```http
X-BGP-X-Latency-Class: lora
X-BGP-X-Batch-Resources: true
X-BGP-X-Cache-TTL: 86400
Content-Encoding: br
Cache-Control: max-age=86400, immutable
```

### Page Weight Budget

| Resource | Maximum Size (Compressed) | Rationale |
|---|---|---|
| HTML document | 20 KB (Brotli) | One round-trip |
| CSS total | 15 KB (Brotli) | One additional round-trip |
| JavaScript total | 30 KB (Brotli) | One additional round-trip |
| Images (each) | 10 KB | Progressive load acceptable |
| Fonts | Use system fonts | Avoid web fonts entirely |
| Total page weight | 100 KB (Brotli) | 3-5 round-trips total |

### Latency-Adaptive Rendering

Your service should detect the client's latency class and adapt:

```javascript
// Client sends X-BGP-X-Latency-Class header
const latencyClass = req.headers['x-bgpx-latency-class'];

if (latencyClass === 'lora' || latencyClass === 'satellite-geo') {
    // Return minimal, aggressively cached response
    res.setHeader('X-BGP-X-Mode', 'minimal');
    // Inline all critical CSS
    // Skip non-essential JavaScript
    // Use server-sent events for updates instead of polling
} else {
    // Return full response
}
```

### Design Patterns for LoRa

```html
<!-- Inline critical CSS instead of external stylesheet (saves one round-trip) -->
<style>
/* Only critical above-fold styles here */
body { font-family: system-ui; margin: 0; }
.header { background: #2c3e50; color: white; padding: 1rem; }
</style>

<!-- Preload critical resources so browser can batch them -->
<link rel="preload" href="/app.css" as="style">
<link rel="preload" href="/critical.js" as="script">

<!-- Lazy load below-fold images -->
<img src="/photo.webp" loading="lazy" width="400" height="300">

<!-- Use server-sent events for updates instead of polling -->
<script>
const events = new EventSource('/updates');
events.onmessage = e => updateUI(JSON.parse(e.data));
</script>

<!-- Prefer WebP over PNG/JPEG for images -->
<picture>
    <source srcset="/image.webp" type="image/webp">
    <img src="/image.jpg" alt="...">
</picture>

<!-- Defer non-critical JavaScript -->
<script src="/analytics.js" defer></script>
```

### Progressive Enhancement

Your service should work at three levels:

1. **Basic**: HTML only, functional, accessible with any client
2. **Enhanced**: CSS for styling, cached aggressively
3. **Full**: JavaScript for interactivity, loaded last

---

## 7. Hosting a Service Inside a Mesh Island

Services registered from a mesh island node are discoverable from clearnet when the island's gateway is online:

```bash
# On your island node (not the gateway):
bgpx-serve --dir /var/www/local-service \
    --key /etc/bgpx/local-service.key \
    --name local-news \
    --advertise

# The gateway publishes your service's DHT record to the internet DHT
# Clearnet users can discover and reach local-news.bgpx
```

### How Mesh Island Service Discovery Works

1. Your service registers in the **unified DHT** via the mesh island's bridge/gateway node
2. The bridge/gateway node stores your service's advertisement in the internet-accessible portion of the DHT
3. Clearnet clients query the DHT and discover your service
4. Path construction includes a DOMAIN_BRIDGE hop through the gateway
5. Clearnet users can reach your mesh service without any mesh hardware

### When the Gateway Goes Offline

When the island's gateway goes offline:

- Your service remains accessible to **island members** (intra-island traffic continues)
- Your service becomes **unreachable from clearnet** until the gateway reconnects
- Your service's DHT record **expires after 48 hours** if the gateway doesn't republish
- When the gateway reconnects, it **republishes the island's service records** automatically

### Multiple Gateways

For reliability, mesh islands should have **multiple gateway nodes** from different operators:

- If one gateway fails, others continue publishing to the DHT
- Clients automatically use an available gateway for cross-domain paths
- Service discovery remains functional even with gateway failures

---

## 8. Cross-Domain Service Hosting

A `.bgpx` service can be hosted in any routing domain and accessed from any other domain:

| Service Location | Accessible From | Path Requirements |
|---|---|---|
| Clearnet server | Clearnet, Overlay, Mesh | Standard BGP-X path (clearnet relays) |
| Mesh island server | Clearnet, Overlay, Mesh | Cross-domain path with DOMAIN_BRIDGE hop |
| Multi-domain (bridge node) | Clearnet, Overlay, Mesh | Shortest path to bridge node |

### Example: Clearnet User Accessing Mesh Island Service

```
Clearnet User (no mesh hardware)
         │
         │ Standard internet
         ▼
[Entry: clearnet relay]
         │
         │ BGP-X overlay (encrypted)
         ▼
[Relay: clearnet relay]
         │
         │ BGP-X overlay
         ▼
[Bridge: domain bridge node (gateway)]
         │
         │ DOMAIN_BRIDGE hop
         │ Gateway forwards via mesh radio
         ▼
[Relay: mesh relay]
         │
         │ Mesh transport
         ▼
[Service: mesh island service]

The clearnet user experiences:
- Standard .bgpx URL: bgpx://mesh-local-news.bgpx
- Automatic path construction
- Full onion encryption throughout
- No special hardware or configuration needed
```

---

## 9. Monitoring Your Service

### Check Service Advertisement

```bash
# Check service is advertised in DHT
bgpx-cli node dht-stats | grep services

# Query DHT for your service directly
bgpx-cli nodes lookup --service-id a3f2b9c1....bgpx

# Check which nodes know about your service
bgpx-cli nodes list --knows-service my-service
```

### Check Active Connections

```bash
# See active connections to your service
bgpx-cli sessions list | grep service

# Count active streams to your service
bgpx-cli sessions list --service my-service | wc -l

# Monitor connection rate (requires watch)
watch -n 5 'bgpx-cli sessions list --service my-service | wc -l'
```

### Check Name Registry Status

```bash
# List your registered names
bgpx-cli names list --self

# Check name expiry
bgpx-cli names show my-service

# Verify name resolves to correct ServiceID
bgpx-cli names resolve my-service
```

### Prometheus Metrics

If metrics are enabled:

```bash
# Get service-specific metrics
curl http://localhost:9090/metrics | grep bgpx_service

# Key metrics:
# bgpx_service_connections_total
# bgpx_service_bytes_sent
# bgpx_service_bytes_received
# bgpx_service_errors_total
```

---

## 10. Updating and Migrating a Service

### Content Update (No Address Change)

Update content in your service's directory. No BGP-X configuration change needed — the `.bgpx` address stays the same. Users accessing your service immediately see the new content.

### Key Rotation (Changes Address)

If you must rotate your Ed25519 service key (changes your `.bgpx` address):

1. **Register a Name Registry redirect** from old name to new address
2. **Run old service briefly** with a redirect page pointing to new address
3. **Update `.bgpx` browser bookmarks** (automated if you registered a short name — update name to point to new ServiceID)
4. **Communicate the change** to your users via any available channel
5. **Shut down old service** after enough time for users to discover new address (recommend: 7 days minimum)
6. **Withdraw old short name** and register new short name pointing to new ServiceID

### Migration Procedure

```bash
# 1. Generate new key
bgpx-keygen --service --output /etc/bgpx/my-service-new.key

# 2. Register redirect on old name
bgpx-cli names update my-service \
    --old-key /etc/bgpx/my-service.key \
    --new-address $(bgpx-keytool show-public --key /etc/bgpx/my-service-new.key --format hex)

# 3. Start new service alongside old
bgpx-serve --dir /var/www/my-service-new \
    --key /etc/bgpx/my-service-new.key \
    --name my-service-new \
    --advertise &

# 4. Update old service to return redirect
echo '<html><head><meta http-equiv="refresh" content="0;url=bgpx://my-service-new.bgpx/"></head></html>' \
    > /var/www/my-service/index.html

# 5. Wait 7 days, then withdraw old name
bgpx-cli names withdraw my-service --service-key /etc/bgpx/my-service.key

# 6. Update my-service to point to new address
bgpx-cli names register my-service --service-key /etc/bgpx/my-service-new.key

# 7. Stop old service
pkill -f "bgpx-serve.*my-service.key"
```

---

## 11. Security Practices

### Key Security

- **Back up your service key**: loss = permanent loss of `.bgpx` address
- **Store key outside web root**: key must not be accessible via HTTP
- **Use hardware security**: TPM 2.0 on BGP-X hardware, HSM for high-value services
- **Never commit private keys to version control**

### Application Security

- **Rate limiting**: implement at application level (bgpx-sdk provides connection count limits)
- **Input validation**: standard web security practices apply
- **No SQL injection**: your service handles HTTP/2 requests like any web app
- **HTTPS is NOT needed**: BGP-X provides transport encryption; end-to-end via onion layers
- **Session management**: use the SDK's session identity features if you need user sessions

### Logging Policy

- **No private data in logs**: do not log client path information even if SDK provides it
- **Do not log client NodeIDs**: this could link activity patterns
- **Do not log destination addresses**: defeats BGP-X's purpose
- **Do not log timing patterns**: enables correlation attacks
- **Log only**: service start/stop, connection count (aggregate), errors (no identifiers)

### Keep Software Updated

- **Update BGP-X daemon regularly**: daemon updates include security patches
- **Monitor security advisories**: subscribe to BGP-X security mailing list
- **Apply firmware updates**: if running on BGP-X hardware

### Content Responsibility

- **Review content carefully**: anything hosted at your `.bgpx` address is your responsibility
- **No illegal content**: BGP-X provides privacy, not legal immunity
- **Community standards**: if your service is in a community mesh island, follow community guidelines

---

## 12. Performance Tuning

### For Clearnet-Accessible Services

- **Use standard HTTP/2 optimization**: multiplex resources, minimize round-trips
- **Enable Brotli compression**: significantly smaller payloads
- **Set aggressive cache headers**: reduce repeated requests
- **Use a CDN for static assets** (optional, external to BGP-X)

### For Mesh-Island Services

- **Minimize round-trips**: every round-trip costs 1-5 seconds on LoRa
- **Inline critical resources**: avoid separate CSS/JS files for critical path
- **Use aggressive caching**: LoRa bandwidth is precious
- **Implement progressive rendering**: show content as it arrives
- **Use server-sent events** instead of polling for real-time updates

### For Satellite-Connected Services

- **Accept high latency**: GEO satellite adds 600ms+ RTT
- **Batch operations**: minimize number of requests
- **Use background sync**: allow operations to complete asynchronously

---

## 13. Troubleshooting

### Service Not Reachable

1. **Check daemon is running**: `systemctl status bgpx-node`
2. **Check service is registered**: `bgpx-cli services list`
3. **Check advertisement in DHT**: `bgpx-cli nodes lookup --service-id <your-service-id>`
4. **Check name resolves**: `bgpx-cli names resolve <your-service-name>`
5. **Check firewall rules**: ensure BGP-X ports are open

### Service Slow on LoRa

1. **Reduce page weight**: follow LoRa optimization guidelines above
2. **Enable Brotli compression**: use `bgpx-serve` with `--compress br`
3. **Check page weight budget**: total should be <100KB compressed
4. **Minimize HTTP requests**: use HTTP/2 multiplexing, inline resources
5. **Consider text-only mode**: strip images for critical information

### Name Not Resolving

1. **Check name exists**: `bgpx-cli names show <name>`
2. **Check name hasn't expired**: names expire after 90 days
3. **Check DHT connectivity**: `bgpx-cli node dht-stats`
4. **Check name points to correct ServiceID**: verify public key matches

### Connection Drops Frequently

1. **Check network stability**: clearnet or mesh connectivity
2. **Check for session timeouts**: sessions expire after 90 seconds idle
3. **Check for re-handshakes**: sessions re-handshake every 24 hours
4. **Check logs for errors**: `journalctl -u bgpx-node | grep -i error`

---

## 14. Advanced: Running a Domain Bridge with a Service

If you're operating a domain bridge node AND hosting a service on the same device:

```toml
# /etc/bgpx/config.toml

[node]
role = "relay,bridge"

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ address = "0.0.0.0", port = 7474 }]

[[routing_domains]]
domain_type = "mesh"
island_id = "lima-district-1"
transports = ["wifi_mesh", "lora"]

[domain_bridge]
enabled = true
pairs = [
    { from = "clearnet", to = "mesh:lima-district-1" }
]

[[services]]
name = "local-directory"
key_path = "/etc/bgpx/directory.key"
type = "static"

[services.static]
directory = "/var/www/directory"
```

Your service will be:
- Discoverable from both clearnet and mesh island
- Accessible via the shortest path to your bridge node
- Subject to the same cross-domain routing as any other mesh service

---

## 15. Summary Checklist

Before going live with your `.bgpx` service:

- [ ] Service key generated and backed up securely
- [ ] Short name registered (if desired)
- [ ] Service advertised in DHT
- [ ] Content optimized for target latency class
- [ ] HTTP/2 properly configured
- [ ] Security practices followed
- [ ] Monitoring configured
- [ ] Documentation written for your users

Your `.bgpx` service is now ready for the BGP-X network.
