# KIP-0003: Extension Spec Requirements

**Status:** Draft
**Version:** 1.0

## Abstract

KIP-0003 defines what an extension is, how extensions are identified, what an extension spec must contain, and how extensions are declared in manifests.

## 1. What is an Extension

An extension adds behavior to a KIP network. Everything beyond core concepts (KIP-0001) and the manifest format (KIP-0002) is an extension. Data exchange, discovery, auth, encryption, streaming — all extensions.

## 2. Extension Identification

Extensions are identified by name, not by number.

### 2.1 Built-in Extensions

Built-in extensions are defined in the KIP repository and identified by a short name:

```
chunk-exchange
kippit-tracker
jwt-auth
aes-encryption
key-delivery
hls-streaming
resource-catalog
resource-metadata
sync
messaging
bt-bridge
bt-tracker
manifest-visibility
webrtc-signaling
mdns-discovery
```

Built-in extensions are maintained as part of the KIP spec. Their specs live in the KIP repository as separate documents.

### 2.2 External Extensions

External extensions are identified by a repository path:

```
github.com/someone/cool-extension
github.com/kippit-net/kip-ext-something
gitlab.com/org/custom-tracker
```

The path points to the repository where the extension spec and (optionally) reference implementation live. The path is an identifier — nodes don't need to fetch the repository at runtime.

### 2.3 Naming Rules

- Built-in names: lowercase, kebab-case, no dots, no slashes.
- External paths: standard URL path format (domain/org/repo).
- A name without dots or slashes = built-in. A name with dots = external.

## 3. Extensions in Manifests

Extensions are declared per interface in the manifest (see KIP-0002 section 3):

