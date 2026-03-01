# KIP-0002: Discovery & Signaling Protocol (Layer 1 + Layer 2)

**Status:** Draft
**Version:** 0.2
**Component:** spec

## Abstract

KIP-0002 defines two interfaces:
1. **Discovery** (layer 1) — how a PEER finds other PEERs. Who has resource X?
2. **Signaling** (layer 2) — how a PEER establishes a connection with another PEER behind NAT.

The spec defines an **abstract interface** + a **reference implementation** (KIP tracker protocol). Other implementations (BT tracker, mDNS, DHT, static peer list) are acceptable as long as they satisfy the interface.

## 1. Roles

| Role | Description | Speaks Protocol |
|---|---|---|
| PEER | Has data or wants data. Discovery and signaling client. | KIP-0001 (wire) + discovery client |
| REGISTRY | Knows who has what. Responds to discovery queries. | Discovery protocol (server-side) |
| SIGNALER | Relays connection setup messages between PEERs. | Signaling protocol |
| RELAY | Forwards data when P2P is impossible. Transparent proxy. | KIP-0001 (wire) |

A single node may fulfill multiple roles. Our implementation: server = REGISTRY + SIGNALER + RELAY.

## 2. Discovery Interface (Layer 1)

### 2.1 Abstract Interface

Every discovery implementation MUST provide:

```
DiscoveryProvider:
  Announce(peer_id, resources[]) → ack/error
  Want(resource_id) → PeerInfo[]
  Leave(resource_id) → ack/error

PeerInfo:
  peer_id: bytes[16]
  capabilities: string[]       // e.g. ["signaling", "relay", "direct-http"]
  connection_hint: string      // how to connect (address, transport type)
```

Does not mandate a transport. The implementation may be WebSocket, HTTP, UDP, a file on disk, anything.

### 2.2 Discovery Implementations

| Implementation | Transport | Scope | Notes |
|---|---|---|---|
| **KIP tracker** | WebSocket (section 4) | WAN | Reference. Our server. |
| **mDNS/DNS-SD** | UDP multicast | LAN | Zero-config, section 5 |
| **BT tracker** | HTTP (BEP 3) | WAN | Via bridge extension |
| **DHT** | UDP (BEP 5-like) | WAN | Future, decentralized |
| **Static peer list** | File / env var | Any | Dev/tests |

A PEER may use multiple implementations simultaneously (resolver chain, US-0015).

## 3. Signaling Interface (Layer 2)

### 3.1 Abstract Interface

Signaling is needed when two PEERs cannot connect directly (NAT). The SIGNALER relays setup messages.

```
SignalingProvider:
  Signal(to_peer_id, payload) → ack/error

payload: opaque bytes — the spec does not know what's inside.
         WebRTC: SDP offer/answer, ICE candidates.
         Other transport: whatever the connection setup requires.
```

Signaling is **transport-specific**. WebRTC needs SDP/ICE exchange. TCP doesn't need signaling (direct connect). In-memory pipes don't need signaling.

### 3.2 When Signaling is Needed

| Transport (Layer 2) | Requires Signaling? |
|---|---|
| WebRTC data channel | YES — SDP offer/answer + ICE candidates |
| TCP direct (public IP) | NO |
| HTTP LAN (mDNS) | NO |
| In-memory pipe (tests) | NO |
| QUIC (future) | Depends on NAT |

If discovery returns a `connection_hint` with sufficient information for a direct connect, signaling is not needed.

## 4. KIP Tracker Protocol (Reference Implementation)

Implementation of REGISTRY + SIGNALER in a single endpoint. This is what our server speaks.

### 4.1 Transport

Persistent bidirectional connection. Reference: WebSocket (`wss://`/`ws://`). But the spec does not require WebSocket — an implementation may use gRPC streams, TCP with framing, SSE + POST, etc.

### 4.2 Manifest (REGISTRY → PEER)

After establishing the transport connection, the REGISTRY sends a **manifest** — a set of rules governing this network. The REGISTRY is the authority — this is federation, not democracy. The PEER accepts the rules or disconnects.

```json
{
  "type": "manifest",
  "version": 1,
  "rules": {
    "auth_required": false,
    "announce_mode": "per-resource",
    "signaling": "provided",
    "relay": "none",
    "push_peers": false,
    "required_extensions": [],
    "encryption_required": false
  }
}
```

The PEER responds with acceptance:

```json
{
  "type": "accept",
  "version": 1,
  "peer_id": "a1b2c3d4e5f6..."
}
```

#### Manifest Fields

| Field | Values | Description |
|---|---|---|
| `auth_required` | bool | Whether announce requires an auth token (KIP-0004) |
| `announce_mode` | `"per-resource"`, `"per-server"` | Whether peer announces a list of resource_ids or just "I'm a host of servers X, Y" |
| `signaling` | `"provided"`, `"none"` | Whether REGISTRY offers signaling relay (SIGNALER role) |
| `relay` | `"available"`, `"none"` | Whether REGISTRY offers chunk relay (RELAY role) |
| `push_peers` | bool | Whether REGISTRY proactively informs about new peers or waits for want |
| `required_extensions` | int[] | Extensions required to participate in the network |
| `encryption_required` | bool | Whether encryption is mandatory |

