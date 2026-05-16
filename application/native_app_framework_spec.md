**Version**: 0.1.0-draft  
**Status**: Pre-implementation specification  
**License**: MIT (Code), CC BY 4.0 (Documentation)

---

## 1. Application Tiers

BGP-X applications exist in three tiers with increasing levels of integration, privacy awareness, and resilience. The framework supports each tier distinctly, allowing developers to choose the appropriate level of engagement.

### 1.1 Tier 1: Standard Applications

**BGP-X Unaware**. Standard applications (browsers, email clients, CLI tools, game clients) running on a LAN device protected by a **BGP-X Router v1** have all their traffic routed through the overlay automatically via the TUN interface (`bgpx0`) and the routing policy engine.

- **Requirements**: None. The application functions unchanged.
- **Capabilities**: Transparent encryption, anonymity, and privacy.
- **Limitations**: No path control, no `.bgpx` native service hosting, no cross-domain path specification, no insight into routing status.

### 1.2 Tier 2: BGP-X-Aware Applications

**SDK Integrated**. These applications link against `bgpx-sdk` to explicitly control path parameters, subscribe to network events, or access routing domain information. They do not necessarily host services or handle extreme latency gracefully, but they leverage the overlay programmatically.

- **Requirements**:
  - Link against `bgpx-sdk` (Rust, Python, TypeScript, Go).
  - Connect to the BGP-X daemon via the SDK Socket (`/var/run/bgpx/sdk.sock` or TCP endpoint).
  - Handle SDK error types gracefully.
- **Capabilities**:
  - Path constraints (hops, pool selection, ECH requirements).
  - Cross-domain path specification (`DomainSegmentConfig`).
  - Event subscription (path status, latency changes, domain transitions).
  - BGP-X native service access (connecting to `.bgpx` addresses).

### 1.3 Tier 3: BGP-X-Native Applications

**Fully Integrated**. These applications are designed from the ground up for the BGP-X ecosystem. They host `.bgpx` services, handle LoRa-class latency gracefully, operate fully offline, and adapt dynamically to all routing domain conditions (clearnet, mesh, satellite).

- **Requirements**: All Tier 2 requirements **plus** the specific requirements detailed in Section 2.
- **Capabilities**:
  - All Tier 2 capabilities.
  - `.bgpx` service hosting.
  - Graceful degradation on high-latency or offline paths.
  - Progressive UI rendering and data synchronization.
  - Cross-domain awareness (operating seamlessly across clearnet and mesh islands).

---

## 2. Tier 3 Application Requirements

Tier 3 applications represent the highest standard of BGP-X integration. They must be resilient, responsive, and privacy-preserving even under adverse network conditions.

### 2.1 Latency Tolerance

Applications **MUST NOT** assume sub-100ms response times. They **MUST** handle the full spectrum of latency classes defined in the protocol:

| Latency Class | RTT Range | Context |
|---|---|---|
| `standard` | < 100ms | Fiber/Cellular clearnet |
| `overlay` | 50–200ms | BGP-X clearnet relay path |
| `mesh-wifi` | 20–200ms | WiFi 802.11s mesh |
| `mesh-lora` | 500ms–5000ms | LoRa mesh (SF7–SF12) |
| `satellite-leo` | 20–60ms | Starlink, OneWeb |
| `satellite-geo` | 600ms–3000ms | Inmarsat, HughesNet |

**Implementation**:

```rust
use bgpx_sdk::{Client, LatencyClass, PathEvent};

let client = Client::connect_auto().await?;

// Subscribe to path events
let mut path_events = client.subscribe_path_events().await?;

while let Some(event) = path_events.recv().await {
    match event.latency_class {
        LatencyClass::MeshLora | LatencyClass::SatelliteGeo => {
            // Switch to high-latency mode
            app.set_mode(AppMode::HighLatency {
                batch_requests: true,
                polling_interval_secs: 300, // 5 minutes
                timeout_secs: 120,         // Extended timeout
            });
            app.disable_realtime_features();
        }
        LatencyClass::MeshWifi | LatencyClass::SatelliteLeo => {
            app.set_mode(AppMode::ModerateLatency);
        }
        _ => {
            app.set_mode(AppMode::Standard);
        }
    }
}
```

### 2.2 Batched Operations

To minimize the impact of high RTT, applications **MUST** batch multiple logical operations into single network round-trips when operating on high-latency paths.

**Anti-Pattern (Sequential Fetches on LoRa)**:
```rust
// N requests × 3 seconds RTT = N×3 seconds total
let user = fetch_user(id).await?;              // 3s
let posts = fetch_posts(user.id).await?;       // 3s
let comments = fetch_comments(posts[0].id).await?; // 3s
// Total: 9 seconds for 3 items
```

**Best Practice (Batched Request)**:
```rust
// 1 request × 3 seconds RTT = 3 seconds total
let response = fetch_batch(BatchRequest {
    user_id: id,
    include_posts: true,
    include_comments_for_first_post: true,
}).await?;

// Total: 3 seconds for all data
```

