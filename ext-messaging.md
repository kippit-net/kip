# Extension: messaging

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer

## Purpose

Real-time small data exchange between peers. Not for file transfer (that's `chunk-exchange`) — for control messages, status updates, notifications, and any low-latency small payloads that don't fit the chunk model.

## Manifest Contribution

**Config:**
```json
{
  "name": "messaging",
  "version": "1.0.0"
}
```

## Negotiation

Peer-negotiated. Both peers must declare `messaging`. Uses extension messages (0x80+) on the `chunk-exchange` connection.

## Communication

Extension message payload:
```json
{
  "channel": "status",
  "data": { }
}
```

- `channel`: Namespace for the message. Implementations define their own channels.
- `data`: Arbitrary JSON payload.

Max message size: 64 KB. For larger payloads, use `chunk-exchange`.

## Dependencies

- `chunk-exchange` — uses its connection and extension message mechanism.

## Example

Runner sends a "processing complete" notification to a connected peer:

```json
{
  "channel": "processing",
  "data": {
    "resource_id": "deadbeef...",
    "status": "ready",
    "duration": 7200
  }
}
```