**`announce_mode`** solves the 50,000 files problem:
- `"per-resource"`: peer announces a list of resource_ids (like a BT tracker — good for small libraries, sharing)
- `"per-server"`: peer announces itself as a host (e.g. "I'm runner X with servers A, B" — file list available via API or directly from the peer)

Different trackers may have different manifests:

| Tracker | Manifest |
|---|---|
| Ours (kippit.net, phase 1) | auth required, per-server, signaling provided, push |
| Ours (phase 0) | no auth, per-resource, signaling provided, pull |
| Public community | no auth, per-resource, no relay, pull |
| Corporate self-hosted | auth required, encryption required, per-server |

The tracker operator sets the policy. The RFC defines the manifest mechanism, not the specific rules.

#### Governance Model

The manifest MAY contain a `governance` field defining who sets the rules:

| Governance | Meaning |
|---|---|
| `"federated"` | REGISTRY imposes rules. Peer accepts or leaves. **Default.** |
| `"democratic"` | The swarm negotiates rules by consensus. REGISTRY is a bootstrap node, not an authority. |
| `"hybrid"` | REGISTRY proposes defaults, swarm may override on selected fields. |

**Democracy can decide on everything** — including requiring auth, encryption, or transitioning to federation. The swarm votes "auth required" → designates a node to enforce it → that node becomes a de facto federator. Democracy can decide to federate. A federator can hand the vote back to the swarm. Governance is dynamic.

**The limitations of democracy in P2P are not about WHAT can be voted on, but HOW to vote and WHO enforces:**

1. **Sybil attack** — who has the right to vote? If 1 peer = 1 vote, an attacker creates 1,000 peers and outvotes everyone. A mechanism of identity (who is a real peer) requires auth — but auth is a rule we're voting on. Chicken-and-egg problem.

2. **Enforcement** — who kicks out a peer that ignores the vote result? If no one — the rules are non-binding. If a designated node — that node is a de facto federator. Democracy at the moment of enforcement **converges to federation**.

3. **Bootstrap** — at startup there are no peers, no one to vote. Someone must stand up the first node with some initial rules.

In practice, P2P democracy has two forms of enforcement:
- **Mutual enforcement** — every peer enforces rules on its own connections (e.g. "I don't talk to unencrypted peers"). No central enforcer. Works for encryption, bandwidth, chunk verification.
- **Delegated enforcement** — the swarm designates a node to enforce rules. That node becomes a federator with a democratic mandate. Works for auth, moderation, commerce.

The RFC defines the governance mechanism — the choice of model is the operator's/community's decision. Governance can change over the network's lifetime.

### 4.3 Discovery Messages

#### announce (PEER → REGISTRY)

```json
{
  "type": "announce",
  "resources": [
    {
      "resource_id": "deadbeef01234567...",
      "chunk_count": 2048,
      "bitfield": "//////////8="
    }
  ]
}
```

- **resources:** list of seeded resources.
- **bitfield:** base64-encoded. Optional — absence = 100% (full seed).
- **peer_id:** from handshake, not repeated.

A PEER SHOULD send announce after handshake. A PEER MAY re-send with updates.

#### want (PEER → REGISTRY)

```json
{
  "type": "want",
  "resource_id": "deadbeef01234567..."
}
```

The REGISTRY responds with `peers`.

#### peers (REGISTRY → PEER)

```json
{
  "type": "peers",
  "resource_id": "deadbeef01234567...",
  "peers": [
    {
      "peer_id": "f6e5d4c3b2a1...",
      "connection_hint": "webrtc:signal-required"
    }
  ]
}
```

- **connection_hint:** how to connect to the peer. Format depends on transport:
  - `"webrtc:signal-required"` — requires signaling through SIGNALER
  - `"http:192.168.1.50:9000"` — direct HTTP (LAN)
  - `"tcp:1.2.3.4:6881"` — direct TCP (public IP)

#### leave (PEER → REGISTRY)

```json
{
  "type": "leave",
  "resource_id": "deadbeef01234567..."
}
```

Optional — disconnect also cleans up the peer.

### 4.4 Signaling Messages

#### signal (PEER → SIGNALER → PEER)

```json
{
  "type": "signal",
  "to": "f6e5d4c3b2a1...",
  "payload": { }
}
```

SIGNALER:
1. Appends `"from": "<sender_peer_id>"` to the message
2. Forwards to the target PEER without inspecting the payload
3. If the target PEER does not exist → error `PEER_NOT_FOUND`

The payload is **opaque** — the spec does not know what's inside. For WebRTC it's SDP/ICE. For another transport it's whatever the connection setup needs.

### 4.5 Error Messages

