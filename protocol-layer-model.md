# Idea: Protocol Layer Model

**Status:** Active — current architectural direction

## Context

Attempting to describe the protocol as a single monolith (KIP-0001 + KIP-0002 + KIP-0003) leads to layer confusion. The tracker is described as a "WebSocket JSON API" instead of as a role in the protocol. Extension negotiation only works between peers, not between a peer and a tracker. It's unclear where WebRTC, HLS, BT live — because the model doesn't distinguish layers.

## Idea: KIP is a protocol stack, not a single protocol

Like TCP/IP is not a single protocol, KIP defines layers and interfaces between them.

### Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  5. APPLICATION                                                 │
│     "I'm watching a movie" / "Syncing files" / "Sharing w/ mom"│
│     CLI, Web App, Mobile — NOT in protocol spec                 │
├─────────────────────────────────────────────────────────────────┤
│  4. SEMANTICS                                                   │
│     What the data MEANS. Extensions live here.                  │
│                                                                 │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│     │Streaming │ │  Sync    │ │Messaging │ │   Auth   │ ...   │
│     │HLS/m3u8  │ │op log,  │ │real-time │ │JWT, TLS  │       │
│     │sequential│ │vector   │ │channels  │ │encryption│       │
│     │priority  │ │clocks   │ │          │ │          │       │
│     └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├─────────────────────────────────────────────────────────────────┤
│  3. EXCHANGE                                                    │
│     Handshake, bitfield, request, piece, cancel.                │
│     Transport-agnostic. Binary framing.                         │
│                                                                 │
│     Implementations:                                            │
│     ┌────────────────┐  ┌─────────────────┐                    │
│     │ KIP wire       │  │ BT wire (bridge)│                    │
│     └────────────────┘  └─────────────────┘                    │
├─────────────────────────────────────────────────────────────────┤
│  2. CONNECTION                                                  │
│     How I establish a channel with a peer. NAT traversal here.  │
│                                                                 │
│     ┌────────┐ ┌──────┐ ┌──────┐ ┌──────────┐ ┌────────────┐  │
│     │WebRTC  │ │ TCP  │ │ HTTP │ │ In-memory│ │ QUIC/other │  │
│     │data ch.│ │      │ │ LAN  │ │ (tests)  │ │ (future)   │  │
│     └────────┘ └──────┘ └──────┘ └──────────┘ └────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  1. DISCOVERY                                                   │
│     Where are the peers? Who has what I'm looking for?          │
│                                                                 │
│     ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐ ┌───────────┐ │
│     │KIP      │ │BT        │ │ mDNS │ │ DHT  │ │ static    │ │
│     │tracker  │ │tracker   │ │      │ │      │ │ peer list │ │
│     └──────────┘ └──────────┘ └──────┘ └──────┘ └───────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

Each layer has multiple implementations. The protocol defines interfaces between layers.

### Protocol Roles

A node in the network fulfills roles. A role is a set of capabilities + the protocol it speaks. A role is not a binary.

```
PEER        Has data, wants data. Speaks wire protocol (layer 3).
REGISTRY    Knows who has what. Speaks discovery protocol (layer 1).
SIGNALER    Relays SDP/ICE. Speaks signaling protocol (layer 2).
RELAY       Forwards data when P2P is impossible. Speaks wire protocol (layer 3).
```

Any node can fulfill any combination of roles:

