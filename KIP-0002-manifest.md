# KIP-0002: Manifest

**Status:** Draft
**Version:** 2.0

## Abstract

KIP-0002 defines the **manifest** — a JSON document that every KIP node serves to describe itself. The manifest declares what the node is, what interfaces it exposes, what protocol it speaks, and who else it knows.

Any node in a KIP network — peer, tracker, relay, or any combination — MUST be able to serve its manifest when asked.

## 1. Serving the Manifest

A node serves its manifest as a JSON response over HTTP:

```
GET /.well-known/kip/manifest.json
```

- **Transport:** HTTP or HTTPS, depending on the node's configuration.
- **Port:** The node's primary port. No separate manifest port.
- **Content-Type:** `application/json`
- **Authentication:** None. The manifest is public. It describes what the node is, not what it contains.

A node that cannot serve HTTP (e.g. a peer behind NAT with no open ports) MAY deliver its manifest through other means — embedded in a handshake message, published at a known URL by a third party, or exchanged out-of-band. The format is the same regardless of delivery method.

The manifest is always public. What appears in it is controlled by the node's configuration and optionally by the `manifest-visibility` extension. Interfaces or extensions that the node does not want to publish simply do not appear in the manifest — the node runs them internally without advertising them.

## 2. Manifest Schema

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "interfaces": [],
  "rules": {},
  "network": []
}
```

### 2.1 Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `kip` | integer | YES | KIP spec version. Currently `1`. |
| `protocol` | string | YES | Protocol identifier. `"kippit"` for the Kippit protocol. Other protocols may use KIP as a framework with their own identifier. |
| `version` | string | YES | Protocol version (semver). Currently `"1.0.0"`. |
| `interfaces` | object[] | YES | Interfaces this node exposes. See section 3. |
| `rules` | object | NO | Network rules. Contents defined by extensions. |
| `network` | object[] | NO | Known nodes and services. See section 6. |
| `name` | string | NO | Human-readable name for this node. |
| `description` | string | NO | Human-readable description. |

### 2.2 Minimal Manifest

The smallest valid manifest:

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "interfaces": [
    { "id": "default", "role": "peer" }
  ]
}
```

This says: "I exist, I speak Kippit protocol v1.0.0, I have one interface as a peer with no extensions."

## 3. Interfaces

An **interface** is a context in which a node operates. A node may expose multiple interfaces — each with its own role and set of extensions. The same node can be a peer in one context and a tracker in another, or a peer in two contexts with different extension configurations.

```json
{
  "interfaces": [
    {
      "id": "lan-video",
      "role": "peer",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "hls-streaming", "version": "1.0.0"},
        {"name": "mdns-discovery", "version": "1.0.0"}
      ]
    },
    {
      "id": "p2p-encrypted",
      "role": "peer",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0"},
        {"name": "kippit-tracker", "version": "1.0.0"}
      ]
    }
  ]
}
```

### 3.1 Interface Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique identifier for this interface within the manifest. Free-form string. |
| `role` | string | YES | Role this interface fulfills. See section 4. |
| `description` | string | NO | Human-readable description of what this interface does. |
| `extensions` | object[] | NO | Extensions active in this interface. See section 5. |

### 3.2 Interface Semantics

- Each interface is independent. Extensions in one interface do not affect another.
- The same extension MAY appear in multiple interfaces with different configurations.
- A node with no published interfaces has nothing to offer (but may still run unpublished interfaces internally).
- Interfaces that the node does not want to publish simply do not appear in the manifest. The node runs them without advertising.

### 3.3 Simple Nodes

A node with a single purpose has a single interface:

```json
{
  "interfaces": [
    {
      "id": "default",
      "role": "peer",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"}
      ]
    }
  ]
}
```

There is no shorthand. Even the simplest node uses the `interfaces` array.

## 4. Roles

Roles declare what a node does within an interface. Kippit protocol v1.0.0 defines two roles:

| Role | Description |
|---|---|
| `peer` | Has data, wants data. Participates in data exchange with other peers. |
| `tracker` | Coordinates the network. Knows who is online, helps peers find each other, may relay data. |

Roles are extensible. Extensions MAY define additional roles for the Kippit protocol. Other protocols using KIP as a framework define their own role vocabulary.

### 4.1 Role Capabilities

