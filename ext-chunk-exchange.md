# Extension: chunk-exchange

**Version:** 1.0.0
**Layer:** Exchange (3)
**Roles:** peer

## Purpose

Chunk-based file transfer between peers. Splits files into fixed-size chunks, tracks which peer has which chunks via bitfields, and transfers chunks via request/response. Transport-agnostic — works on any bidirectional byte channel.

## Manifest Contribution

**Config:**
```json
{
  "name": "chunk-exchange",
  "version": "1.0.0",
  "config": {
    "chunk_size": 2097152,
    "max_concurrent_requests": 16
  }
}
```

- `chunk_size`: Chunk size in bytes. Default: 2 MB (2,097,152). Set by the tracker manifest for network-wide consistency, or per-node for standalone operation.
- `max_concurrent_requests`: Max pending requests from a single peer. Default: 16.

**Rule fields:** A tracker MAY set `chunk_size` in rules to enforce a network-wide chunk size.

## Negotiation

Peer-negotiated. Both peers must declare `chunk-exchange` in their manifests. If only one does, no data exchange occurs.

A tracker MAY require `chunk-exchange` in `required_extensions` — in practice most Kippit networks will.

## Communication

### Identifiers

**resource_id:** 16 bytes (128-bit). Identifies a logical file. Generation is context-dependent:
- Standalone: `SHA256(absolute_path)[:16]`
- Registered runner: `SHA256(runner_id + ":" + relative_path)[:16]`

Path-based, not content-based. Rename changes resource_id — acceptable.

**peer_id:** 16 bytes, random, ephemeral per session. Privacy-by-default.

**chunk_index:** uint32, 0-based. A chunk is addressed by `(resource_id, chunk_index)`.

### Framing

Binary, message-oriented. Every message:

```
┌──────────┬───────────┬─────────────────────┐
│ type (1B)│ length (4B)│ payload (variable)  │
└──────────┴───────────┴─────────────────────┘
```

Big-endian. Max message size: 4 MB.

### Message Types

| Type | Name | Direction | Payload |
|---|---|---|---|
| 0x00 | Handshake | ↔ | version(2B), peer_id(16B), extensions(var), resources(var) |
| 0x01 | Bitfield | ↔ | resource_id(16B), total_chunks(4B), bitfield(var) |
| 0x02 | Have | → | resource_id(16B), chunk_index(4B) |
| 0x03 | Request | → | resource_id(16B), chunk_index(4B) |
| 0x04 | Piece | → | resource_id(16B), chunk_index(4B), data(var) |
| 0x05 | Cancel | → | resource_id(16B), chunk_index(4B) |
| 0x06 | KeepAlive | ↔ | (empty) |
| 0x07 | ResourceUpdate | → | action(1B), resource_id(16B), total_chunks(4B) |
| 0x08 | Interested | ↔ | resource_id(16B) |
| 0x09 | Error | ↔ | code(2B), message(UTF-8, var) |
| 0x80+ | Extension | ↔ | Reserved for other extensions |

### Connection Flow

```
Peer A (seeder)                    Peer B (leecher)
  ├── Handshake ──────────────────►│
  │◄── Handshake ──────────────────┤
  ├── Bitfield {res1} ────────────►│
  │◄── Interested {res1} ─────────┤
  │◄── Request {res1, chunk 0} ───┤
  ├── Piece {res1, chunk 0} ──────►│
  │◄── Request {res1, chunk 1} ───┤
  │◄── Request {res1, chunk 2} ───┤  (pipelining)
  ├── Piece {res1, chunk 1} ──────►│
  ├── Piece {res1, chunk 2} ──────►│
```

### Behaviors

**MUST:**
- Send Handshake as first message
- Send Bitfield for every resource declared in Handshake
- Not send Piece without prior Request
- Not send Request without prior Interested
- Respond to Request within 30 seconds
- Send KeepAlive every 30s when idle
- Close connection after sending Error
- Ignore unknown message types

**SHOULD:**
- Use pipelining
- Randomize chunk request order
- Track bandwidth per peer
- React to Cancel

### Error Codes

| Code | Name |
|---|---|
| 0x0001 | VERSION_MISMATCH |
| 0x0002 | UNKNOWN_RESOURCE |
| 0x0003 | CHUNK_NOT_AVAILABLE |
| 0x0004 | TOO_MANY_REQUESTS |
| 0x0005 | INVALID_MESSAGE |
| 0x0006 | INTERNAL_ERROR |
| 0x0064+ | Reserved for other extensions |

Error is terminal — closes the connection. Extensions that need partial failure handling define their own signaling via extension messages (0x80+).

## Dependencies

None. Core exchange mechanism. Other extensions build on top (hls-streaming overrides chunk selection, aes-encryption wraps chunk data, sync adds change propagation).

## Transport Interface

The extension does not manage transport. It receives a bidirectional byte channel and exchanges messages. How the channel was established (WebRTC, TCP, HTTP, in-memory) is outside this extension's scope.

```go
type Transport interface {
    Send(msg []byte) error
    Recv() ([]byte, error)
    Close() error
}
```

## Example

Phase 0 CLI tool — two peers on LAN, no auth, no encryption:

Manifest:
```json
{
  "kip": 1,
  "protocol": "kippit",
  "version": "1.0.0",
  "roles": ["peer"],
  "extensions": [
    {"name": "chunk-exchange", "version": "1.0.0"}
  ]
}
```

Flow: direct TCP connect → Handshake → Bitfield → Request/Piece → done.
