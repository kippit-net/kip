# KIP-0001: Core Wire Protocol

**Status:** Draft
**Version:** 0.1
**Component:** spec

## Abstract

KIP-0001 defines the wire protocol for exchanging chunks between peers. Transport-agnostic — works on WebRTC data channels, TCP, in-memory pipes, or anything that has `Send([]byte)` and `Recv() []byte`.

The protocol is binary, message-oriented (not stream-oriented). It does not include auth, encryption, or application semantics (streaming, sync). Those capabilities are provided by extensions (KIP-0004+), negotiated via KIP-0003.

## 1. Terminology

| Term | Definition |
|---|---|
| **Peer** | Protocol participant. Can be a seeder, leecher, or both. |
| **Resource** | Logical set of chunks identified by `resource_id`. Corresponds to a single file. |
| **Chunk** | Transfer unit. Fixed-size bytes (except the last one). |
| **Swarm** | Group of peers exchanging chunks of the same resource. |
| **Bitfield** | Bitmask of chunks that a peer possesses. |

## 2. Identifiers

### 2.1 resource_id

**Format:** 16 bytes (128-bit), represented as a 32-character hex string in JSON/text contexts.

**Generation:** `SHA256(seed_input)[:16]`

Seed input depends on context:
- **Standalone peer (phase 0):** `SHA256(absolute_path)[:16]` — deterministic, changes on rename/move
- **Registered runner (phase 1+):** `SHA256(runner_id + ":" + relative_path)[:16]` — deterministic within a runner

**Decision: path-based, not content-based.**

Rationale:
- Content-addressing (`SHA256(file_data)`) requires reading the entire file at startup — unacceptable for 50GB movies on a slow NAS
- Rename changes resource_id — acceptable. Runner re-scans, tracker gets a new announce, old ID expires. Viewer re-downloads (chunks may still be cached)
- Two runners seeding the same file = two different resource_ids — OK. Dedup at the resource level is not a goal of phases 0-1 (chunk dedup is potentially phase 5+)

**Stability:** resource_id is stable as long as the file is not moved, modified, or deleted. Runner_id is persistent (generated once, stored in config).

**What leaks:** resource_id by itself reveals nothing about the file name, size, type, or content. The tracker only sees an opaque ID + chunk count. This satisfies US-0005.

### 2.2 peer_id

**Format:** 16 bytes (128-bit), hex string.

**Generation:** `random(16)` — random, generated at session start.

**Lifecycle:** ephemeral. New peer_id on every connection to the tracker. Peers are not trackable across sessions.

**Rationale:** persistent peer_id enables tracking (who, when, how often). Ephemeral = privacy-by-default. Reconnect/resume after disconnect doesn't require a persistent ID — the tracker matches by WebSocket connection, not by peer_id.

### 2.3 chunk_index

**Format:** uint32 (0-based).

**Addressing:** a chunk is identified by `(resource_id, chunk_index)`. Not content-addressed (no per-chunk hash in phase 0).

**Chunk size:** configurable per resource. Default: 2 MB. Last chunk may be smaller.

**Future note:** content-addressed chunks (per-chunk hash) are a potential KIP extension for integrity verification and dedup. Not in core — core trusts the transport.

## 3. Framing

Every message has a fixed header:

```
┌──────────┬───────────┬─────────────────────┐
│ type (1B)│ length (4B)│ payload (variable)  │
└──────────┴───────────┴─────────────────────┘
```

- **type:** uint8, identifies the message type (0x00-0xFF)
- **length:** uint32 big-endian, payload size in bytes (excludes header)
- **payload:** bytes specific to the given type

**Max message size:** 4 MB (2MB chunk data + overhead). Messages exceeding the limit = disconnect.

**Byte order:** big-endian (network byte order) for all multi-byte fields.

## 4. Handshake

The handshake initiates a connection. Both peers send the handshake simultaneously (not request-response).

```
Handshake message (type 0x00):
┌──────────┬───────────┬──────────┬───────────┬──────────────┬────────────┐
│ type=0x00│ length    │ version  │ peer_id   │ extensions   │ resources  │
│ (1B)     │ (4B)      │ (2B)     │ (16B)     │ (variable)   │ (variable) │
└──────────┴───────────┴──────────┴───────────┴──────────────┴────────────┘
```