Applications should detect high latency and automatically switch to batched query modes.

### 2.3 Stream State Handling

Applications **MUST** handle `StreamPaused` and `StreamResumed` events without crashing, data loss, or user confusion. These events occur when:
- Duty cycle limits are hit on LoRa links
- Mesh island connectivity is temporarily lost
- Satellite link degrades

**Implementation**:

```rust
use bgpx_sdk::{SdkError, StreamEvent};

loop {
    match stream.write(&data).await {
        Ok(_) => continue,
        Err(SdkError::StreamPaused { reason, domain }) => {
            // Queue data locally; resume when stream resumes
            pending_queue.push(data);
            ui.show_status(&format!("Offline: {} ({})", reason, domain));
        }
        Err(e) => return Err(e.into()),
    }
}

// Event listener for resume
sdk_events.on(StreamEvent::Resumed, |_| {
    for item in pending_queue.drain(..) {
        let _ = stream.write(&item); // Retry
    }
    ui.clear_status();
});
```

### 2.4 Offline Capability

Tier 3 applications **MUST** remain functional (at minimum: read-only) when the cross-domain stream is unavailable. This requires a local-first data architecture.

**Implementation Pattern**:

```rust
// 1. Always read from local cache first
if let Some(cached) = local_db.get(&resource_key)? {
    ui.display(cached.data);
}

// 2. Attempt sync if network available
if let Ok(fresh) = service.fetch(&resource_key).await {
    local_db.store(&resource_key, fresh.clone())?;
    ui.display(fresh.data);
}
```

Local storage mechanisms (SQLite, sled, redb, or native platform storage) should be encrypted if they contain sensitive user data.

### 2.5 Progressive UI Rendering

Applications **MUST** display meaningful content immediately, utilizing "skeleton" UIs and progressive loading techniques. Users should never stare at a blank screen.

**Implementation**:

```rust
// Render placeholder UI instantly
ui.render_skeleton(&layout);

// Stream partial data as it arrives
stream.on_partial_data(|chunk| {
    if let Some(partial) = parse_partial(chunk) {
        ui.update_progressive(partial);
    }
});

// Final render on completion
stream.on_complete(|full_data| {
    ui.render_complete(full_data);
});
```

### 2.6 App Manifest

Tier 3 applications declare capabilities in a `.bgpx` app manifest (`manifest.json`), served at the root of their `.bgpx` address.

**Example Structure**:

```json
{
  "bgpx_manifest_version": 1,
  "name": "Community Forum",
  "service_id": "a3f2b9c1deadbeef...",
  "short_name": "community-forum",
  "version": "1.2.0",
  "latency_classes_supported": [
    "standard", "overlay", "mesh-wifi", "mesh-lora", "satellite-geo"
  ],
  "offline_capable": true,
  "min_bgpx_version": "0.1.0",
  "permissions": ["overlay_path", "domain_discovery", "cross_domain_path"],
  "service_type": "web",
  "description": "Community discussion forum for Lima District 1 mesh island",
  "icon": "bgpx://a3f2b9c1....bgpx/icon.svg",
  "start_url": "bgpx://a3f2b9c1....bgpx/"
}
```

