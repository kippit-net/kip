# KIP-0001: Core Protocol

**Status:** Draft
**Version:** 1.0

## Abstract

KIP (Kippit Improvement Proposal) is a protocol framework for peer-to-peer networks. It does not define how data is exchanged, how peers connect, or what security model is used. It defines the **vocabulary** and **mechanisms** that allow any network to describe itself, and any node to understand what a network expects.

The core protocol has three concepts: **nodes** with **roles**, **manifests** that describe networks, and **extensions** that implement behavior.

Everything else — chunk exchange, signaling, auth, encryption, streaming — is an extension.

## 1. Nodes and Roles

A **node** is any participant in a KIP network. A node fulfills one or more **roles**. A role describes what a node does, not how it's implemented.

| Role | Description |
|---|---|
| PEER | Has data, wants data. Participates in exchange. |
| REGISTRY | Knows who is in the network. Answers "who has X?" or "who is online?" |
| SIGNALER | Helps two nodes establish a direct connection (e.g. NAT traversal). |
| RELAY | Forwards data between nodes that cannot connect directly. |

Roles are not exclusive. A single node may be PEER + REGISTRY + SIGNALER + RELAY, or just PEER, or just REGISTRY. The combination depends on the network's needs and the node's configuration.

A role is not a binary, not a service, not an endpoint. It is a **capability declaration**. A node says "I can do these things" and the network decides whether that's useful.

## 2. Layers

KIP networks operate across five conceptual layers. Each layer has multiple possible implementations. No layer mandates a specific technology.

```
5. APPLICATION     What the user sees. Outside of KIP spec.
4. SEMANTICS       What the data means. Extensions define meaning.
3. EXCHANGE        How data moves between nodes. Extensions define format.
2. CONNECTION      How nodes establish channels. Extensions define transport.
1. DISCOVERY       How nodes find each other. Extensions define mechanism.
```

A given network may use any combination of implementations at each layer. A network manifest (section 3) declares which implementations are active.

Existing standards fit naturally into this model — they are not competitors but building blocks:

| Standard | Layer | Example Use |
|---|---|---|
| WebRTC | 2 | Data channel transport + ICE/STUN signaling |
| TCP | 2 | Direct connection (LAN, public IP) |
| HTTP | 2, 5 | LAN transport, API endpoints |
| BitTorrent wire | 3 | Exchange via BT-compatible protocol |
| HLS/m3u8 | 4 | Streaming semantics |
| JWT | 4 | Auth semantics |
| AES | 4 | Encryption semantics |
| mDNS | 1 | LAN discovery |
| DHT | 1 | Decentralized discovery |

## 3. Manifest

A **manifest** is a document that describes a network or a node. It declares: what roles are available, what extensions are active, what rules apply, and what other nodes or services are known.

A manifest is not a protocol message. It is a **description of reality** — it confirms "this is how this network/node operates." A manifest may be served dynamically (e.g. on WebSocket connect), published statically (e.g. as a JSON file at a known URL), or embedded in configuration.

### 3.1 What a Manifest Contains

A manifest has four sections:

**Identity** — who or what is this?
- Node/network name or identifier
- Roles fulfilled (PEER, REGISTRY, SIGNALER, RELAY — any combination)
- Protocol version

**Rules** — how should participants behave?
- Required extensions (participants must support these to join)
- Governance model (federated, democratic, hybrid — see section 3.2)
- Any extension-specific rules (auth required, encryption required, etc.)

**Extensions** — what capabilities are available?
- List of active extensions with versions
- Extension-specific configuration

**Network map** — who else is known?
- References to other nodes and their roles
- Service endpoints (e.g. "content listing at URL X", "TURN relay at Y")
- Other networks this node participates in

### 3.2 Governance

The manifest declares a governance model — who sets the rules.

| Model | Meaning |
|---|---|
| `federated` | The manifest author imposes rules. Participants accept or leave. **Default.** |
| `democratic` | Participants negotiate rules by consensus. The manifest is a starting point. |
| `hybrid` | The manifest author sets defaults, participants may override on specified fields. |

Democracy can decide on everything — including transitioning to federation. Federation can hand control to participants. Governance is dynamic.

Practical limitations of democracy in P2P: sybil attacks (who votes?), enforcement (who kicks violators? — that node becomes a federator), bootstrap (someone must set initial rules). These are not protocol limitations but inherent properties of distributed consensus. The KIP spec does not solve them — extensions and implementations do.

### 3.3 Manifest as Network Map

A manifest is also a window into the network topology. A node's manifest may reference other nodes:

```
"I am a tracker for kippit.net.
 Roles: REGISTRY, SIGNALER.
 I know:
   - turn.kippit.net (RELAY, TURN protocol)
   - cdn.kippit.net (RELAY, HTTP cache)
   - peer-a.local (PEER, mDNS)
 Content listing: https://kippit.net/api/library"
```

