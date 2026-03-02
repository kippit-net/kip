# KIP-0002: Manifest

**Status:** Draft
**Version:** 1.0

## Abstract

KIP-0002 defines the **manifest** — a JSON document that every KIP node serves to describe itself. The manifest declares what the node is, what interfaces it exposes, how to communicate with it, what it expects from other participants, and who else it knows.

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
| `rules` | object | NO | Network rules, including peer interface templates. See section 6. |
| `network` | object[] | NO | Known nodes and services. See section 7. |
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
    {
      "id": "default",
      "role": "peer",
      "transport": [{ "type": "tcp", "port": 9000 }]
    }
  ]
}
```

This says: "I exist, I speak Kippit v1.0.0, I'm a peer reachable over TCP on port 9000."

## 3. Interfaces

An **interface** is a communication context. Each interface declares a role, one or more transports, and the extensions active in that context. A node may expose multiple interfaces — the same node can be a peer on LAN over HTTP without encryption, a peer over WebRTC with encryption, and a tracker over WebSocket, simultaneously.

```json
{
  "interfaces": [
    {
      "id": "lan-video",
      "role": "peer",
      "transport": [
        { "type": "http", "host": "192.168.1.50", "port": 9000 }
      ],
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "hls-streaming", "version": "1.0.0"}
      ]
    },
    {
      "id": "p2p-encrypted",
      "role": "peer",
      "transport": [
        { "type": "webrtc", "signaling": "tracker-conn" }
      ],
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0"},
        {"name": "jwt-auth", "version": "1.0.0"}
      ]
    },
    {
      "id": "tracker-conn",
      "role": "peer",
      "transport": [
        { "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
      ],
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0"},
        {"name": "jwt-auth", "version": "1.0.0"}
      ]
    }
  ]
}
```

### 3.1 Interface Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique identifier within this manifest. Free-form string. Other interfaces may reference it (e.g. for signaling). |
| `role` | string | YES | Role this interface fulfills. See section 4. |
| `transport` | object[] | YES | Transport protocols, ordered by priority (first = preferred). See section 5. |
| `description` | string | NO | Human-readable description. |
| `extensions` | object[] | NO | Extensions active in this interface. See section 5.1. |

### 3.2 Interface Semantics

- Each interface is independent. Extensions in one interface do not affect another.
- The same extension MAY appear in multiple interfaces with different configurations.
- Interfaces MAY reference each other by `id` (e.g. a WebRTC interface referencing a WebSocket interface for signaling).
- Interfaces that the node does not want to publish simply do not appear in the manifest. The node runs them without advertising.

## 4. Roles

Roles declare what a node does within an interface. Kippit protocol v1.0.0 defines two roles:

| Role | Description |
|---|---|
| `peer` | Has data, wants data. Participates in data exchange with other peers. |
| `tracker` | Coordinates the network. Knows who is online, helps peers find each other, may relay data. |

Roles are extensible. Extensions MAY define additional roles. Other protocols using KIP as a framework define their own role vocabulary.

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
    { "id": "tracker", "role": "tracker", "transport": [...], "extensions": [...] },
    { "id": "seeder", "role": "peer", "transport": [...], "extensions": [...] }
  ]
}
```

## 5. Transport

Each interface declares its transport protocols as an ordered array. The first entry is preferred; subsequent entries are fallbacks. Two nodes connecting via the same interface negotiate by finding the first mutually supported transport.

```json
{
  "transport": [
    { "type": "utp", "port": 6881, "priority": 1 },
    { "type": "tcp", "port": 6881, "priority": 2 }
  ]
}
```

### 5.1 Transport Types

| Type | Description | Config fields |
|---|---|---|
| `tcp` | Raw TCP connection | `host`, `port` |
| `udp` | Raw UDP datagrams | `host`, `port` |
| `utp` | Micro Transport Protocol (LEDBAT over UDP) | `host`, `port` |
| `http` | HTTP/HTTPS request-response | `host`, `port`, `url` |
| `websocket` | WebSocket persistent connection | `url` |
| `webrtc` | WebRTC data channel (requires signaling) | `signaling` (interface id) |
| `unix` | Unix domain socket | `path` |

Transport types are extensible — external extensions may define additional types.

### 5.2 Transport Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | Transport protocol identifier. |
| `priority` | integer | NO | Preference order. Lower = more preferred. Default: array position. |
| `host` | string | NO | Hostname or IP. |
| `port` | integer | NO | Port number. |
| `url` | string | NO | Full URL (for http, websocket). |
| `path` | string | NO | Filesystem path (for unix sockets). |
| `signaling` | string | NO | Interface `id` used for signaling (for webrtc). |

Unknown fields are ignored. Transport-specific extensions may define additional fields.

### 5.3 Transport Negotiation

When two nodes want to communicate via the same interface type, they compare transport arrays and use the first mutually supported type. If no overlap exists, the connection cannot be established for that interface.

### 5.4 Extension Compatibility