The manifest allows the **BGP-X Browser** to:
- List the app in the service directory.
- Negotiate capabilities (e.g., warn if app doesn't support LoRa latency).
- Manage offline caching strategies.

---

## 3. SDK Integration for Native Apps

Tier 2 and Tier 3 applications utilize the `bgpx-sdk` to interact with the BGP-X daemon.

### 3.1 Initialization

The SDK connects to the daemon via the SDK Socket.

```rust
use bgpx_sdk::{Client, ClientConfig, SdkError};

// Auto-discover socket path
let client = Client::connect_auto().await?;

// Or connect explicitly via TCP (LAN device connecting to router daemon)
let client = Client::connect_tcp_with_auth(
    "192.168.1.1:7475",
    "your-auth-token"
).await?;
```

### 3.2 Path Construction

Applications can specify paths, including cross-domain segments.

```rust
use bgpx_sdk::{PathConfig, DomainSegmentConfig, RoutingDomain, SegmentConstraints};

let path = PathConfig {
    segments: vec![
        DomainSegmentConfig {
            domain: RoutingDomain::Clearnet,
            hops: 2,
            pool: "bgpx-default".into(),
            constraints: SegmentConstraints::default(),
            ..Default::default()
        },
        DomainSegmentConfig {
            domain: RoutingDomain::Mesh { 
                island_id: "lima-district-1".into() 
            },
            hops: 1,
            constraints: SegmentConstraints {
                require_ech: true,
                ..Default::default()
            },
            ..Default::default()
        },
    ],
    ..Default::default()
};

let stream = client.connect_stream_with_path(
    "sensitive-service.mesh.lima-district-1.bgpx:443",
    path,
    Default::default(),
).await?;
```

### 3.3 Error Handling for Domain-Aware Apps

Applications must handle cross-domain specific errors:

```rust
match stream.open().await {
    Ok(_) => { /* success */ },
    Err(SdkError::DomainBridgeUnavailable { from, to }) => {
        ui.show_error("Cannot reach mesh island. No bridge available.");
    }
    Err(SdkError::MeshIslandUnreachable { island_id }) => {
        ui.show_error(&format!("Island {} is currently offline.", island_id));
    }
    Err(SdkError::StreamPaused { .. }) => {
        // Queue data locally
    }
    Err(e) => return Err(e.into()),
}
```

---

## 4. Service Worker for .bgpx

The **BGP-X Browser** supports Service Workers for `.bgpx` services, enabling offline caching, background sync, and push notifications without relying on centralized infrastructure.

### 4.1 Service Worker Registration

```javascript
// In the .bgpx service's main page:
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('bgpx://community-forum.bgpx/sw.js')
    .then(reg => console.log('Service Worker registered'))
    .catch(err => console.error('SW registration failed:', err));
}
```

### 4.2 Offline Caching

```javascript
// sw.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('community-forum-v1').then((cache) => {
      return cache.addAll([
        'bgpx://community-forum.bgpx/',
        'bgpx://community-forum.bgpx/style.css',
        'bgpx://community-forum.bgpx/app.js',
        'bgpx://community-forum.bgpx/offline.html',
      ]);
    })
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request).catch(() => 
        caches.match('bgpx://community-forum.bgpx/offline.html')
      );
    })
  );
});
```

### 4.3 BGP-X Stream Resumed Event

The BGP-X Browser extends the Service Worker API with a custom event: `bgpx-stream-resumed`. This event fires when connectivity to the `.bgpx` service is restored after a PAUSED state.

```javascript
self.addEventListener('bgpx-stream-resumed', (event) => {
  event.waitUntil(syncPendingPosts());
});
```

This enables background synchronization without polling.

---

## 5. Push Notifications Without Central Provider

`.bgpx` services deliver push notifications directly via **Server-Sent Events (SSE)** over the BGP-X stream. This removes the dependency on third-party push providers (Firebase, APNs).

### 5.1 Server-Side (Service Implementation)

```javascript
// Node.js / Express example for .bgpx service
app.get('/notifications', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-BGP-X-Latency-Class': req.headers['bgp-x-path-type'] || 'standard',
  });

  // Send notifications as events
  userService.on('new_message', (msg) => {
    res.write(`id: ${msg.id}\n`);
    res.write(`event: message\n`);
    res.write(`data: ${JSON.stringify(msg)}\n\n`);
  });
});
```

### 5.2 Client-Side (In Browser or Service Worker)

```javascript
const notifications = new EventSource('bgpx://community-forum.bgpx/notifications');

notifications.addEventListener('message', (e) => {
  const msg = JSON.parse(e.data);
  
  // Display system notification
  if (Notification.permission === 'granted') {
    new Notification(msg.title, { body: msg.body });
  }
});

// Handle connection loss gracefully
notifications.onerror = () => {
  // The BGP-X Browser will attempt reconnection automatically
  console.log('Notification stream disconnected. Awaiting resume...');
};
```

On **Android** and **iOS**, the BGP-X Browser background service maintains persistent SSE connections and displays system notifications even when the browser is not in the foreground.

---

## 6. Application Discovery

Users discover `.bgpx` applications via multiple channels:

1.  **BGP-X Browser Directory**: The new tab page lists services from the Name Registry DHT (filtered by `public_directory: true`).
2.  **Island-Local Discovery**: The BGP-X daemon reports services registered in nearby mesh islands via `DOMAIN_ADVERTISE` and `MESH_ISLAND_ADVERTISE` messages.
3.  **Direct Links**: Users share `bgpx://shortname.bgpx` URLs via message, QR code, or printed materials.
4.  **Web Links**: Standard HTTPS pages can link to `bgpx://` URLs. BGP-X Browser intercepts these; other browsers prompt the user to install BGP-X Browser.

---

## 7. Application Installation Model

`.bgpx` applications follow a **Progressive Web App (PWA) model with BGP-X extensions**:

1.  User navigates to `bgpx://community-forum.bgpx/`.
2.  Browser fetches and parses `manifest.json`.
3.  Browser prompts: "Install Community Forum?".
4.  User confirms; Service Worker caches assets; app icon appears in BGP-X Browser home screen.
5.  App opens in standalone mode (no URL bar).
6.  Offline access is served by Service Worker.
7.  Updates are detected via Service Worker versioning and applied automatically.

**No Central App Store**: There is no review process or centralized marketplace. Distribution is peer-to-peer via `.bgpx` addresses, and discovery is decentralized via the Name Registry DHT.
