# Transport: unix

**Layer:** Transport
**Type identifier:** `unix`

## Description

Unix domain socket. Inter-process communication on the same machine. No network stack involved — data passes through the kernel directly. Faster than TCP loopback (`localhost`), lower latency, no port conflicts.

## Connection Establishment

1. **Server** creates a Unix socket at the configured path (e.g. `/var/run/kippit.sock`).
2. **Client** connects to the socket path.
3. Connection is established. Data flows as a byte stream (SOCK_STREAM) or datagrams (SOCK_DGRAM).

KIP uses SOCK_STREAM (stream mode) by default — same semantics as TCP.

## Transport Config Fields

```json
{ "type": "unix", "path": "/var/run/kippit.sock" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"unix"` |
| `path` | string | YES | Filesystem path for the socket. |

## Characteristics

| Property | Value |
|---|---|
| Reliable | YES — kernel-managed, same as TCP |
| NAT traversal | N/A — same machine only |
| Browser support | NO |
| Bidirectional | YES |
| Stream/datagram | Stream (SOCK_STREAM, default) |
| Encryption | NO — not needed (same machine, kernel-protected). Process-level access controlled by filesystem permissions. |
| Security | Filesystem permissions (chmod/chown) control who can connect |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `chunk-exchange` | YES | Binary stream, same as TCP |
| `sync` | YES | Via chunk-exchange |
| `messaging` | YES | Via chunk-exchange extension messages |
| `jwt-auth` | OPTIONAL | Same-machine communication may not need auth. Filesystem permissions suffice. |
| `aes-encryption` | OPTIONAL | Same-machine data doesn't need encryption. |
| `hls-streaming` | NO | HTTP endpoint |
| `resource-catalog` | NO | HTTP endpoint |
| `kippit-tracker` | NO | Requires WebSocket |

## Use Cases

- **Runner + server on same VPS:** Runner process communicates with Node.js server via socket. No network exposure, no auth overhead.
- **Docker containers:** Shared volume mount with socket file. Containers communicate without exposing ports.
- **Plugin architecture:** Main process communicates with plugin processes via sockets.

## Shutdown

Either side closes the socket file descriptor. Server removes the socket file on shutdown.

## Example

Runner communicating with local server:

```json
{
  "id": "local-server",
  "role": "peer",
  "transport": [
    { "type": "unix", "path": "/var/run/kippit.sock" }
  ],
  "extensions": [
    { "name": "chunk-exchange", "version": "1.0.0" },
    { "name": "key-delivery", "version": "1.0.0", "config": { "methods": ["peer-exchange"] } }
  ]
}
```

Note: this interface would typically be unpublished (not in the manifest) since it's internal to the machine. Listed here for documentation.
