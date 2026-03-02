# Transport: tcp

**Layer:** Transport
**Type identifier:** `tcp`

## Description

Standard TCP connection. Reliable, ordered, stream-oriented. The most basic transport — works everywhere, no NAT traversal, no special libraries.

## Connection Establishment

1. **Initiator** opens TCP connection to `host:port` from the target interface's transport config.
2. TCP three-way handshake completes.
3. Connection is established. Data flows as a byte stream.

No KIP-level handshake is required at the transport layer. Extension-level handshakes (e.g. `chunk-exchange` Handshake message) happen after TCP is connected.

## Transport Config Fields

```json
{ "type": "tcp", "host": "192.168.1.50", "port": 9001 }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"tcp"` |
| `host` | string | NO | Hostname or IP. If absent, determined by the network entry or peer discovery. |
| `port` | integer | YES | TCP port number. |

## Characteristics

| Property | Value |
|---|---|
| Reliable | YES — TCP guarantees delivery and order |
| NAT traversal | NO — requires open port or port forwarding |
| Browser support | NO — browsers cannot open raw TCP sockets |
| Bidirectional | YES |
| Stream/datagram | Stream (byte-oriented) |
| Encryption | NO — plaintext by default. Use TLS wrapper or extension-level encryption. |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `chunk-exchange` | YES | Binary framing over TCP stream |
| `bt-bridge` | YES | BT wire protocol is TCP-native |
| `sync` | YES | Via chunk-exchange |
| `messaging` | YES | Via chunk-exchange extension messages |
| `jwt-auth` | YES | Extension message after handshake |
| `aes-encryption` | YES | Encrypts chunk-exchange Piece data |
| `hls-streaming` | NO | HLS requires HTTP (use `http` transport) |
| `resource-catalog` | NO | HTTP endpoint (use `http` transport) |
| `resource-metadata` | NO | HTTP endpoint (use `http` transport) |
| `endpoint-discovery` | NO | HTTP endpoint (use `http` transport) |
| `kippit-tracker` | NO | Requires WebSocket |

## Shutdown

Either side may close the TCP connection. Extensions should handle abrupt disconnects (TCP RST) gracefully.

## Example

NAS-to-NAS replication over LAN:

```json
{
  "id": "replication",
  "role": "peer",
  "transport": [
    { "type": "tcp", "host": "192.168.1.51", "port": 9001 }
  ],
  "extensions": [
    { "name": "chunk-exchange", "version": "1.0.0" },
    { "name": "sync", "version": "1.0.0" }
  ]
}
```