```json
{
  "interfaces": [
    {
      "id": "p2p-video",
      "role": "peer",
      "extensions": [
        {
          "name": "chunk-exchange",
          "version": "1.0.0"
        },
        {
          "name": "jwt-auth",
          "version": "1.0.0",
          "config": {
            "method": "ES256"
          }
        },
        {
          "name": "github.com/someone/custom-discovery",
          "version": "0.2.0"
        }
      ]
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | YES | Extension identifier (built-in name or repo path). |
| `version` | string | YES | Extension version (semver). |
| `config` | object | NO | Extension-specific configuration. Schema defined by the extension spec. |

Required extensions in rules also use names:

```json
{
  "rules": {
    "required_extensions": ["jwt-auth", "aes-encryption"]
  }
}
```

## 4. Extension Spec Template

Every extension spec — built-in or external — MUST define the following sections:

### 4.1 Identity

- **Name:** The extension identifier.
- **Layer(s):** Which layer(s) of the KIP model it operates on (discovery, connection, exchange, semantics — or multiple).
- **Roles:** Which roles it applies to (peer, tracker, or both).

### 4.2 Purpose

What the extension does. One paragraph. If you can't explain it in one paragraph, it's too many extensions bundled together.

### 4.3 Manifest Contribution

What the extension adds to a node's manifest:

- **Config fields:** Extension-specific fields in the `config` object (e.g. `{"method": "ES256"}` for auth).
- **Rule fields:** Fields it adds to the `rules` object (e.g. `"auth_required": true`).
- **Network entries:** Service endpoints it exposes via the `network` map (e.g. HLS manifest URLs).

### 4.4 Negotiation

How nodes agree to use this extension:

- **Tracker-mandated:** The tracker's manifest lists it in `required_extensions`. Peers that don't support it can't join.
- **Peer-negotiated:** Both peers declare the extension in their manifests. Active only if both support it.
- **Configuration-only:** Enabled by node configuration, no runtime negotiation needed.
- **Any combination** of the above.

### 4.5 Communication

How nodes communicate using this extension:

- **Message formats** (if any) — binary, JSON, or whatever fits the layer.
- **Endpoints** (if any) — HTTP paths, WebSocket message types, etc.
- **State machines** (if any) — connection lifecycle, handshake flow.
- **Error handling** — what happens when things go wrong.

### 4.6 Dependencies

What other extensions this extension requires or optionally interacts with.

### 4.7 Example

At least one concrete example showing the extension in use: a manifest fragment, a message exchange, or a flow diagram.

## 5. Built-in Extension Registry

The following built-in extensions are defined or planned for the Kippit protocol:

| Name | Layer | Roles | Status | Description |
|---|---|---|---|---|
| `chunk-exchange` | Exchange | peer | Planned | Chunk-based file transfer. Binary framing, bitfield, request/piece. |
| `kippit-tracker` | Discovery | tracker, peer | Planned | WebSocket-based peer discovery and signaling. |
| `jwt-auth` | Semantics | tracker, peer | Planned | JWT authentication (ES256). |
| `aes-encryption` | Semantics | peer | Planned | Per-chunk encryption (AES-128-CBC, AES-256-GCM). |
| `key-delivery` | Semantics | peer, tracker | Planned | Encryption key delivery (API, URL, peer-exchange, manual). |
| `hls-streaming` | Semantics | peer | Planned | Sequential chunk priority, HLS manifest generation. |
| `resource-catalog` | Discovery | peer, tracker | Planned | Browseable resource listing (JSON/text, pagination, search). |
| `resource-metadata` | Semantics | peer, tracker | Planned | Per-resource metadata (name, size, type, extras). |
| `sync` | Semantics | peer | Planned | Multi-node replication, operation log, conflict resolution. |
| `messaging` | Semantics | peer | Planned | Real-time small data exchange between peers. |
| `bt-bridge` | Exchange | peer | Planned | BitTorrent wire protocol compatibility layer. |
| `bt-tracker` | Discovery | tracker | Planned | BitTorrent tracker protocol (HTTP/UDP announce, scrape). |
| `manifest-visibility` | Semantics | tracker, peer | Planned | Controls what appears in public manifest (public/unlisted/hidden). |
| `webrtc-signaling` | Connection | tracker, peer | Planned | WebRTC SDP/ICE exchange for NAT traversal. |
| `mdns-discovery` | Discovery | peer | Planned | LAN zero-config peer discovery via mDNS/DNS-SD. |

### Governance & Operations (future)

Extensions for network governance, resource management, and business logic. Not needed for PoC — needed for production networks with multiple participants and shared resources.

| Name | Layer | Roles | Status | Description |
|---|---|---|---|---|
| `lease-agreement` | Semantics | peer, tracker | Future | Terms of hosting between peers. Storage limits, duration, SLA, what happens on disconnect. |
| `metered-usage` | Semantics | peer, tracker | Future | Bandwidth and storage accounting. Usage reporting to tracker or between peers. |
| `quota` | Semantics | tracker | Future | Resource limits enforced by tracker. Max storage, bandwidth, resources per peer. |
| `reputation` | Semantics | tracker | Future | Peer reliability scoring. Uptime, response time, fulfillment rate. Influences mesh allocation. |
| `billing` | Semantics | tracker | Future | Payment and settlement for hosting/bandwidth. Integrates with external payment systems. |
| `moderation` | Semantics | tracker | Future | Content policy enforcement. Flagging, takedown, abuse reporting. |
| `governance` | Semantics | peer, tracker | Future | Democratic/hybrid rule negotiation. Voting mechanism, quorum, rule proposals, enforcement delegation. |

These extensions compose with the technical layer. For example, mesh hosting requires `chunk-exchange` + `sync` + `lease-agreement` + `metered-usage` — the technical extensions move data, the governance extensions manage the relationship.

Each built-in extension has its own spec document in the KIP repository (e.g. `ext-chunk-exchange.md`, `ext-kippit-tracker.md`).

## 6. Extension Composability

Extensions are independent by default. A node runs whatever combination its configuration and network require. Examples:

| Node | Extensions | Why |
|---|---|---|
| Kippit runner (phase 1) | chunk-exchange, kippit-tracker, jwt-auth, aes-encryption, key-delivery, hls-streaming, resource-catalog, resource-metadata, webrtc-signaling, mdns-discovery | Full Kippit stack |
| Kippit runner (phase 0) | chunk-exchange, kippit-tracker, resource-catalog | Minimum viable peer with browseable files |
| Kippit tracker | kippit-tracker, jwt-auth, webrtc-signaling, resource-catalog | Discovery + auth + signaling + index |
| BT peer | bt-bridge, chunk-exchange | Speaks BT and KIP from the same data |
| BT tracker | bt-tracker | HTTP/UDP announce and scrape for BT clients |
| Hybrid tracker | kippit-tracker, bt-tracker | Bridges BT and KIP discovery |
| LAN video server | chunk-exchange, hls-streaming, mdns-discovery, resource-catalog, resource-metadata | No auth, no tracker, LAN only, browseable |
| Signaling-only server | webrtc-signaling | Just relays SDP/ICE |
| Public file index | chunk-exchange, resource-catalog, resource-metadata | Browseable, searchable, no auth |
| Custom network | chunk-exchange, github.com/acme/custom-auth | Mix of built-in and external |

## 7. Configuration Cascading

A tracker's manifest drives peer configuration. When a peer joins a network, the tracker's rules determine what extensions the peer must run and how.

### 7.1 How Rules Cascade

```
Tracker manifest                    Peer configuration
┌──────────────────────┐           ┌──────────────────────────┐
│ required_extensions:  │           │                          │
│   - jwt-auth          │──────────►│ Must enable jwt-auth     │
│   - aes-encryption    │──────────►│ Must enable aes-encrypt  │
│                       │           │                          │
│ rules:                │           │                          │
│   auth_required: true │──────────►│ Must provide JWT token   │
│   encryption_required │──────────►│ Must encrypt all chunks  │
│   chunk_size: 2097152 │──────────►│ Must use 2MB chunks      │
│   announce_mode:      │           │                          │
│     per-resource      │──────────►│ Must list resource_ids   │
└──────────────────────┘           └──────────────────────────┘
```

A peer that cannot satisfy the tracker's requirements MUST NOT join the network.

### 7.2 Extension Configuration Negotiation

Extensions MAY define configurable parameters with alternatives. When two nodes negotiate, they find a compatible configuration:

**Example: encryption cipher negotiation.**

Tracker mandates `aes-encryption` but doesn't specify cipher. Peer A supports `AES-128-CBC` and `AES-256-GCM`. Peer B supports `AES-256-GCM` and `ChaCha20-Poly1305`. They negotiate `AES-256-GCM` (the intersection).

This is extension-specific — each extension defines:
- What parameters are configurable
- What alternatives exist
- How to negotiate between options
- What happens when no compatible option exists (error, fallback, reject)

### 7.3 Extension Spec: Configuration Section

In addition to the template (section 4), extensions SHOULD define a **configuration section** that documents:

| What | Description |
|---|---|
| **Configurable parameters** | What can be configured (cipher, chunk size, mode, etc.) |
| **Alternatives** | What options exist for each parameter |
| **Defaults** | What value is used when not specified |
| **Tracker-mandatable** | Which parameters a tracker can force via rules |
| **Negotiable** | Which parameters peers negotiate between themselves |
| **Fixed** | Which parameters are not configurable |

### 7.4 Resource Identification

A tracker MAY mandate how resources are identified in its network. For example:

- "Hash file paths using SHA256 and announce the first 16 bytes as resource_id" — this is the Kippit default
- "Use content-based hashing" — a different network might require this
- "Encrypt file paths before hashing" — for privacy-sensitive networks

The hashing algorithm, input format, and output size are part of the network's rules. A peer joining the network must comply. This is declared in the tracker's manifest rules:

```json
{
  "rules": {
    "resource_id": {
      "algorithm": "SHA256",
      "input": "runner_id:relative_path",
      "output_bytes": 16
    }
  }
}
```

## 8. Network Profiles

A **network profile** is an informal description of what a specific network requires. It's not a KIP concept — it's a convenience for documenting "if you want to join network X, here's what you need."

### 8.1 Kippit Network Profile

The Kippit network (`kippit.net`) requires:

| Requirement | Extension | Config |
|---|---|---|
| Authentication | `jwt-auth` | ES256, JWKS at `kippit.net/.well-known/jwks.json` |
| Encryption | `aes-encryption` | AES-128-CBC (HLS standard) |
| Key delivery | `key-delivery` | API + URL methods |
| Data exchange | `chunk-exchange` | 2MB chunks |
| Discovery | `kippit-tracker` | WebSocket, per-resource announce |
| NAT traversal | `webrtc-signaling` | ICE servers provided by tracker |
| Video | `hls-streaming` | m3u8 manifests served over HTTP |
| Resource listing | `resource-catalog` | JSON format, searchable via API |
| Resource info | `resource-metadata` | Name, type, size, HLS extras |

Optional:
| Feature | Extension | When |
|---|---|---|
| LAN discovery | `mdns-discovery` | Runner on local network |
| Replication | `sync` | Multiple runners, change tracking |
| Small data | `messaging` | Status updates, notifications |

The tracker manifest enforces `jwt-auth`, `aes-encryption`, and `key-delivery` as required. Other extensions are available but not mandated — a peer without `hls-streaming` can still participate in the network (it just can't serve video efficiently).

### 8.2 Other Network Profiles (Hypothetical)

**Public file sharing network:**
- No auth, no encryption
- `chunk-exchange` only
- `announce_mode: per-resource`
- Anyone can join

**Corporate LAN-only network:**
- `jwt-auth` required (corporate SSO)
- `aes-encryption` required
- `mdns-discovery` only (no external tracker)
- No `webrtc-signaling` (everything on LAN)

**BitTorrent bridge (peer):**
- `bt-bridge` — speaks BT wire protocol to BT peers
- `chunk-exchange` — speaks KIP to KIP peers
- Same data served to both worlds

**BitTorrent tracker:**
- `bt-tracker` — HTTP/UDP announce and scrape for BT clients
- Optionally combined with `kippit-tracker` for hybrid BT+KIP discovery

**Public file index:**
- `chunk-exchange` + `resource-catalog` + `resource-metadata`
- No auth, no encryption
- `resource-catalog` with `searchable: true` — public browseable index
- Anyone can join and list files

## 9. Versioning

Extension versions follow semver:

- **Major:** Breaking changes. Nodes on different major versions may be incompatible.
- **Minor:** New features, backward compatible.
- **Patch:** Bug fixes, clarifications.

Two nodes with the same extension at different versions negotiate by using the lower version, unless the extension spec defines otherwise. This is an extension-level concern — KIP-0003 does not mandate a specific version negotiation algorithm.