Extensions declare which transports they are compatible with in their spec. An implementation SHOULD validate that the extensions in an interface are compatible with the declared transports. For example:

- `hls-streaming` requires HTTP-capable transport (`http`, `websocket`)
- `chunk-exchange` works on stream-oriented transports (`tcp`, `utp`, `webrtc`, `websocket`)
- `bt-bridge` requires `tcp` or `utp` (BT wire protocol)

## 5.1 Extensions

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

## 6. Rules

The `rules` field contains network governance and participation requirements. Rules are declared by tracker manifests and describe what the network expects from participants.

### 6.1 Global Rules

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
| `required_extensions` | string[] | Extension names that participants MUST support (in at least one interface). |

Extensions MAY define additional rule fields. The rules object is open — unknown fields are ignored.

### 6.2 Peer Interface Templates

A tracker MAY define `peer_interfaces` — templates that describe how peers in the network should configure their interfaces. This is how a tracker communicates "here's how peers talk to each other":

```json
{
  "rules": {
    "governance": "federated",
    "required_extensions": ["jwt-auth", "aes-encryption"],
    "peer_interfaces": [
      {
        "id": "p2p-primary",
        "transport": [
          { "type": "webrtc", "signaling": "tracker" }
        ],
        "required_extensions": ["chunk-exchange", "aes-encryption", "jwt-auth"],
        "optional_extensions": ["hls-streaming", "resource-metadata", "sync"]
      },
      {
        "id": "lan-optional",
        "transport": [
          { "type": "http" }
        ],
        "required_extensions": ["chunk-exchange"],
        "optional_extensions": ["hls-streaming", "resource-catalog", "resource-metadata"]
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Template identifier. Peers use this id for their matching interface. |
| `transport` | object[] | YES | Required transport types for this interface. |
| `required_extensions` | string[] | YES | Extensions peers MUST have in this interface. |
| `optional_extensions` | string[] | NO | Extensions peers MAY have. |

A peer joining the network reads the tracker's `peer_interfaces` and configures its own interfaces to match. The `id` matches — the tracker can verify compliance by comparing the peer's manifest against the templates.

Peer interfaces not listed in the templates are allowed — the tracker only mandates what's in `peer_interfaces`. A peer may have additional interfaces for other purposes (LAN, replication, etc.).

## 7. Network Map

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
      "url": "stun:stun.kippit.net:3478",
      "roles": [],
      "protocol": "STUN",
      "description": "STUN server for NAT traversal"
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

## 8. Examples

### 8.1 Kippit Tracker

The tracker declares its own interface (how peers connect to it) and peer interface templates (how peers should communicate with each other):

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "kippit.net",
  "interfaces": [
    {
      "id": "tracker",
      "role": "tracker",
      "transport": [
        { "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
      ],
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0", "config": {"announce_mode": "per-resource", "push_peers": true}},
        {"name": "jwt-auth", "version": "1.0.0", "config": {"method": "ES256", "jwks_url": "https://kippit.net/.well-known/jwks.json"}},
        {"name": "webrtc-signaling", "version": "1.0.0", "config": {"ice_servers": [{"urls": ["stun:stun.kippit.net:3478"]}]}},
        {"name": "resource-catalog", "version": "1.0.0", "config": {"endpoint": "/api/library", "format": "json", "searchable": true}},
        {"name": "resource-metadata", "version": "1.0.0", "config": {"endpoint": "/api/metadata"}},
        {"name": "endpoint-discovery", "version": "1.0.0"}
      ]
    }
  ],
  "rules": {
    "governance": "federated",
    "required_extensions": ["jwt-auth", "aes-encryption", "key-delivery"],
    "peer_interfaces": [
      {
        "id": "p2p",
        "transport": [{ "type": "webrtc", "signaling": "tracker" }],
        "required_extensions": ["chunk-exchange", "aes-encryption", "jwt-auth", "key-delivery"],
        "optional_extensions": ["hls-streaming", "resource-catalog", "resource-metadata", "sync", "messaging"]
      },
      {
        "id": "lan",
        "transport": [{ "type": "http" }],
        "required_extensions": ["chunk-exchange"],
        "optional_extensions": ["hls-streaming", "resource-catalog", "resource-metadata", "mdns-discovery"]
      }
    ]
  },
  "network": [
    {"url": "stun:stun.kippit.net:3478", "roles": [], "protocol": "STUN"}
  ]
}
```

### 8.2 NAS Runner (Multi-Interface)

