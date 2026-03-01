# KIP-0002: Manifest

**Status:** Draft
**Version:** 1.0

## Abstract

KIP-0002 defines the **manifest** — a JSON document that every KIP node serves to describe itself. The manifest declares what the node is, what it does, what protocol it speaks, and who else it knows.

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

## 2. Manifest Schema

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "roles": ["peer"],
  "extensions": [],
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
| `roles` | string[] | YES | Roles this node fulfills. See section 3. |
| `extensions` | object[] | NO | Active extensions. See section 4. |
| `rules` | object | NO | Network rules. Contents defined by extensions. |
| `network` | object[] | NO | Known nodes and services. See section 5. |
| `name` | string | NO | Human-readable name for this node. |
| `description` | string | NO | Human-readable description. |

### 2.2 Minimal Manifest

The smallest valid manifest:

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "roles": ["peer"]
}
```

This says: "I exist, I speak Kippit protocol v1.0.0, I'm a peer. I have no extensions, no special rules, and I don't know anyone else."

## 3. Roles

Roles declare what a node does. Kippit protocol v1.0.0 defines two roles:

| Role | Description |
|---|---|
| `peer` | Has data, wants data. Participates in data exchange with other peers. |
| `tracker` | Coordinates the network. Knows who is online, helps peers find each other, may relay data. |

A node may have multiple roles:

```json
{
  "roles": ["peer", "tracker"]
}
```

Roles are extensible. Extensions MAY define additional roles for the Kippit protocol. Other protocols using KIP as a framework define their own role vocabulary.

### 3.1 Role Capabilities

The `tracker` role in Kippit protocol encompasses what KIP-0001 (Core) calls REGISTRY, SIGNALER, and RELAY. A tracker MAY fulfill all of these or a subset — the specific capabilities are declared via extensions and the `network` field:

- A tracker that only does discovery = REGISTRY
- A tracker that also relays connection setup = REGISTRY + SIGNALER
- A tracker that also forwards data = REGISTRY + SIGNALER + RELAY

The manifest doesn't enumerate sub-capabilities as roles. Instead, extensions and the network map make capabilities discoverable.

## 4. Extensions

The `extensions` field lists active extensions with their versions and configuration. Extension identification follows KIP-0003 — built-in extensions use short names, external extensions use repository paths.

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

Extensions listed in the manifest are **available** on this node. Whether they are **required** for participants is declared in `rules`.

## 5. Rules

The `rules` field contains network governance and participation requirements:

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

Extensions MAY define additional rule fields. For example, an auth extension (KIP-0004) might add `"auth_required": true` to rules. A chunk exchange extension might add `"chunk_size": 2097152`. The rules object is open — unknown fields are ignored by nodes that don't understand them.

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
    },
    {
      "url": "https://kippit.net/api/library",
      "roles": [],
      "description": "Content listing API"
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

### 7.1 Kippit Tracker (Phase 1)

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "kippit.net",
  "roles": ["tracker"],
  "extensions": [
    {"name": "kippit-tracker", "version": "1.0.0"},
    {"name": "jwt-auth", "version": "1.0.0", "config": {"method": "ES256"}},
    {"name": "aes-encryption", "version": "1.0.0", "config": {"cipher": "AES-128-CBC"}},
    {"name": "hls-streaming", "version": "1.0.0"},
    {"name": "webrtc-signaling", "version": "1.0.0"}
  ],
  "rules": {
    "governance": "federated",
    "required_extensions": ["jwt-auth", "aes-encryption"]
  },
  "network": [
    {
      "url": "https://turn.kippit.net",
      "roles": ["relay"],
      "protocol": "TURN"
    },
    {
      "url": "https://kippit.net/api/library",
      "roles": [],
      "description": "Content listing and user API"
    }
  ]
}
```

### 7.2 Runner on a NAS (Phase 1)

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "my-nas",
  "roles": ["peer"],
  "extensions": [
    {"name": "chunk-exchange", "version": "1.0.0"},
    {"name": "jwt-auth", "version": "1.0.0"},
    {"name": "aes-encryption", "version": "1.0.0"},
    {"name": "hls-streaming", "version": "1.0.0"},
    {"name": "webrtc-signaling", "version": "1.0.0"},
    {"name": "mdns-discovery", "version": "1.0.0"}
  ],
  "network": [
    {
      "url": "wss://tracker.kippit.net/ws",
      "roles": ["tracker"],
      "manifest": "https://tracker.kippit.net/.well-known/kip/manifest.json"
    }
  ]
}
```

### 7.3 Phase 0 CLI Tool (No Extensions)

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "roles": ["peer"]
}
```

### 7.4 Hypothetical BT-Compatible Tracker

```json
{
  "kip": 1,
  "protocol": "bittorrent",
  "version": "1.0.0",
  "roles": ["tracker"],
  "extensions": [
    {"name": "bt-bridge", "version": "1.0.0"}
  ],
  "rules": {
    "governance": "federated"
  }
}
```

### 7.5 Hypothetical WebRTC Signaling Server

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "roles": ["tracker"],
  "extensions": [
    {"name": "webrtc-signaling", "version": "1.0.0"}
  ],
  "description": "WebRTC signaling only. No discovery, no relay.",
  "network": [
    {
      "url": "stun:stun.kippit.net:3478",
      "roles": [],
      "protocol": "STUN"
    }
  ]
}
```

## 8. Versioning and Compatibility

- **`kip` field:** Spec version. If a node receives a manifest with `kip: 2`, it knows the schema may have changed. Nodes SHOULD be tolerant of unknown fields (ignore them, don't reject).
- **`version` field:** Protocol version (semver). Nodes speaking different protocol versions may be incompatible depending on the extensions they use. Version compatibility is an extension concern, not a manifest concern.
- **Unknown fields:** Nodes MUST ignore fields they don't recognize. This allows extensions and future spec versions to add fields without breaking existing implementations.
