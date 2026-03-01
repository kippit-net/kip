# Extension: kippit-tracker

**Version:** 1.0.0
**Layer:** Discovery (1)
**Roles:** tracker, peer

## Purpose

WebSocket-based peer discovery and signaling. Defines how a peer connects to a Kippit tracker, announces itself, finds other peers, and optionally uses the tracker for signaling (SDP/ICE relay). This is the Kippit-native way to discover peers — other discovery mechanisms (mDNS, DHT, BT tracker) are separate extensions.

## Manifest Contribution

**Config (tracker):**
```json
{
  "name": "kippit-tracker",
  "version": "1.0.0",
  "config": {
    "endpoint": "wss://tracker.kippit.net/ws",
    "announce_mode": "per-resource",
    "push_peers": true
  }
}
```

- `endpoint`: WebSocket URL for peer connections.
- `announce_mode`: `"per-resource"` (peer lists resource_ids) or `"per-server"` (peer identifies as a host). Default: `"per-resource"`.
- `push_peers`: Whether the tracker proactively pushes new peer info to interested peers. Default: `false`.

**Config (peer):** No config needed — the peer connects to the tracker endpoint from the tracker's manifest.

**Rule fields:** A tracker MAY set `announce_mode` in rules to mandate the announcement strategy.

## Negotiation

Configuration-based. A peer knows its tracker URL (from config, from another manifest's network map, or from user input). The peer fetches the tracker's manifest, checks compatibility, then connects.

## Communication

### Transport

Persistent WebSocket connection (`wss://` or `ws://`). The spec does not mandate WebSocket — an implementation could use gRPC streams, SSE + POST, or any persistent bidirectional channel. WebSocket is the reference transport.

### Connection Flow

```
Peer                                     Tracker
  │                                          │
  │── GET /.well-known/kip/manifest.json ───►│
  │◄── manifest (check required_extensions) ─┤
  │                                          │
  │── WebSocket connect ────────────────────►│
  │◄── manifest (over WS, confirms rules) ──┤
  │── accept {peer_id} ────────────────────►│
  │                                          │
  │── announce {resources} ────────────────►│
  │◄── ack ──────────────────────────────────┤
  │                                          │
  │── want {resource_id} ──────────────────►│
  │◄── peers [{peer_id, connection_hint}] ──┤
  │                                          │
  │   ... signaling, keepalive ...           │
```

### Messages (JSON over WebSocket)

**manifest** (tracker → peer, first message after WS connect):
```json
{
  "type": "manifest",
  "version": 1,
  "rules": { }
}
```
Same content as the HTTP manifest. Sent over WS so the peer doesn't have to fetch it separately if it connected directly.

**accept** (peer → tracker):
```json
{
  "type": "accept",
  "version": 1,
  "peer_id": "a1b2c3d4e5f6..."
}
```

**announce** (peer → tracker):
```json
{
  "type": "announce",
  "resources": [
    {
      "resource_id": "deadbeef01234567...",
      "chunk_count": 2048
    }
  ]
}
```

**want** (peer → tracker):
```json
{
  "type": "want",
  "resource_id": "deadbeef01234567..."
}
```

**peers** (tracker → peer):
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

**leave** (peer → tracker, optional — disconnect also cleans up):
```json
{
  "type": "leave",
  "resource_id": "deadbeef01234567..."
}
```

**signal** (peer → tracker → peer, if `webrtc-signaling` is active):
```json
{
  "type": "signal",
  "to": "f6e5d4c3b2a1...",
  "payload": { }
}
```
Tracker appends `"from"` field and forwards to target peer. Payload is opaque.

**error** (tracker → peer):
```json
{
  "type": "error",
  "code": "PEER_NOT_FOUND",
  "message": "Target peer is not connected"
}
```

Error codes: `PEER_NOT_FOUND`, `INVALID_MESSAGE`, `RATE_LIMITED`, `AUTH_REQUIRED`, `UNSUPPORTED_VERSION`.

### State Management

Tracker holds state in memory. Restart = state lost. Peers re-announce after reconnect.

Eviction: peer with no activity for 90 seconds = removed. Tracker pings every 30 seconds.

### connection_hint

Free-form string. Convention between tracker and peers:
- `"webrtc:signal-required"` — needs signaling via tracker
- `"http:192.168.1.50:9000"` — direct HTTP (LAN)
- `"tcp:1.2.3.4:6881"` — direct TCP

## Dependencies

None for core discovery. If `webrtc-signaling` is active on the tracker, signal messages are relayed through the tracker's WebSocket connection.

## Example

Peer on a NAS joins kippit.net:

1. Runner config has `tracker: wss://tracker.kippit.net/ws`
2. Runner fetches `https://tracker.kippit.net/.well-known/kip/manifest.json`
3. Manifest says: `required_extensions: ["jwt-auth", "aes-encryption"]` — runner has both
4. Runner connects WebSocket, receives manifest, sends accept with peer_id
5. Runner announces its resources
6. When a viewer wants a resource, tracker sends peers → signaling → P2P connection
