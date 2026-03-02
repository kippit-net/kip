# Transport: websocket

**Layer:** Transport
**Type identifier:** `websocket`

## Description

WebSocket (RFC 6455). Persistent, bidirectional, message-oriented connection over HTTP upgrade. Combines the firewall-friendliness of HTTP with persistent bidirectional communication. The standard transport for real-time signaling and tracker communication.

## Connection Establishment

1. **Client** sends an HTTP Upgrade request to the WebSocket URL.
2. **Server** responds with 101 Switching Protocols.
3. Connection is upgraded to WebSocket. Both sides can send messages (text or binary) at any time.

The initial HTTP handshake traverses firewalls and proxies. After upgrade, the connection is persistent and bidirectional.

## Transport Config Fields

```json
{ "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"websocket"` |
| `url` | string | YES | WebSocket URL (`ws://` or `wss://`). |

## Characteristics

| Property | Value |
|---|---|
| Reliable | YES — TCP underneath |
| NAT traversal | YES — outbound WebSocket works through most NATs |
| Browser support | YES — native WebSocket API |
| Bidirectional | YES — both sides push messages at any time |
| Stream/datagram | Message-oriented (each WebSocket frame is a discrete message) |
| Encryption | Optional — `wss://` uses TLS |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `kippit-tracker` | YES | Primary tracker transport (JSON messages over WebSocket) |
| `webrtc-signaling` | YES | SDP/ICE exchange via tracker WebSocket |
| `jwt-auth` | YES | Token in `accept` message |
| `messaging` | YES | JSON messages |
| `chunk-exchange` | LIMITED | Possible (binary frames) but WebRTC is preferred for peer data exchange |
| `hls-streaming` | NO | Requires HTTP request-response for m3u8 |
| `resource-catalog` | NO | HTTP endpoint |
| `bt-bridge` | NO | BT wire protocol is TCP stream |
| `bt-tracker` | NO | BT tracker uses HTTP GET or UDP |

## Shutdown

Either side sends a WebSocket Close frame (opcode 0x8). Graceful close with status code and optional reason. Connection drops detected by missing pong to ping.

## Ping/Pong

WebSocket has built-in ping/pong frames for keepalive. Extensions may define their own keepalive on top (e.g. `kippit-tracker` 30s ping interval).

## Example

Tracker connection:

```json
{
  "id": "tracker-conn",
  "role": "peer",
  "transport": [
    { "type": "websocket", "url": "wss://tracker.kippit.net/ws" }
  ],
  "extensions": [
    { "name": "kippit-tracker", "version": "1.0.0" },
    { "name": "jwt-auth", "version": "1.0.0" }
  ]
}
```
