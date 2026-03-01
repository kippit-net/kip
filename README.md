# KIP — Kippit Improvement Proposals

Protocol framework for peer-to-peer networks. KIP defines the vocabulary and mechanisms that allow any network to describe itself and any node to understand what a network expects.

KIP does not define how data is exchanged, how peers connect, or what security model is used. Those are extensions.

## Core

| KIP | Name | Description |
|---|---|---|
| [KIP-0001](KIP-0001-core.md) | Core Protocol | Nodes, roles, layers, manifests, extensions |

## Extensions (planned)

| KIP | Name | Layer | Description |
|---|---|---|---|
| KIP-0002 | Chunk Exchange | 3 (Exchange) | Chunk-based file transfer between peers |
| KIP-0003 | KIP Tracker | 1 (Discovery) | Tracker-based peer discovery and signaling |
| KIP-0004 | Auth | 4 (Semantics) | JWT authentication (ES256) |
| KIP-0005 | Encryption | 4 (Semantics) | Data encryption (AES) |
| KIP-0006 | Streaming | 4 (Semantics) | Sequential priority, HLS manifest |
| KIP-0007 | Sync | 4 (Semantics) | Multi-node replication |
| KIP-0008 | Messaging | 4 (Semantics) | Real-time small data |
| KIP-0009 | BT Bridge | 3 (Exchange) | BitTorrent wire protocol compatibility |
| KIP-0010 | WebRTC Signaling | 2 (Connection) | WebRTC SDP/ICE exchange |
| KIP-0011 | mDNS Discovery | 1 (Discovery) | LAN zero-config discovery |

## Architecture

- [protocol-layer-model.md](protocol-layer-model.md) — 5-layer model with roles (design document)
- [tracker-manifest-authority.md](tracker-manifest-authority.md) — federation governance model (design document)

## Numbering

- **0001:** Core protocol
- **0002-0099:** Official extensions
- **0100+:** Community extensions

## Status

| Status | Meaning |
|---|---|
| Draft | Work in progress, may change fundamentally |
| Review | Complete, requires review |
| Accepted | Approved, ready for implementation |
| Final | Implemented and tested |

## Contributing

KIP specs are written in Markdown. Each KIP is a self-contained document. To propose a new KIP, open an issue or PR.
