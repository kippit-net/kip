# KIP — Kippit Improvement Proposals

Protocol framework for peer-to-peer networks. KIP defines the vocabulary and mechanisms that allow any network to describe itself and any node to understand what a network expects.

KIP does not define how data is exchanged, how peers connect, or what security model is used. Those are extensions.

## Core

| KIP | Name | Status | Description |
|---|---|---|---|
| [KIP-0001](KIP-0001-core.md) | Core Protocol | Draft | Nodes, roles, layers, manifests, extensions |
| [KIP-0002](KIP-0002-manifest.md) | Manifest | Draft | Manifest format, roles, network map |
| [KIP-0003](KIP-0003-extensions.md) | Extensions | Draft | Extension identification, spec requirements, registry |

## Built-in Extensions

| Name | Layer | Roles | Status | Description |
|---|---|---|---|---|
| [`chunk-exchange`](ext-chunk-exchange.md) | Exchange | peer | Draft | Chunk-based file transfer between peers |
| [`kippit-tracker`](ext-kippit-tracker.md) | Discovery | tracker, peer | Draft | WebSocket-based peer discovery and signaling |
| [`jwt-auth`](ext-jwt-auth.md) | Semantics | tracker, peer | Draft | JWT authentication (ES256) |
| [`aes-encryption`](ext-aes-encryption.md) | Semantics | peer | Draft | Data encryption (AES) |
| [`hls-streaming`](ext-hls-streaming.md) | Semantics | peer | Draft | Sequential priority, HLS manifest |
| [`sync`](ext-sync.md) | Semantics | peer | Draft | Multi-node replication with change tracking |
| [`messaging`](ext-messaging.md) | Semantics | peer | Draft | Real-time small data |
| [`bt-bridge`](ext-bt-bridge.md) | Exchange | peer | Draft | BitTorrent wire protocol compatibility |
| [`webrtc-signaling`](ext-webrtc-signaling.md) | Connection | tracker, peer | Draft | WebRTC SDP/ICE exchange |
| [`mdns-discovery`](ext-mdns-discovery.md) | Discovery | peer | Draft | LAN zero-config discovery |

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
