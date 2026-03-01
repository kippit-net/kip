# Extension: bt-bridge

**Version:** 1.0.0
**Layer:** Exchange (3)
**Roles:** peer

## Purpose

BitTorrent wire protocol compatibility. Allows a KIP node to participate in BT swarms by translating between the KIP internal model and the BT wire protocol. A node with `bt-bridge` can seed/leech torrents alongside native KIP peers.

## Manifest Contribution

**Config:**
```json
{
  "name": "bt-bridge",
  "version": "1.0.0",
  "config": {
    "listen_port": 6881
  }
}
```

- `listen_port`: TCP port for incoming BT connections. Default: 6881.

## Negotiation

Configuration-only. The node is configured to speak BT wire protocol on a TCP port. No KIP-level negotiation — BT peers don't know about KIP.

## Communication

Translates between:
- **BT wire protocol** (BEP 3): handshake, bitfield, request, piece, have, choke/unchoke
- **KIP internal model**: resource_id, chunk_index, chunk data

Mapping:
- BT info_hash ↔ KIP resource_id (via a mapping table or hash conversion)
- BT piece index ↔ KIP chunk_index
- BT block offset/length → KIP assumes full chunks

The bridge handles BT-specific concepts (choking, unchoking, optimistic unchoking) internally — these don't propagate to KIP extensions.

## Dependencies

None at the KIP level. Operates independently of other extensions.

## Example

Runner seeds a file both as a KIP resource and as a torrent:

1. Runner has `chunk-exchange` (for KIP peers) and `bt-bridge` (for BT peers)
2. BT peers connect on port 6881, speak BT wire protocol
3. KIP peers connect via WebRTC, speak chunk-exchange
4. Runner serves both from the same file data
5. BT peers and KIP peers don't know about each other — the runner bridges
