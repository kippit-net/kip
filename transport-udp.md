# Transport: udp

**Layer:** Transport
**Type identifier:** `udp`

## Description

Raw UDP datagrams. Unreliable, unordered, message-oriented. Used for protocols that manage their own reliability (DHT, custom game protocols) or don't need it (fire-and-forget messages).

## Connection Establishment

UDP is connectionless. There is no handshake at the transport level.

1. **Sender** sends a UDP datagram to `host:port`.
2. **Receiver** processes the datagram. No acknowledgment at the transport level.

Extensions using UDP MUST handle reliability, ordering, and fragmentation themselves if needed.

## Transport Config Fields

```json
{ "type": "udp", "host": "0.0.0.0", "port": 6881 }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"udp"` |
| `host` | string | NO | Hostname or IP to bind/send to. |
| `port` | integer | YES | UDP port number. |

## Characteristics

| Property | Value |
|---|---|
| Reliable | NO — datagrams may be lost, duplicated, or reordered |
| NAT traversal | Partial — UDP hole punching possible with STUN |
| Browser support | NO — browsers cannot send raw UDP |
| Bidirectional | YES (both sides can send datagrams) |
| Stream/datagram | Datagram (message-oriented) |
| Encryption | NO — plaintext. Use DTLS or extension-level encryption. |
| Max datagram size | ~1400 bytes safe (MTU minus headers). Larger requires fragmentation. |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `chunk-exchange` | NO | Stream-oriented protocol, needs reliable transport |
| `bt-bridge` | NO | BT wire protocol is stream-oriented (TCP/uTP) |
| `bt-tracker` | YES | BEP 15 UDP tracker protocol |
| `messaging` | LIMITED | Only for small, fire-and-forget messages. No reliability guarantee. |
| `hls-streaming` | NO | Requires HTTP |
| `resource-catalog` | NO | Requires HTTP |

## Shutdown

No connection to close. Peers simply stop sending datagrams.

## Example

BT tracker UDP announce:

```json
{
  "id": "bt-udp-tracker",
  "role": "tracker",
  "transport": [
    { "type": "udp", "port": 6969 }
  ],
  "extensions": [
    { "name": "bt-tracker", "version": "1.0.0", "config": { "protocol": "udp" } }
  ]
}
```
