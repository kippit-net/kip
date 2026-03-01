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
hls-streaming
sync
messaging
bt-bridge
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

Extensions are declared in the manifest's `extensions` field:

```json
{
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
| `hls-streaming` | Semantics | peer | Planned | Sequential chunk priority, HLS manifest generation. |
| `sync` | Semantics | peer | Planned | Multi-node replication, operation log, conflict resolution. |
| `messaging` | Semantics | peer | Planned | Real-time small data exchange between peers. |
| `bt-bridge` | Exchange | peer | Planned | BitTorrent wire protocol compatibility layer. |
| `webrtc-signaling` | Connection | tracker, peer | Planned | WebRTC SDP/ICE exchange for NAT traversal. |
| `mdns-discovery` | Discovery | peer | Planned | LAN zero-config peer discovery via mDNS/DNS-SD. |

Each built-in extension will have its own spec document in the KIP repository (e.g. `ext-chunk-exchange.md`, `ext-kippit-tracker.md`).

## 6. Extension Composability

Extensions are independent by default. A node runs whatever combination its configuration and network require. Examples:

| Node | Extensions | Why |
|---|---|---|
| Kippit runner (phase 1) | chunk-exchange, kippit-tracker, jwt-auth, aes-encryption, hls-streaming, webrtc-signaling, mdns-discovery | Full Kippit stack |
| Kippit runner (phase 0) | chunk-exchange, kippit-tracker | Minimum viable peer |
| BT client | bt-bridge | Just speaks BitTorrent |
| LAN video server | chunk-exchange, hls-streaming, mdns-discovery | No auth, no tracker, LAN only |
| Signaling-only server | webrtc-signaling | Just relays SDP/ICE |
| Custom network | chunk-exchange, github.com/acme/custom-auth | Mix of built-in and external |

## 7. Versioning

Extension versions follow semver:

- **Major:** Breaking changes. Nodes on different major versions may be incompatible.
- **Minor:** New features, backward compatible.
- **Patch:** Bug fixes, clarifications.

Two nodes with the same extension at different versions negotiate by using the lower version, unless the extension spec defines otherwise. This is an extension-level concern — KIP-0003 does not mandate a specific version negotiation algorithm.