The `tracker` role in Kippit protocol encompasses what KIP-0001 (Core) calls REGISTRY, SIGNALER, and RELAY. A tracker MAY fulfill all of these or a subset — the specific capabilities are declared via extensions within the interface:

- A tracker with only `kippit-tracker` = REGISTRY
- A tracker with `kippit-tracker` + `webrtc-signaling` = REGISTRY + SIGNALER
- A tracker that also forwards data = REGISTRY + SIGNALER + RELAY

### 4.2 Multiple Roles

A node may fulfill different roles in different interfaces:

```json
{
  "interfaces": [
    { "id": "tracker", "role": "tracker", "extensions": [...] },
    { "id": "seeder", "role": "peer", "extensions": [...] }
  ]
}
```

This replaces the old `"roles": ["peer", "tracker"]` flat array. Roles are now per-interface, tied to the extensions that implement them.

## 5. Extensions

Extensions within an interface list active extensions with their versions and configuration. Extension identification follows KIP-0003 — built-in extensions use short names, external extensions use repository paths.

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
| `name` | string | YES | Extension identifier. Short name = built-in, path with dots = external. See KIP-0003. |
| `version` | string | YES | Extension version (semver). |
| `config` | object | NO | Extension-specific configuration. Schema defined by the extension spec. |

Extensions listed in an interface are **available** in that context. Whether they are **required** for network participants is declared in `rules`.

## 5.1 Rules

The `rules` field contains network governance and participation requirements. Rules are global — they apply to the node, not to a specific interface:

```json
{
  "rules": {
    "governance": "federated",
    "required_extensions": ["jwt-auth", "aes-encryption"]
  }
}
```

| Field | Type | Description |
|---|---|---|
| `governance` | string | `"federated"` (default), `"democratic"`, `"hybrid"`. See KIP-0001 section 3.2. |
| `required_extensions` | string[] | Extension names that participants MUST support. See KIP-0003. |

Extensions MAY define additional rule fields. For example, `jwt-auth` adds `"auth_required": true` to rules. `chunk-exchange` adds `"chunk_size": 2097152`. The rules object is open — unknown fields are ignored by nodes that don't understand them.

## 6. Network Map

The `network` field lists other nodes and services this node knows about:

```json
{
  "network": [
    {
      "url": "wss://tracker.kippit.net/ws",
      "roles": ["tracker"],
      "description": "Kippit main tracker"
    },
    {
      "url": "https://turn.kippit.net",
      "roles": ["relay"],
      "protocol": "TURN",
      "description": "TURN relay for NAT traversal"
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | string | YES | How to reach this node/service. |
| `roles` | string[] | YES | Roles this node fulfills. Empty array for non-KIP services. |
| `protocol` | string | NO | Protocol spoken if not Kippit (e.g. `"TURN"`, `"STUN"`, `"BitTorrent"`). |
| `description` | string | NO | Human-readable description. |
| `manifest` | string | NO | URL to this node's manifest, if different from `url`. |

The network map enables **mesh discovery** — a node joining a network learns about other nodes through manifests. No single node needs to know the entire topology.

## 7. Examples

### 7.1 Kippit Tracker

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "kippit.net",
  "interfaces": [
    {
      "id": "kippit-network",
      "role": "tracker",
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0", "config": {"endpoint": "wss://tracker.kippit.net/ws", "announce_mode": "per-resource", "push_peers": true}},
        {"name": "jwt-auth", "version": "1.0.0", "config": {"method": "ES256", "jwks_url": "https://kippit.net/.well-known/jwks.json"}},
        {"name": "webrtc-signaling", "version": "1.0.0", "config": {"ice_servers": [{"urls": ["stun:stun.kippit.net:3478"]}]}},
        {"name": "resource-catalog", "version": "1.0.0", "config": {"endpoint": "/api/library", "format": "json", "searchable": true}},
        {"name": "resource-metadata", "version": "1.0.0", "config": {"endpoint": "/api/metadata"}}
      ]
    }
  ],
  "rules": {
    "governance": "federated",
    "required_extensions": ["jwt-auth", "aes-encryption", "key-delivery"]
  },
  "network": [
    {"url": "stun:stun.kippit.net:3478", "roles": [], "protocol": "STUN"}
  ]
}
```

