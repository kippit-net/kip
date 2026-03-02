# Transport: webrtc

**Layer:** Transport
**Type identifier:** `webrtc`

## Description

WebRTC data channels. Peer-to-peer transport with NAT traversal via ICE (STUN/TURN). The primary browser-to-browser and browser-to-runner transport. Requires a signaling channel (typically a WebSocket interface) for initial connection setup.

## Connection Establishment

WebRTC connection setup requires a signaling channel — another interface where SDP offers/answers and ICE candidates are exchanged. The `signaling` field references that interface by `id`.

1. **Peer A** creates an RTCPeerConnection and generates an SDP offer.
2. **Peer A** sends the offer through the signaling interface (e.g. `kippit-tracker` `signal` message via WebSocket).
3. **Peer B** receives the offer, creates an RTCPeerConnection, generates an SDP answer.
4. **Peer B** sends the answer back through signaling.
5. Both peers exchange ICE candidates through signaling.
6. ICE negotiation completes — direct P2P connection (or TURN relay fallback) established.
7. Data channel is opened. Binary or text messages flow peer-to-peer.

After connection, the signaling channel is no longer needed for this pair. The tracker exits the data path.

## Transport Config Fields

```json
{ "type": "webrtc", "signaling": "tracker-conn" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"webrtc"` |
| `signaling` | string | YES | Interface `id` used for SDP/ICE exchange. Must be another interface in the same manifest (typically a WebSocket interface with `webrtc-signaling` extension). |

## Characteristics

| Property | Value |
|---|---|
| Reliable | Configurable — data channels can be reliable (ordered) or unreliable (unordered) |
| NAT traversal | YES — ICE with STUN (direct) and TURN (relayed) |
| Browser support | YES — native RTCPeerConnection API |
| Bidirectional | YES |
| Stream/datagram | Message-oriented (data channel messages). Reliable mode emulates stream. |
| Encryption | YES — DTLS mandatory. All WebRTC traffic is encrypted by default. |

## ICE Servers

ICE servers (STUN/TURN) are configured by the `webrtc-signaling` extension in the signaling interface, not in the transport config. The transport only declares that it uses WebRTC and where to signal.

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `chunk-exchange` | YES | Binary framing over reliable data channel. Primary use case. |
| `sync` | YES | Via chunk-exchange |
| `messaging` | YES | Via chunk-exchange extension messages |
| `jwt-auth` | YES | Extension message after chunk-exchange handshake |
| `aes-encryption` | YES | Encrypts Piece data (on top of DTLS) |
| `hls-streaming` | LIMITED | Chunk delivery via chunk-exchange, but m3u8 manifest served separately over HTTP |
| `resource-catalog` | NO | HTTP endpoint |
| `resource-metadata` | LIMITED | Peer-to-peer metadata via extension message (not HTTP) |
| `bt-bridge` | NO | BT wire protocol is TCP-oriented |
| `kippit-tracker` | NO | Tracker uses WebSocket, not WebRTC |

## Shutdown

Either peer closes the data channel and RTCPeerConnection. ICE connection state changes to "disconnected" or "failed" if the remote peer drops.

## Data Channel Configuration

The data channel for `chunk-exchange` SHOULD be created with:
- `ordered: true` — binary framing relies on ordered delivery
- `maxRetransmits: null` — reliable mode (unlimited retransmits)
- `label: "kip"` — channel label convention

For `messaging` without `chunk-exchange`, a separate data channel MAY be created with `label: "kip-msg"`.

## Example

P2P encrypted video exchange:

```json
{
  "id": "p2p",
  "role": "peer",
  "transport": [
    { "type": "webrtc", "signaling": "tracker-conn" }
  ],
  "extensions": [
    { "name": "chunk-exchange", "version": "1.0.0" },
    { "name": "aes-encryption", "version": "1.0.0" },
    { "name": "jwt-auth", "version": "1.0.0" },
    { "name": "hls-streaming", "version": "1.0.0" }
  ]
}
```

The `tracker-conn` interface (WebSocket to tracker) provides the signaling channel:

```json
{
  "id": "tracker-conn",
  "role": "peer",
  "transport": [
    { "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
  ],
  "extensions": [
    { "name": "kippit-tracker", "version": "1.0.0" },
    { "name": "webrtc-signaling", "version": "1.0.0" },
    { "name": "jwt-auth", "version": "1.0.0" }
  ]
}
```
