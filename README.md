# KIP — Kippit Improvement Proposals

Protocol framework for peer-to-peer networks. KIP defines the vocabulary and mechanisms that allow any network to describe itself and any node to understand what a network expects.

KIP does not define how data is exchanged, how peers connect, or what security model is used. Those are extensions.

## Core

| KIP | Name | Status | Description |
|---|---|---|---|
| [KIP-0001](KIP-0001-core.md) | Core Protocol | Draft | Nodes, roles, layers, manifests, extensions |
| [KIP-0002](KIP-0002-manifest.md) | Manifest | Draft | Manifest format, roles, network map |
| [KIP-0003](KIP-0003-extensions.md) | Extensions | Draft | Extension identification, spec requirements, registry |

## Built-in Extensions (planned)

| Name | Layer | Roles | Description |
|---|---|---|---|
| `chunk-exchange` | Exchange | peer | Chunk-based file transfer between peers |
| `kippit-tracker` | Discovery | tracker, peer | WebSocket-based peer discovery and signaling |
| `jwt-auth` | Semantics | tracker, peer | JWT authentication (ES256) |
| `aes-encryption` | Semantics | peer | Data encryption (AES) |
| `hls-streaming` | Semantics | peer | Sequential priority, HLS manifest |
| `sync` | Semantics | peer | Multi-node replication |
| `messaging` | Semantics | peer | Real-time small data |
| `bt-bridge` | Exchange | peer | BitTorrent wire protocol compatibility |
| `webrtc-signaling` | Connection | tracker, peer | WebRTC SDP/ICE exchange |
| `mdns-discovery` | Discovery | peer | LAN zero-config discovery |

External extensions use repository paths (e.g. `github.com/someone/cool-extension`). See KIP-0003.

## Architecture

- [protocol-layer-model.md](protocol-layer-model.md) — 5-layer model with roles (design document)
- [tracker-manifest-authority.md](tracker-manifest-authority.md) — federation governance model (design document)

## Status

| Status | Meaning |
|---|---|
| Draft | Work in progress, may change fundamentally |
| Review | Complete, requires review |
| Accepted | Approved, ready for implementation |
| Final | Implemented and tested |

## Contributing

KIP specs are written in Markdown. Each KIP is a self-contained document. To propose a new KIP, open an issue or PR.