### 7.2 NAS Runner (Multi-Interface)

A runner that serves video unencrypted on LAN, encrypted over P2P, and replicates with other NASes:

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "jm-nas",
  "interfaces": [
    {
      "id": "lan-video",
      "role": "peer",
      "description": "LAN video streaming, no encryption",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "hls-streaming", "version": "1.0.0"},
        {"name": "mdns-discovery", "version": "1.0.0", "config": {"port": 9000}},
        {"name": "resource-catalog", "version": "1.0.0"},
        {"name": "resource-metadata", "version": "1.0.0"}
      ]
    },
    {
      "id": "p2p-video",
      "role": "peer",
      "description": "P2P encrypted video via Kippit network",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "kippit-tracker", "version": "1.0.0", "config": {"endpoint": "wss://tracker.kippit.net/ws"}},
        {"name": "jwt-auth", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0", "config": {"cipher": "AES-128-CBC"}},
        {"name": "key-delivery", "version": "1.0.0", "config": {"methods": ["api"]}},
        {"name": "hls-streaming", "version": "1.0.0"},
        {"name": "webrtc-signaling", "version": "1.0.0"}
      ]
    },
    {
      "id": "replication",
      "role": "peer",
      "description": "NAS-to-NAS encrypted replication",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0"},
        {"name": "key-delivery", "version": "1.0.0", "config": {"methods": ["peer-exchange"]}},
        {"name": "sync", "version": "1.0.0", "config": {"history": true, "conflict_strategy": "last-writer-wins"}}
      ]
    }
  ],
  "network": [
    {"url": "wss://tracker.kippit.net/ws", "roles": ["tracker"]},
    {"url": "http://192.168.1.50:9000", "roles": ["peer"], "protocol": "HTTP", "description": "LAN direct"}
  ]
}
```

Note: this runner may also have unpublished interfaces (e.g. a local HLS server on 127.0.0.1:8080) that do not appear in the manifest. The node runs them internally; the manifest only shows what the node wants others to know about.

### 7.3 Phase 0 CLI Tool

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "interfaces": [
    {
      "id": "default",
      "role": "peer"
    }
  ]
}
```

### 7.4 Hybrid BT + Kippit Tracker

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "hybrid-tracker",
  "interfaces": [
    {
      "id": "kippit",
      "role": "tracker",
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0", "config": {"endpoint": "wss://tracker.example.com/ws", "announce_mode": "per-resource"}},
        {"name": "resource-catalog", "version": "1.0.0", "config": {"searchable": true}}
      ]
    },
    {
      "id": "bittorrent",
      "role": "tracker",
      "extensions": [
        {"name": "bt-tracker", "version": "1.0.0", "config": {"protocol": "http", "port": 6969, "scrape": true}}
      ]
    }
  ]
}
```

### 7.5 BT + KIP Bridge Peer

A peer that serves the same data to both BT and KIP networks:

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "bridge-seeder",
  "interfaces": [
    {
      "id": "kip-peer",
      "role": "peer",
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "kippit-tracker", "version": "1.0.0"}
      ]
    },
    {
      "id": "bt-peer",
      "role": "peer",
      "extensions": [
        {"name": "bt-bridge", "version": "1.0.0", "config": {"listen_port": 6881}}
      ]
    }
  ]
}
```

### 7.6 WebRTC Signaling Server

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "interfaces": [
    {
      "id": "signaling",
      "role": "tracker",
      "description": "WebRTC signaling only. No discovery, no relay.",
      "extensions": [
        {"name": "webrtc-signaling", "version": "1.0.0"}
      ]
    }
  ],
  "network": [
    {"url": "stun:stun.kippit.net:3478", "roles": [], "protocol": "STUN"}
  ]
}
```

## 8. Versioning and Compatibility

- **`kip` field:** Spec version. If a node receives a manifest with `kip: 2`, it knows the schema may have changed. Nodes SHOULD be tolerant of unknown fields (ignore them, don't reject).
- **`version` field:** Protocol version (semver). Nodes speaking different protocol versions may be incompatible depending on the extensions they use. Version compatibility is an extension concern, not a manifest concern.
- **Unknown fields:** Nodes MUST ignore fields they don't recognize. This allows extensions and future spec versions to add fields without breaking existing implementations.