- **version:** uint16, protocol version. Current: `0x0001`.
- **peer_id:** 16 bytes, ephemeral ID of this peer.
- **extensions:** extension list — format defined in KIP-0003.
- **resources:** list of resource_ids that the peer is seeding (may be empty for a leecher).

**Version mismatch:** a peer with an unsupported version sends `Error` (0x09) with code `VERSION_MISMATCH` and disconnects.

## 5. Message Types

### 5.1 Type Table

| Type | Hex | Name | Direction | Description |
|---|---|---|---|---|
| 0 | 0x00 | Handshake | ↔ | Connection initialization |
| 1 | 0x01 | Bitfield | ↔ | Bitmask of possessed chunks |
| 2 | 0x02 | Have | → | "I have a new chunk" |
| 3 | 0x03 | Request | → | "Give me chunk X" |
| 4 | 0x04 | Piece | → | "Here is chunk X" |
| 5 | 0x05 | Cancel | → | "I no longer need chunk X" |
| 6 | 0x06 | KeepAlive | ↔ | Connection keepalive |
| 7 | 0x07 | ResourceUpdate | → | "I have a new/removed resource" |
| 8 | 0x08 | Interested | ↔ | "I'm interested in your resources" |
| 9 | 0x09 | Error | ↔ | Error + disconnect |
| 128+ | 0x80+ | Extension | ↔ | Reserved for KIP extensions |

### 5.2 Bitfield (0x01)

Sent after handshake. One bitfield per resource.

```
┌──────────────────┬───────────────┬───────────────┐
│ resource_id (16B)│ total_chunks  │ bitfield       │
│                  │ (4B, uint32)  │ (ceil(N/8) B)  │
└──────────────────┴───────────────┴───────────────┘
```

Bit 0 = chunk 0, bit 1 = chunk 1, etc. Bit set (1) = chunk possessed.

A peer may send multiple Bitfield messages (one per resource).

### 5.3 Have (0x02)

Informs about a new chunk (incremental update after Bitfield).

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

### 5.4 Request (0x03)

Requests a chunk.

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

The peer SHOULD respond with a Piece or Error message. No response within 30 seconds = timeout; the requester may retry or choose another peer.

**Request queue:** a peer may have a max of 16 pending requests from a single peer. New requests beyond the limit = Error with code `TOO_MANY_REQUESTS`.

### 5.5 Piece (0x04)

Response to a Request.

```
┌──────────────────┬───────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │ data           │
│                  │ (4B, uint32)  │ (variable)     │
└──────────────────┴───────────────┴───────────────┘
```

**Data length:** derived from `length` in the framing header minus 20 bytes (resource_id + chunk_index).

### 5.6 Cancel (0x05)

Cancels a previous Request (peer found the chunk from another peer).

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

The peer SHOULD stop sending the Piece for the canceled chunk. If the Piece is already in transit, Cancel may be ignored.

### 5.7 KeepAlive (0x06)

Empty payload. Sent every 30 seconds if there is no other traffic. No KeepAlive for 90 seconds = peer considered dead → disconnect.

### 5.8 ResourceUpdate (0x07)

Dynamic resource management — peer informs about a new or removed resource.

```
┌───────┬──────────────────┬───────────────┐
│ action│ resource_id (16B)│ total_chunks  │
│ (1B)  │                  │ (4B, uint32)  │
└───────┴──────────────────┴───────────────┘
```

- **action:** `0x01` = ADD (new resource), `0x02` = REMOVE (stopped seeding)
- **total_chunks:** for ADD — how many chunks the resource has. For REMOVE — ignored (0).

After ADD, the peer sends a Bitfield for the new resource.

### 5.9 Interested (0x08)

Declaration of interest in a peer's resources. A peer does not send data until the other peer sends Interested.

```
┌──────────────────┐
│ resource_id (16B)│
└──────────────────┘
```

A peer may send multiple Interested messages (one per resource).

### 5.10 Error (0x09)

```
┌───────────┬──────────────────────┐
│ code (2B) │ message (UTF-8, var) │
└───────────┴──────────────────────┘
```

**Error codes:**

| Code | Hex | Name | Description |
|---|---|---|---|
| 1 | 0x0001 | VERSION_MISMATCH | Unsupported protocol version |
| 2 | 0x0002 | UNKNOWN_RESOURCE | Requested resource does not exist on this peer |
| 3 | 0x0003 | CHUNK_NOT_AVAILABLE | Chunk exists but is not ready yet |
| 4 | 0x0004 | TOO_MANY_REQUESTS | Concurrent request limit exceeded |
| 5 | 0x0005 | INVALID_MESSAGE | Malformed message |
| 6 | 0x0006 | INTERNAL_ERROR | Internal peer error |
| 100+ | 0x0064+ | Extension errors | Reserved for KIP extensions |

