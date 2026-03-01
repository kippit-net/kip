# KIP — Kippit Improvement Proposals

Protocol specification for [Kippit](https://github.com/kippit-net) — a P2P media platform. KIP is a protocol stack (like TCP/IP), not a single protocol.

## Layer Model

```
5. Application    CLI, Web App, Mobile — outside of spec
4. Semantics      Extensions: streaming, sync, auth, encryption
3. Exchange       Wire protocol: handshake, bitfield, request, piece
2. Connection     Transport: WebRTC, TCP, HTTP, in-memory
1. Discovery      Registry: tracker, mDNS, DHT, BT tracker
```

Roles: PEER (has/wants data), REGISTRY (knows who has what), SIGNALER (relays connection setup), RELAY (transparent proxy).

Full model description: [protocol-layer-model.md](protocol-layer-model.md)

## Core Protocol

| KIP | Name | Layer | Status | Description |
|---|---|---|---|---|
| [KIP-0001](KIP-0001-wire-protocol.md) | Wire Protocol | 3 (Exchange) | Draft v0.1 | Framing, message types, chunk exchange (PEER ↔ PEER) |
| [KIP-0002](KIP-0002-peer-discovery.md) | Discovery & Signaling | 1+2 (Discovery + Connection) | Draft v0.2 | Discovery interface, KIP tracker protocol, mDNS, signaling (PEER ↔ REGISTRY/SIGNALER) |
| [KIP-0003](KIP-0003-extension-negotiation.md) | Extension Negotiation | 3+1 (Exchange + Discovery) | Draft v0.2 | Extension negotiation PEER ↔ PEER + manifest PEER ↔ REGISTRY |

## Extensions (planned)

| KIP | Name | Layer | Description |
|---|---|---|---|
| KIP-0004 | Auth | 4 (Semantics) | JWT authentication (ES256) |
| KIP-0005 | Encryption | 4 (Semantics) | Per-chunk encryption (AES-128-CBC) |
| KIP-0006 | Streaming | 4 (Semantics) | Sequential priority, HLS manifest |
| KIP-0007 | Sync | 4 (Semantics) | Multi-runner replication |
| KIP-0008 | Messaging | 4 (Semantics) | Real-time small data |
| KIP-0009 | BT Bridge | 3 (Exchange) | BitTorrent wire protocol compatibility |

## Architecture Decisions

- [protocol-layer-model.md](protocol-layer-model.md) — 5-layer protocol model with roles
- [tracker-manifest-authority.md](tracker-manifest-authority.md) — tracker as federation authority, governance model

## Numbering

- **0001-0003:** Core protocol (stable, small)
- **0004-0099:** Official extensions
- **0100+:** Community extensions

## Status

| Status | Meaning |
|---|---|
| Draft | Work in progress, may change fundamentally |
| Review | Complete, requires review |
| Accepted | Approved, ready for implementation |
| Final | Implemented and tested |

## Contributing

KIP specs are written in Markdown. Each KIP is a self-contained document defining one aspect of the protocol. To propose a new KIP, open an issue or PR.
