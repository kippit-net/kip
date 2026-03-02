# Extension: bt-tracker

**Version:** 1.0.0
**Layer:** Discovery (1)
**Roles:** tracker

## Purpose

BitTorrent tracker protocol compatibility. Allows a KIP node with the `tracker` role to serve as a BT tracker — accepting announces from standard BT clients and returning peer lists. This is the discovery counterpart to `bt-bridge` (which handles the exchange layer). Together, they allow a KIP network to fully participate in the BitTorrent ecosystem.

## Manifest Contribution

**Config:**
```json
{
  "name": "bt-tracker",
  "version": "1.0.0",
  "config": {
    "protocol": "http",
    "announce_path": "/announce",
    "scrape": true,
    "port": 6969
  }
}
```

- `protocol`: Transport protocol. `"http"` (BEP 3) or `"udp"` (BEP 15). Default: `"http"`.
- `announce_path`: HTTP path for announce requests (HTTP protocol only). Default: `"/announce"`.
- `scrape`: Whether scrape endpoint is available (BEP 48). Default: `true`.
- `port`: Listening port. Default: `6969`.

**Network entries:**
```json
{
  "network": [
    { "url": "http://tracker.example.com:6969/announce", "roles": ["tracker"], "protocol": "BT-HTTP" }
  ]
}
```

## Negotiation

Configuration-only. BT clients don't know about KIP — they just speak the BT tracker protocol. The node is configured to accept BT announces on a port.

## Communication

### HTTP Tracker Protocol (BEP 3)

**Announce:**
```
GET /announce?info_hash=%XX...&peer_id=%XX...&port=6881&uploaded=0&downloaded=0&left=1234&event=started
```

Standard BT announce parameters. The tracker:

1. Maps `info_hash` to internal state (KIP `resource_id` if `bt-bridge` peers exist, or standalone BT state).
2. Records the announcing peer.
3. Returns a bencoded peer list.

Response (compact format):
```
d8:completei5e10:incompletei3e8:intervali1800e5:peers12:...e
```

| Parameter | BEP | Description |
|---|---|---|
| `info_hash` | 3 | 20-byte SHA1 hash identifying the torrent |
| `peer_id` | 3 | 20-byte peer identifier |
| `port` | 3 | Peer's listening port |
| `event` | 3 | `started`, `stopped`, `completed`, or empty |
| `compact` | 23 | If `1`, return compact peer list |

**Scrape (BEP 48):**
```
GET /scrape?info_hash=%XX...
```

Returns swarm statistics (seeders, leechers, completed).

### UDP Tracker Protocol (BEP 15)

Connect → Announce → optional Scrape. Binary protocol on UDP. Implementation follows BEP 15 exactly.

### Bridge between BT and KIP peers

When a node runs both `bt-tracker` and `kippit-tracker`:

- BT peers announce `info_hash` via BT protocol.
- KIP peers announce `resource_id` via `kippit-tracker`.
- The tracker maintains a mapping: `info_hash` ↔ `resource_id`.
- `want` requests from KIP peers can return BT peers (with `connection_hint: "tcp:IP:port"`).
- BT announce requests can return KIP peers that also run `bt-bridge` (with their `listen_port`).

This bridge is transparent — BT clients see a normal tracker, KIP clients see a normal tracker.

### info_hash ↔ resource_id Mapping

The mapping between BT's 20-byte `info_hash` and KIP's 16-byte `resource_id` is implementation-defined. Common strategies:

| Strategy | Description |
|---|---|
| **Lookup table** | Explicit mapping stored in config or database. Most flexible. |
| **Hash derivation** | `resource_id = SHA256(info_hash)[:16]`. One-way, no collision in practice. |
| **Dual identity** | Resource tracked under both identifiers independently. No mapping needed. |

The tracker's config determines which strategy is used. This is NOT mandated by the extension spec — different networks may choose different strategies.

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `protocol` | `http`, `udp` | `http` | N/A (tracker is the authority) | NO |
| `announce_path` | any HTTP path | `"/announce"` | N/A | NO |
| `scrape` | `true`, `false` | `true` | N/A | NO |
| `port` | any valid port | `6969` | N/A | NO |

## Dependencies

- None required at the protocol level. Standalone BT tracker functionality needs no other KIP extensions.
- **`bt-bridge`** — optional. When peers in the network also run `bt-bridge`, the tracker can bridge BT and KIP swarms.
- **`resource-catalog`** — optional. A BT tracker MAY expose a catalog of tracked torrents.

## Example

### Standalone BT tracker

Node acts as a pure BitTorrent tracker. No KIP peer functionality:

```json
{
  "kip": 1,
  "roles": ["tracker"],
  "name": "public-bt-tracker",
  "extensions": [
    { "name": "bt-tracker", "version": "1.0.0", "config": {
      "protocol": "http",
      "port": 6969,
      "scrape": true
    }}
  ]
}
```

### Hybrid tracker (BT + Kippit)

Node serves as tracker for both BT and KIP peers:

```json
{
  "kip": 1,
  "roles": ["tracker"],
  "name": "hybrid-tracker",
  "extensions": [
    { "name": "kippit-tracker", "version": "1.0.0", "config": {
      "endpoint": "wss://tracker.example.com/ws",
      "announce_mode": "per-resource"
    }},
    { "name": "bt-tracker", "version": "1.0.0", "config": {
      "protocol": "http",
      "port": 6969
    }}
  ]
}
```

BT clients connect on port 6969, KIP clients connect via WebSocket. Both see peers from both worlds (when `bt-bridge` is active on peers).