```
"I am a runner on a NAS.
 Roles: PEER.
 Networks: kippit.net (via tracker), LAN (via mDNS).
 I serve: video content, listing at port 9000.
 I know: another runner (replication peer)."
```

A node joining a network discovers topology through manifests — each manifest reveals a fragment of the network. There is no requirement for a central registry to know everything.

### 3.4 Manifest Format

The KIP spec does not mandate a manifest format. A manifest may be JSON, YAML, binary, a static file, a dynamic API response, or a negotiated handshake message. What matters is that the content is interpretable by the receiving node.

Extensions MAY define specific manifest formats for their domain (e.g. a tracker extension defines a JSON manifest schema, a BT bridge extension uses BEP-compatible announce format).

## 4. Extensions

An **extension** adds behavior to a KIP network. Everything beyond the core concepts (nodes, roles, manifest, layers) is an extension.

### 4.1 What Extensions Define

An extension is a KIP document (KIP-NNNN) that specifies:

1. **What it does** — the capability it provides (e.g. "chunk-based file exchange", "JWT authentication", "WebRTC signaling")
2. **Which layer(s) it operates on** — discovery, connection, exchange, semantics, or multiple
3. **How nodes negotiate it** — how nodes agree to use this extension (manifest rules, handshake fields, configuration)
4. **How nodes communicate** — message formats, state machines, error handling specific to this extension
5. **What it requires** — dependencies on other extensions, if any

### 4.2 Extension Identification

Extensions are identified by their KIP number. The namespace:

| Range | Purpose |
|---|---|
| 0001 | Core protocol (this document) |
| 0002-0099 | Official extensions |
| 0100+ | Community extensions |

### 4.3 Extension Negotiation

How two nodes agree on which extensions to use depends on the context:

**Node joining a network (PEER → REGISTRY):** The REGISTRY's manifest declares required and available extensions. The PEER either supports them or doesn't join. This is asymmetric — the REGISTRY sets the rules.

**Two peers connecting (PEER ↔ PEER):** Both peers declare their extensions. The active set is the intersection — only extensions supported by both are used. This is symmetric — neither peer has authority over the other.

**Node configured directly:** A node may be configured to use specific extensions without negotiation (e.g. a CLI tool configured as a BT client doesn't need to negotiate — it just speaks BT protocol).

The specific negotiation mechanism (binary handshake, JSON exchange, out-of-band configuration) is defined by the relevant extensions, not by core.

### 4.4 Extension Composability

Extensions are composable. A node may run any combination of extensions simultaneously. For example:

- A Kippit runner: chunk-exchange + hls-streaming + jwt-auth + aes-encryption + kippit-tracker-discovery
- A BT client: bt-wire-exchange + bt-tracker-discovery
- A hybrid node: chunk-exchange + bt-bridge + kippit-tracker-discovery + bt-tracker-discovery
- A WebRTC signaling server: webrtc-signaling (just one extension, role: SIGNALER)
- A LAN video server: chunk-exchange + hls-streaming + mdns-discovery (no auth, no encryption)

The combination is determined by the node's configuration and the networks it participates in.

## 5. What KIP Core Does NOT Define

- **Message formats** — extensions define their own wire formats
- **Transport protocols** — extensions define how nodes connect (WebRTC, TCP, HTTP, etc.)
- **Data semantics** — extensions define what data means (files, streams, messages, etc.)
- **Security model** — extensions define auth, encryption, access control
- **Discovery mechanism** — extensions define how nodes find each other (tracker, mDNS, DHT, etc.)
- **Chunk size, piece selection, bandwidth allocation** — these are extension concerns
- **Specific implementations** (Go, Node.js, etc.) — the spec is language-agnostic

## 6. End-to-End Scenarios

These scenarios illustrate how extensions compose to create different network behaviors. The core protocol provides the framework; extensions provide the behavior.

**Watching a movie from a NAS while away from home:**
```
Discovery:    kippit-tracker extension → "Runner A has resource X"
Connection:   webrtc-signaling extension → data channel established
Exchange:     chunk-exchange extension → request/receive pieces
Semantics:    hls-streaming ext + jwt-auth ext + aes-encryption ext
Application:  HLS.js in the browser (outside KIP spec)
```

**Downloading a torrent:**
```
Discovery:    bt-tracker extension → "Peers B, C, D have this torrent"
Connection:   tcp-connect extension → direct TCP
Exchange:     bt-wire extension → BT protocol messages
Semantics:    none — raw file download
Application:  file on disk (outside KIP spec)
```

**Syncing two NAS devices:**
```
Discovery:    kippit-tracker extension → "Runner B is online"
Connection:   webrtc-signaling or lan-http extension
Exchange:     chunk-exchange extension
Semantics:    sync ext (op log) + jwt-auth ext + aes-encryption ext
Application:  Runner B has a replica (outside KIP spec)
```

**Pure WebRTC signaling server:**
```
Node roles:   SIGNALER only
Extensions:   webrtc-signaling
Manifest:     "I relay SDP/ICE between peers. That's it."
No exchange, no discovery, no semantics.
```