**After Error:** the connection is closed. Error is a terminal message.

## 6. Connection Flow

```
Peer A (seeder)                              Peer B (leecher)
     │                                            │
     ├── Handshake {v1, peer_id_A, [res1]} ──────►│
     │◄── Handshake {v1, peer_id_B, []} ──────────┤
     │                                            │
     │   (extension negotiation — KIP-0003)       │
     │                                            │
     ├── Bitfield {res1, 100 chunks, 0xFF...} ───►│
     │                                            │
     │◄── Interested {res1} ──────────────────────┤
     │                                            │
     │◄── Request {res1, chunk 0} ────────────────┤
     ├── Piece {res1, chunk 0, <data>} ──────────►│
     │                                            │
     │◄── Request {res1, chunk 1} ────────────────┤
     │◄── Request {res1, chunk 2} ────────────────┤  (pipelining)
     ├── Piece {res1, chunk 1, <data>} ──────────►│
     ├── Piece {res1, chunk 2, <data>} ──────────►│
     │                                            │
     │   ... (repeat until all chunks) ...        │
     │                                            │
     ├── Have {res1, chunk 100} (new chunk) ─────►│  (incremental)
     │                                            │
```

## 7. Chunk Size

**Default:** 2 MB (2,097,152 bytes).

**Rationale:**
- Too small (64KB like BT) = too much overhead on message framing + bitfield explosion for large files
- Too large (16MB) = too coarse granularity, slow playback start
- 2MB = sweet spot: 4GB movie = 2,000 chunks, bitfield = 250 bytes, reasonable granularity

**Chunk size is per-resource** and communicated in the Bitfield message (total_chunks + resource size allows deriving chunk size). In phase 0 it's fixed (2MB). Configurable chunk size is a potential KIP extension.

## 8. Transport Interface

The protocol is transport-agnostic. The implementation must provide:

```go
type Transport interface {
    Send(msg []byte) error
    Recv() ([]byte, error)
    Close() error
}
```

Specific transports are separate components:
- **WebRTC data channel** (US-0012) — default, NAT traversal
- **HTTP** (LAN) — localhost/mDNS
- **In-memory pipe** (US-0014) — tests

The protocol does NOT manage the transport. It doesn't know how the connection was established (mDNS? STUN? relay?). It receives a Transport interface and exchanges messages.

## 9. Required Behaviors (MUST)

1. A peer MUST send Handshake as the first message.
2. A peer MUST send Bitfield for every resource_id declared in the Handshake.
3. A peer MUST NOT send Piece without a prior Request from the other side.
4. A peer MUST NOT send Request for a resource for which it has not sent Interested.
5. A peer MUST respond to a Request within 30 seconds (Piece or Error).
6. A peer MUST send KeepAlive every 30s when there is no other traffic.
7. A peer MUST close the connection after sending an Error.
8. A peer MUST ignore unknown message types (no Error, no disconnect — enables forward compatibility).
9. A peer MUST respect the max 16 concurrent requests limit.

## 10. Recommended Behaviors (SHOULD)

1. A peer SHOULD use pipelining (multiple Requests before receiving a Piece).
2. A peer SHOULD randomize chunk request order (to avoid hot spots).
3. A peer SHOULD track bandwidth per peer and prefer faster ones.
4. A peer SHOULD react to Cancel by stopping Piece transmission.

## 11. Out of Scope for KIP-0001

- **Peer discovery** → KIP-0002
- **Extension negotiation** → KIP-0003
- **Auth, encryption, streaming** → KIP-0004+
- **Chunk integrity verification** (hashing) → potential KIP extension
- **Choking/unchoking** (BT-style bandwidth allocation) → not needed with 1-5 peers, potentially a KIP extension in the future
- **Piece selection strategy** (rarest-first, sequential) → implementation detail, not spec. Streaming plugin (KIP-0006) overrides to sequential.

## Open Questions

1. Should chunk size be in the Handshake (per-connection) or in the Bitfield (per-resource)?
2. Should ResourceUpdate REMOVE immediately cancel in-flight Requests for that resource?
3. Should Error have an optional resource_id field (per-resource error vs per-connection error)?
