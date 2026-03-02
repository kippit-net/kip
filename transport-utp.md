# Transport: utp

**Layer:** Transport
**Type identifier:** `utp`

## Description

Micro Transport Protocol (BEP 29). Reliable, ordered transport built on UDP with LEDBAT congestion control. Created by BitTorrent to be "polite" — it yields bandwidth to other applications (TCP, video calls, etc.) instead of competing aggressively.

## Connection Establishment

1. **Initiator** sends a uTP SYN packet (UDP datagram with uTP header) to `host:port`.
2. **Responder** replies with SYN-ACK.
3. Connection is established. Data flows as a reliable stream over UDP datagrams.

The handshake is similar to TCP but over UDP. uTP handles retransmission, ordering, and flow control internally.

## Transport Config Fields

```json
{ "type": "utp", "host": "0.0.0.0", "port": 6881 }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"utp"` |
| `host` | string | NO | Hostname or IP. |
| `port` | integer | YES | UDP port for uTP (shares port space with raw UDP). |

## Characteristics

| Property | Value |
|---|---|
| Reliable | YES — retransmission and ordering built-in |
| NAT traversal | Partial — UDP hole punching possible |
| Browser support | NO — requires native UDP access |
| Bidirectional | YES |
| Stream/datagram | Stream (reliable byte stream over UDP) |
| Encryption | NO — plaintext. Use extension-level encryption. |
| Congestion control | LEDBAT — yields to TCP, polite bandwidth usage |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `chunk-exchange` | YES | Reliable stream, same as TCP |
| `bt-bridge` | YES | BT native transport |
| `sync` | YES | Via chunk-exchange |
| `messaging` | YES | Via chunk-exchange extension messages |
| `jwt-auth` | YES | Extension message after handshake |
| `aes-encryption` | YES | Encrypts chunk-exchange Piece data |
| `hls-streaming` | NO | Requires HTTP |
| `resource-catalog` | NO | Requires HTTP |
| `kippit-tracker` | NO | Requires WebSocket |

## Libraries

| Language | Library | Notes |
|---|---|---|
| Go | `anacrolix/utp` | Pure Go, no cgo, cross-compile friendly |
| C/C++ | `libutp` (BitTorrent official) | Reference implementation |
| Rust | `rust-utp` | Pure Rust |

## Shutdown

Either side sends a FIN packet. Graceful close with acknowledgment. Abrupt disconnects detected by timeout.

## Example

BT peer with uTP primary, TCP fallback:

```json
{
  "id": "bt-data",
  "role": "peer",
  "transport": [
    { "type": "utp", "port": 6881, "priority": 1 },
    { "type": "tcp", "port": 6881, "priority": 2 }
  ],
  "extensions": [
    { "name": "bt-bridge", "version": "1.0.0" }
  ]
}
```
