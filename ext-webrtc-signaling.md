# Extension: webrtc-signaling

**Version:** 1.0.0
**Layer:** Connection (2)
**Roles:** tracker, peer

## Purpose

WebRTC SDP/ICE exchange for NAT traversal. Allows two peers behind NATs to establish a direct WebRTC data channel by relaying signaling messages through a third party (typically the tracker).

## Manifest Contribution

**Config (tracker):**
```json
{
  "name": "webrtc-signaling",
  "version": "1.0.0",
  "config": {
    "ice_servers": [
      {"urls": "stun:stun.kippit.net:3478"},
      {"urls": "turn:turn.kippit.net:3478", "username": "...", "credential": "..."}
    ]
  }
}
```

- `ice_servers`: ICE server configuration provided to peers. Peers use these for NAT traversal.

**Network entries (tracker):** A tracker MAY list STUN/TURN servers in its network map:
```json
{
  "url": "stun:stun.kippit.net:3478",
  "roles": [],
  "protocol": "STUN"
}
```

## Negotiation

Configuration-based. If both the tracker and the peer have `webrtc-signaling`, the tracker's signal messages (from `kippit-tracker`) carry WebRTC payloads. No separate negotiation — the signaling channel is the tracker's WebSocket.

## Communication

Uses the `signal` message from `kippit-tracker`:

```
Peer A                    Tracker                   Peer B
  │                          │                          │
  ├── signal {to:B,          │                          │
  │    payload: SDP offer} ─►│── signal {from:A,        │
  │                          │    payload: SDP offer} ──►│
  │                          │                          │
  │                          │◄── signal {to:A,         │
  │    signal {from:B,       │    payload: SDP answer} ─┤
  │◄── payload: SDP answer} ─┤                          │
  │                          │                          │
  │   ... ICE candidates ... │                          │
  │                          │                          │
  │◄═══════ WebRTC data channel established ═══════════►│
```

After the data channel is established, peers communicate directly using `chunk-exchange` (or any other exchange extension). The tracker is no longer in the data path.

### Payload Format

The signal payload is **opaque** to the tracker — it just forwards. For WebRTC, the payload contains standard SDP and ICE objects:

```json
{
  "type": "offer",
  "sdp": "v=0\r\no=- ..."
}
```

```json
{
  "type": "candidate",
  "candidate": "candidate:... udp ..."
}
```

## Dependencies

- `kippit-tracker` — uses the tracker's WebSocket as the signaling channel. Without a tracker, signaling requires an alternative relay.

## Example

Peer A (NAS at home) and Peer B (browser on mobile) are both connected to the tracker. B wants to watch A's video:

1. Tracker tells B: peer A has the resource, hint: `webrtc:signal-required`
2. B creates RTCPeerConnection with ICE servers from tracker manifest
3. B sends SDP offer via tracker signal
4. A receives, creates answer, sends via tracker signal
5. ICE candidates exchanged
6. Data channel opens — B starts chunk-exchange with A directly
7. Tracker is out of the loop