A runner that serves video unencrypted on LAN, encrypted over P2P, and replicates with other NASes:

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "name": "jm-nas",
  "interfaces": [
    {
      "id": "tracker-conn",
      "role": "peer",
      "transport": [
        { "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
      ],
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0"},
        {"name": "jwt-auth", "version": "1.0.0", "config": {"method": "ES256"}}
      ]
    },
    {
      "id": "p2p",
      "role": "peer",
      "description": "P2P encrypted video via Kippit network",
      "transport": [
        { "type": "webrtc", "signaling": "tracker-conn" }
      ],
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0", "config": {"cipher": "AES-128-CBC"}},
        {"name": "jwt-auth", "version": "1.0.0"},
        {"name": "key-delivery", "version": "1.0.0", "config": {"methods": ["api"]}},
        {"name": "hls-streaming", "version": "1.0.0"},
        {"name": "resource-metadata", "version": "1.0.0"}
      ]
    },
    {
      "id": "lan",
      "role": "peer",
      "description": "LAN video streaming, no encryption",
      "transport": [
        { "type": "http", "host": "192.168.1.50", "port": 9000 }
      ],
      "extensions": [
        {"name": "hls-streaming", "version": "1.0.0"},
        {"name": "resource-catalog", "version": "1.0.0"},
        {"name": "resource-metadata", "version": "1.0.0"},
        {"name": "mdns-discovery", "version": "1.0.0", "config": {"port": 9000}},
        {"name": "endpoint-discovery", "version": "1.0.0"}
      ]
    },
    {
      "id": "replication",
      "role": "peer",
      "description": "NAS-to-NAS encrypted replication",
      "transport": [
        { "type": "tcp", "port": 9001 },
        { "type": "webrtc", "signaling": "tracker-conn" }
      ],
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "aes-encryption", "version": "1.0.0"},
        {"name": "key-delivery", "version": "1.0.0", "config": {"methods": ["peer-exchange"]}},
        {"name": "sync", "version": "1.0.0", "config": {"history": true, "conflict_strategy": "last-writer-wins"}}
      ]
    }
  ],
  "network": [
    {"url": "wss://tracker.kippit.net/ws", "roles": ["tracker"]}
  ]
}
```

Note: `tracker-conn` is a separate interface for the WebSocket connection to the tracker. The `p2p` and `replication` interfaces reference it via `"signaling": "tracker-conn"`. The runner may also have unpublished interfaces (e.g. a local HLS server on 127.0.0.1:8080) that do not appear in the manifest.

### 8.3 Phase 0 CLI Tool

```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "interfaces": [
    {
      "id": "default",
      "role": "peer",
      "transport": [
        { "type": "tcp", "port": 9000 }
      ]
    }
  ]
}
```

### 8.4 Hybrid BT + Kippit Tracker

A tracker that serves both BT clients and KIP clients:

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
      "transport": [
        { "type": "websocket", "url": "wss://tracker.example.com/ws" }
      ],
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0", "config": {"announce_mode": "per-resource"}},
        {"name": "resource-catalog", "version": "1.0.0", "config": {"searchable": true}}
      ]
    },
    {
      "id": "bittorrent",
      "role": "tracker",
      "transport": [
        { "type": "http", "port": 6969 }
      ],
      "extensions": [
        {"name": "bt-tracker", "version": "1.0.0", "config": {"scrape": true}}
      ]
    }
  ],
  "rules": {
    "peer_interfaces": [
      {
        "id": "kip-peer",
        "transport": [{ "type": "webrtc", "signaling": "kippit" }],
        "required_extensions": ["chunk-exchange"],
        "optional_extensions": ["hls-streaming"]
      },
      {
        "id": "bt-peer",
        "transport": [
          { "type": "utp", "priority": 1 },
          { "type": "tcp", "priority": 2 }
        ],
        "required_extensions": ["bt-bridge"],
        "optional_extensions": []
      }
    ]
  }
}
```

### 8.5 BT + KIP Bridge Peer

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
      "transport": [
        { "type": "webrtc", "signaling": "tracker-conn" }
      ],
      "extensions": [
        {"name": "chunk-exchange", "version": "1.0.0"},
        {"name": "kippit-tracker", "version": "1.0.0"}
      ]
    },
    {
      "id": "tracker-conn",
      "role": "peer",
      "transport": [
        { "type": "websocket", "url": "wss://tracker.example.com/ws" }
      ],
      "extensions": [
        {"name": "kippit-tracker", "version": "1.0.0"}
      ]
    },
    {
      "id": "bt-peer",
      "role": "peer",
      "transport": [
        { "type": "utp", "port": 6881, "priority": 1 },
        { "type": "tcp", "port": 6881, "priority": 2 }
      ],
      "extensions": [
        {"name": "bt-bridge", "version": "1.0.0"}
      ]
    }
  ]
}
```

### 8.6 WebRTC Signaling Server

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
      "transport": [
        { "type": "websocket", "url": "wss://signal.kippit.net/ws" }
      ],
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

## 9. Versioning and Compatibility

- **`kip` field:** Spec version. If a node receives a manifest with `kip: 2`, it knows the schema may have changed. Nodes SHOULD be tolerant of unknown fields (ignore them, don't reject).
- **`version` field:** Protocol version (semver). Nodes speaking different protocol versions may be incompatible depending on the extensions they use. Version compatibility is an extension concern, not a manifest concern.
- **Unknown fields:** Nodes MUST ignore fields they don't recognize. This allows extensions and future spec versions to add fields without breaking existing implementations.
