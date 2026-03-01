# KIP — Kippit Improvement Proposals

Specyfikacja protokołu [Kippit](https://github.com/kippit-net) — P2P media platform. KIP to stack protokołów (jak TCP/IP), nie jeden protokół.

## Model warstwowy

```
5. Application    CLI, Web App, Mobile — poza spec
4. Semantics      Extensiony: streaming, sync, auth, encryption
3. Exchange       Wire protocol: handshake, bitfield, request, piece
2. Connection     Transport: WebRTC, TCP, HTTP, in-memory
1. Discovery      Registry: tracker, mDNS, DHT, BT tracker
```

Role: PEER (ma/chce dane), REGISTRY (wie kto ma co), SIGNALER (relay connection setup), RELAY (transparent proxy).

Pełny opis modelu: [protocol-layer-model.md](protocol-layer-model.md)

## Core Protocol

| KIP | Nazwa | Warstwa | Status | Opis |
|---|---|---|---|---|
| [KIP-0001](KIP-0001-wire-protocol.md) | Wire Protocol | 3 (Exchange) | Draft v0.1 | Framing, message types, chunk exchange (PEER ↔ PEER) |
| [KIP-0002](KIP-0002-peer-discovery.md) | Discovery & Signaling | 1+2 (Discovery + Connection) | Draft v0.2 | Discovery interface, KIP tracker protocol, mDNS, signaling (PEER ↔ REGISTRY/SIGNALER) |
| [KIP-0003](KIP-0003-extension-negotiation.md) | Extension Negotiation | 3+1 (Exchange + Discovery) | Draft v0.2 | Extension negotiation PEER ↔ PEER + manifest PEER ↔ REGISTRY |

## Extensions (planned)

| KIP | Nazwa | Warstwa | Opis |
|---|---|---|---|
| KIP-0004 | Auth | 4 (Semantics) | JWT authentication (ES256) |
| KIP-0005 | Encryption | 4 (Semantics) | Per-chunk encryption (AES-128-CBC) |
| KIP-0006 | Streaming | 4 (Semantics) | Sequential priority, HLS manifest |
| KIP-0007 | Sync | 4 (Semantics) | Multi-runner replication |
| KIP-0008 | Messaging | 4 (Semantics) | Real-time small data |
| KIP-0009 | BT Bridge | 3 (Exchange) | BitTorrent wire protocol compatibility |

## Architektura decyzji

- [protocol-layer-model.md](protocol-layer-model.md) — 5-warstwowy model protokołu z rolami
- [tracker-manifest-authority.md](tracker-manifest-authority.md) — tracker jako autorytet federacji, governance model

## Numeracja

- **0001-0003:** Core protocol (stabilny, mały)
- **0004-0099:** Oficjalne extensiony
- **0100+:** Community extensiony

## Status

| Status | Znaczenie |
|---|---|
| Draft | W trakcie pisania, może się zmienić fundamentalnie |
| Review | Kompletny, wymaga przeglądu |
| Accepted | Zatwierdzony, gotowy do implementacji |
| Final | Zaimplementowany i przetestowany |

## Contributing

KIP specs are written in Markdown. Each KIP is a self-contained document defining one aspect of the protocol. To propose a new KIP, open an issue or PR.
