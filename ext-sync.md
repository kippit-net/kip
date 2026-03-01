# Extension: sync

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer

## Purpose

Multi-node replication with change tracking. Maintains an operation log (op log) with vector clocks so that multiple peers can keep synchronized copies of a resource set. Handles: new files, modified files, deleted files, conflict resolution, and partial sync.

## Manifest Contribution

**Config:**
```json
{
  "name": "sync",
  "version": "1.0.0",
  "config": {
    "history": true,
    "conflict_strategy": "last-writer-wins"
  }
}
```

- `history`: Whether to keep history of deleted/modified files. When `true`, a delete on the source is recorded but replicas retain the data until explicitly purged. Default: `true`.
- `conflict_strategy`: How to resolve concurrent edits. `"last-writer-wins"` (default), `"keep-both"`, or a custom strategy name.

## Negotiation

Peer-negotiated. Both peers must declare `sync`. After `chunk-exchange` handshake, they exchange op logs to determine what's different.

## Communication

### Op Log

Each peer maintains an ordered log of operations on its resources:

```json
{
  "type": "op-log",
  "node_id": "runner-abc123",
  "clock": {"runner-abc123": 42, "runner-def456": 37},
  "ops": [
    {
      "seq": 42,
      "action": "add",
      "resource_id": "deadbeef...",
      "path": "encrypted:a1b2c3...",
      "chunks": 2048,
      "timestamp": "2026-03-01T12:00:00Z"
    },
    {
      "seq": 43,
      "action": "modify",
      "resource_id": "deadbeef...",
      "path": "encrypted:a1b2c3...",
      "chunks": 2050,
      "timestamp": "2026-03-01T12:05:00Z"
    },
    {
      "seq": 44,
      "action": "delete",
      "resource_id": "cafebabe...",
      "path": "encrypted:d4e5f6...",
      "timestamp": "2026-03-01T12:10:00Z"
    }
  ]
}
```

- `node_id`: Persistent identifier for this sync node (not ephemeral peer_id — sync needs persistence across sessions).
- `clock`: Vector clock. Maps node_id → latest known sequence number.
- `path`: Encrypted file path. Replicas don't need to know the plaintext path — they just store encrypted blobs.
- `action`: `"add"`, `"modify"`, `"delete"`.

### Sync Flow

```
Runner A (source)                    Runner B (replica)
  │                                       │
  │◄── sync-request {clock: {A:0, B:0}} ─┤
  │                                       │
  │── sync-response {                     │
  │     ops: [all ops since B's clock],   │
  │     clock: {A:44, B:37}              │
  │   } ─────────────────────────────────►│
  │                                       │
  │   B applies ops:                      │
  │     add → chunk-exchange to get data  │
  │     modify → chunk-exchange for diff  │
  │     delete → mark as deleted          │
  │          (keep data if history=true)  │
  │                                       │
  │◄── sync-ack {clock: {A:44, B:38}} ───┤
```

### Delete Behavior

When `history: true`:
- Source sends the full file data, then the delete operation
- Replica stores the data and marks the resource as deleted
- Replica retains the data until explicit purge (future op or manual action)
- This enables "undo delete" and "audit trail" use cases

When `history: false`:
- Source sends delete operation
- Replica removes the data
- In-flight requests for deleted resources time out normally

### Conflict Resolution

When two nodes modify the same resource concurrently (detected by vector clock comparison):

| Strategy | Behavior |
|---|---|
| `last-writer-wins` | Higher timestamp wins. Loser's version is discarded (or kept in history). |
| `keep-both` | Both versions are kept as separate resources. User resolves manually. |

### What Syncs, What Doesn't

| Data | Syncs? | Notes |
|---|---|---|
| Encrypted chunks | YES | Replicas store encrypted blobs |
| Op log | YES | Change history propagated |
| Original files (unencrypted) | NO | Only the source has originals |
| Encryption keys | NO | Keys come from API (out-of-band) |
| File metadata (name, size) | Encrypted in path | Replicas see opaque identifiers |

## Dependencies

- `chunk-exchange` — sync determines WHAT to transfer, chunk-exchange handles HOW.

## Example

NAS replication — Runner A (home NAS) and Runner B (VPS backup):

1. A and B connect, both have `sync` active
2. B sends sync-request with its clock: `{A: 40, B: 10}`
3. A responds with ops 41-44 (new files added, one deleted)
4. B requests missing chunks via chunk-exchange
5. B applies delete (keeps data because `history: true`)
6. B updates its clock: `{A: 44, B: 11}`
7. Next sync: B sends clock `{A: 44, B: 11}`, A has no new ops → nothing to sync