| Binary | Roles | Description |
|---|---|---|
| Runner (ours, Go) | PEER | Seeds/downloads chunks, speaks wire + discovery client |
| Server (ours, Node.js) | REGISTRY + SIGNALER + RELAY | Swarm state, SDP relay, chunk relay, API |
| Fat node (someone's impl) | PEER + REGISTRY | Peer that is also a tracker |
| BT tracker adapter | REGISTRY | Translates KIP discovery ↔ BT tracker HTTP |
| Browser | PEER | WebRTC peer, discovery client |
| CLI tool | PEER | Test peer, in-memory or TCP |

### Our Implementation: 2 Binaries

```
runner (Go)                          server (Node.js)
┌──────────────────────┐            ┌──────────────────────┐
│ PEER                 │            │ REGISTRY             │
│  - seeds chunks      │◄──────────►│  - swarm state       │
│  - downloads chunks  │  discovery │  - peer matching     │
│  - wire protocol     │  protocol  │                      │
│                      │            │ SIGNALER             │
│ (extensions:)        │◄──────────►│  - SDP/ICE relay     │
│  - streaming (HLS)   │  signaling │                      │
│  - encryption (AES)  │  protocol  │ RELAY (optional)     │
│  - auth (JWT)        │            │  - chunk forwarding  │
│  - sync              │            │                      │
│  - bt-bridge         │            │ API (outside proto)  │
│                      │            │  - REST: auth, lib   │
└──────────────────────┘            │  - DB: PostgreSQL    │
                                    └──────────────────────┘
```

But the spec doesn't know about Go or Node.js. The spec defines how PEER talks to REGISTRY and how PEER talks to PEER.

### Interfaces to Define in Spec

| Interface | Who ↔ Who | KIP |
|---|---|---|
| Wire protocol | PEER ↔ PEER | KIP-0001 (exists, draft) |
| Discovery protocol | PEER ↔ REGISTRY | KIP-0002 (to be rewritten) |
| Signaling protocol | PEER ↔ SIGNALER | Part of KIP-0002 or separate KIP |
| Extension negotiation | PEER ↔ PEER, PEER ↔ REGISTRY | KIP-0003 (to be extended) |
| Transport interface | Internal | Not a spec — implementation interface |

RELAY doesn't need a separate spec — it's a transparent proxy on the wire protocol.

### How Existing Standards Fit the Model

```
Standard         Layer      Role in System
─────────        ───────    ───────────────
WebRTC           2          Transport (data channel) + signaling (ICE/STUN)
TCP              2          Transport (BT compat, LAN)
HTTP             2          Transport (LAN chunks) + API (outside protocol)
HLS/m3u8         4          Semantics — streaming extension generates manifest
JWT              4          Semantics — auth extension verifies tokens
AES              4          Semantics — encryption extension encrypts chunks
BT wire          3          Exchange — bridge extension translates ↔ KIP wire
BT tracker       1          Discovery — bridge translates ↔ KIP discovery
mDNS             1          Discovery — LAN, zero-config
STUN/TURN        2          Connection — NAT traversal
```

None of them are "competitors" — they operate at different layers.

### End-to-End Scenarios

**"Watching a movie from my NAS while away from home:"**
```
Discovery:    KIP tracker → "Runner A has resource X"
Connection:   WebRTC (ICE/STUN) → data channel
Exchange:     KIP wire → request chunks
Semantics:    Streaming ext (sequential) + Auth ext (JWT) + Encryption ext (AES)
Application:  HLS.js in the browser
```

**"Downloading a torrent:"**
```
Discovery:    BT tracker → "Peers B,C,D"
Connection:   TCP
Exchange:     BT wire → bridge ext → KIP internal
Semantics:    none — raw download
Application:  file on disk
```

**"Syncing 2 NAS devices:"**
```
Discovery:    KIP tracker → "Runner B online"
Connection:   WebRTC or LAN HTTP
Exchange:     KIP wire
Semantics:    Sync ext (op log) + Auth ext + Encryption ext
Application:  Runner B has a copy
```

### Open Questions

1. **Discovery + signaling: together or separate in spec?** In our implementation it's one WebSocket. But conceptually these are two roles (REGISTRY vs SIGNALER). One KIP or two?

2. **Discovery protocol wire format:** Do we define a concrete format (like KIP-0001 defines binary framing) or an abstract interface + reference implementation?

3. **Capability advertisement:** REGISTRY should tell the peer during handshake: "I require auth, I offer relay, I don't offer signaling." Peer decides whether to proceed.

4. **Extension negotiation on discovery protocol:** KIP-0003 currently only works peer ↔ peer. Should REGISTRY also negotiate extensions? Or is that a separate mechanism (capability advertisement)?

## Related

- [tracker-manifest-authority.md](tracker-manifest-authority.md) — tracker as federation authority
- KIP drafts: [KIP-0001](KIP-0001-wire-protocol.md), [KIP-0002](KIP-0002-peer-discovery.md), [KIP-0003](KIP-0003-extension-negotiation.md)