```json
{
  "type": "error",
  "code": "PEER_NOT_FOUND",
  "message": "Target peer is not connected"
}
```

| Code | Description |
|---|---|
| PEER_NOT_FOUND | Target signaling peer does not exist |
| INVALID_MESSAGE | Malformed message or missing required fields |
| RATE_LIMITED | Too many messages |
| AUTH_REQUIRED | REGISTRY requires auth, peer did not provide it |
| UNSUPPORTED_VERSION | Protocol version not supported |

### 4.6 State Management

The REGISTRY holds state **exclusively in memory**. Restart = state lost. PEERs re-announce after reconnect.

```
State:
  swarms: Map<resource_id, Set<PeerInfo>>
  connections: Map<peer_id, Connection>

PeerInfo:
  peer_id: bytes[16]
  resources: resource_id[]
  bitfield: Map<resource_id, Uint8Array>
  last_seen: timestamp
  connection_hint: string
```

**Eviction:** PEER with no activity for 90 seconds = removed.

**KeepAlive:** REGISTRY sends ping every 30 seconds. No pong = disconnect.

## 5. mDNS Discovery (LAN)

Discovery interface implementation (section 2) for local networks. Does not require REGISTRY — PEERs announce themselves directly.

### 5.1 Service Announcement

```
Service Type:  _kippit._tcp
Instance Name: <peer_id_hex_first_8_chars>
Port:          defined in config (default 9000)
TXT Records:
  v=1                          (protocol version)
  resources=<res_id1>,<res_id2> (comma-separated, max 10)
```

### 5.2 Discovery Flow

1. PEER sends mDNS query: `_kippit._tcp.local.`
2. LAN PEERs respond with SRV + TXT records
3. Discoverer checks `resources` in TXT
4. If match: direct HTTP connection (no signaling, no wire protocol overhead)

### 5.3 LAN Chunk Endpoint

```
GET /chunks/{resource_id}/{chunk_index}
→ 200 OK, body = raw chunk data
→ 404 Not Found
```

Shortcut — does not require wire protocol handshake (KIP-0001). Simpler, less overhead for LAN.

## 6. Combined Flow (All Layers)

```
PEER A wants resource R:

Layer 1 (DISCOVERY):
  1. mDNS cache → peer B on LAN? → YES → skip to layer 2 direct
  2. REGISTRY: want {resource_id: R}
     REGISTRY: peers {R, [{peer_id: B, hint: "webrtc:signal-required"}]}

Layer 2 (CONNECTION):
  If LAN:
    3a. HTTP GET http://192.168.1.50:9000/chunks/R/0 → done (shortcut)

  If remote (WebRTC):
    3b. SIGNALER: signal {to: B, payload: {type: "offer", sdp: ...}}
        SIGNALER → B: signal {from: A, payload: {type: "offer", sdp: ...}}
        B → SIGNALER: signal {to: A, payload: {type: "answer", sdp: ...}}
        SIGNALER → A: signal {from: B, payload: ...}
        (+ ICE candidates)
    4. WebRTC data channel established → Transport interface ready

Layer 3 (EXCHANGE):
  5. KIP-0001 handshake → Bitfield → Request → Piece → ...

Layer 4 (SEMANTICS):
  6. Extensions: auth verification, decryption, sequential playback, ...
```

## 7. Extensions on Discovery Protocol

KIP-0003 defines extension negotiation for layer 3 (PEER ↔ PEER). The discovery protocol has a **separate** mechanism: **capability advertisement** in handshake (section 4.2).

Examples of discovery extensions:

| Extension | What It Does | Handshake |
|---|---|---|
| KIP-0004 (Auth) | REGISTRY requires JWT in announce | `auth_required: true` |
| Relay | REGISTRY offers chunk relay | `relay_available: true` |
| Push notifications | REGISTRY informs about new peers | `push_peers: true` |
| Federation | REGISTRY syncs with other REGISTRYs | `federation: true` |

Discovery extensions ≠ wire protocol extensions. These are separate namespaces, separate negotiation.

## 8. Out of Scope for KIP-0002

- **Wire protocol** → KIP-0001 (layer 3)
- **Extension negotiation peer ↔ peer** → KIP-0003 (layer 3)
- **Auth details** (JWT format, verification) → KIP-0004
- **Chunk relay implementation** → separate spec or part of KIP-0004/tracker docs
- **Transport implementations** (WebRTC, TCP, QUIC) → outside spec, pluggable

## Open Questions

1. **`want` batch:** should `want` with multiple resource_ids be supported?
2. **Push vs pull:** should REGISTRY proactively inform about new peers (push) or only on `want` (pull)?
3. **Resource limit:** max resources per peer in announce? A peer with 10,000 files = large message.
4. **Discovery + signaling together or separate?** Current draft: together in one connection (simpler). Alternative: separate endpoints (cleaner separation of concerns). Our implementation has a single connection anyway.
5. **connection_hint format:** formalize or keep as a free-form string?
