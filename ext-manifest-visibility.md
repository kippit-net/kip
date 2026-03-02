# Extension: manifest-visibility

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** tracker, peer

## Purpose

Controls what appears in a node's public manifest. A tracker MAY require peers to hide certain interfaces or strip extension configs from their published manifests. Without this extension, nodes publish everything they want to — this extension lets the network enforce what's visible and what's not.

## Manifest Contribution

**Config (tracker):**
```json
{
  "name": "manifest-visibility",
  "version": "1.0.0",
  "config": {
    "default": "public",
    "unlisted": ["resource-catalog", "resource-metadata", "key-delivery"],
    "hidden": []
  }
}
```

- `default`: Default visibility for extensions not explicitly listed. `"public"` or `"unlisted"`. Default: `"public"`.
- `unlisted`: Array of extension names whose `config` MUST be stripped from the public manifest. The extension name and version still appear (the node declares "I support this"), but connection details are withheld.
- `hidden`: Array of extension names that MUST NOT appear in the public manifest at all. The node runs the extension, but the manifest doesn't mention it.

**Config (peer):**

Peers declare `manifest-visibility` to indicate they respect the tracker's visibility rules:

```json
{
  "name": "manifest-visibility",
  "version": "1.0.0"
}
```

No peer-side config needed — the peer applies the tracker's rules when generating its manifest.

**Rule fields:**
```json
{
  "rules": {
    "visibility": {
      "default": "public",
      "unlisted": ["resource-catalog", "resource-metadata"],
      "hidden": []
    }
  }
}
```

When the tracker sets `visibility` in rules, all peers in the network MUST apply these visibility rules when generating their public manifest.

## Visibility Levels

| Level | Extension in manifest | Config in manifest | How details are shared |
|---|---|---|---|
| `public` | YES | YES | Everything visible in `GET /.well-known/kip/manifest.json` |
| `unlisted` | YES (name + version) | NO (stripped) | Config shared peer-to-peer after auth/handshake |
| `hidden` | NO | NO | Extension exists only in peer-to-peer communication |

### How unlisted details are exchanged

When an extension is `unlisted`, the peer still runs it but doesn't publish the config. Other peers learn the config through:

1. **Tracker relay:** The tracker knows the full config (peers send it during announce/accept). When the tracker sends `peers` responses, it MAY include the peer's stripped configs for matched extensions.
2. **Post-handshake exchange:** After `chunk-exchange` handshake (and optionally `jwt-auth` verification), peers exchange extension configs via extension messages (type 0x80+).
3. **Out-of-band:** API, pre-shared configuration, etc.

The mechanism depends on the network. The tracker's rules MAY specify a preferred method:

```json
{
  "rules": {
    "visibility": {
      "unlisted": ["resource-catalog"],
      "exchange_method": "tracker-relay"
    }
  }
}
```

| Method | Description |
|---|---|
| `tracker-relay` | Tracker includes unlisted configs in `peers` responses. |
| `peer-exchange` | Peers exchange configs after handshake. |
| `out-of-band` | Implementation-defined. |

Default: `peer-exchange`.

## Negotiation

Tracker-mandated. The tracker sets `visibility` in rules. Peers that support `manifest-visibility` apply the rules. Peers that don't support it publish everything — the tracker MAY reject peers that don't support `manifest-visibility` by including it in `required_extensions`.

## Communication

### Peer → Tracker (during accept)

When connecting to a tracker, the peer sends its **full** manifest (all interfaces, all configs) in the `accept` message. The tracker stores this internally. The peer's **public** manifest (served at `/.well-known/kip/manifest.json`) is filtered according to the visibility rules.

### Tracker → Peer (in peers response)

When the tracker responds to a `want` request, it MAY include filtered extension configs for matched peers:

```json
{
  "type": "peers",
  "resource_id": "a1b2c3d4e5f6a7b8",
  "peers": [
    {
      "peer_id": "...",
      "connection_hint": "webrtc:signal-required",
      "unlisted_configs": {
        "resource-catalog": {"endpoint": "/catalog", "format": "json"},
        "resource-metadata": {"endpoint": "/metadata"}
      }
    }
  ]
}
```

The `unlisted_configs` field is only present when `exchange_method` is `tracker-relay`.

### Peer → Peer (post-handshake)

After `chunk-exchange` handshake, peers MAY exchange unlisted configs via extension message:

```json
{
  "type": "visibility-exchange",
  "configs": {
    "resource-catalog": {"endpoint": "/catalog", "format": "json"},
    "resource-metadata": {"endpoint": "/metadata"}
  }
}
```

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `default` | `public`, `unlisted` | `public` | YES | NO |
| `unlisted` | any extension names | `[]` | YES | NO |
| `hidden` | any extension names | `[]` | YES | NO |
| `exchange_method` | `tracker-relay`, `peer-exchange`, `out-of-band` | `peer-exchange` | YES | NO |

## Dependencies

- None required. Works with any combination of extensions.
- **`kippit-tracker`** — recommended for `tracker-relay` exchange method.
- **`chunk-exchange`** — recommended for `peer-exchange` exchange method.
- **`jwt-auth`** — recommended. Exchanging unlisted configs with unauthenticated peers defeats the purpose.

## Example

### Kippit network: hide file listings from public manifest

Tracker rules:
```json
{
  "rules": {
    "required_extensions": ["jwt-auth", "aes-encryption", "manifest-visibility"],
    "visibility": {
      "default": "public",
      "unlisted": ["resource-catalog", "resource-metadata", "key-delivery"],
      "exchange_method": "tracker-relay"
    }
  }
}
```

Peer's **full** config (sent to tracker):
```json
{
  "id": "p2p-video",
  "role": "peer",
  "extensions": [
    {"name": "chunk-exchange", "version": "1.0.0"},
    {"name": "resource-catalog", "version": "1.0.0", "config": {"endpoint": "/catalog"}},
    {"name": "resource-metadata", "version": "1.0.0", "config": {"endpoint": "/metadata"}},
    {"name": "key-delivery", "version": "1.0.0", "config": {"methods": ["api"], "api_endpoint": "/keys"}}
  ]
}
```

Peer's **public** manifest (served at `/.well-known/kip/manifest.json`):
```json
{
  "id": "p2p-video",
  "role": "peer",
  "extensions": [
    {"name": "chunk-exchange", "version": "1.0.0"},
    {"name": "resource-catalog", "version": "1.0.0"},
    {"name": "resource-metadata", "version": "1.0.0"},
    {"name": "key-delivery", "version": "1.0.0"}
  ]
}
```

Configs stripped. Anyone reading the public manifest sees that the peer supports these extensions, but not the endpoints or methods. After auth via tracker, the tracker relays the configs to authorized peers.

### Public network: everything visible

No `manifest-visibility` extension → no rules → everything public. Simple.

### Paranoid network: hide everything except chunk-exchange

```json
{
  "rules": {
    "visibility": {
      "default": "unlisted",
      "hidden": ["resource-catalog", "resource-metadata"]
    }
  }
}
```

Public manifest shows only extension names (no configs), and catalog/metadata don't appear at all. Peers discover everything through post-handshake exchange.
