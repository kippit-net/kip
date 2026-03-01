# Extension: mdns-discovery

**Version:** 1.0.0
**Layer:** Discovery (1)
**Roles:** peer

## Purpose

LAN zero-config peer discovery via mDNS/DNS-SD. Peers on the same local network find each other without a tracker. Enables direct HTTP connections for chunk transfer — no signaling, no NAT traversal, minimal overhead.

## Manifest Contribution

**Config:**
```json
{
  "name": "mdns-discovery",
  "version": "1.0.0",
  "config": {
    "port": 9000
  }
}
```

- `port`: HTTP port for direct LAN connections. Default: 9000.

**Network entries:** A peer with mDNS MAY list its LAN endpoint:
```json
{
  "url": "http://192.168.1.50:9000",
  "roles": ["peer"],
  "description": "LAN direct access"
}
```

## Negotiation

None. Peers announce themselves on the local network via mDNS. Other peers discover them by querying.

## Communication

### Service Announcement

```
Service Type:  _kippit._tcp
Instance Name: <peer_id first 8 hex chars>
Port:          configured port (default 9000)
TXT Records:
  v=1
  resources=<res_id1>,<res_id2>  (comma-separated, max 10)
```

### Discovery Flow

1. Peer sends mDNS query: `_kippit._tcp.local.`
2. LAN peers respond with SRV + TXT records
3. Discoverer checks `resources` in TXT
4. If match: direct HTTP connection

### LAN Chunk Endpoint

```
GET /chunks/{resource_id}/{chunk_index}
→ 200 OK, body = raw chunk data
→ 404 Not Found
```

Shortcut — does not require `chunk-exchange` handshake. Simpler, lower overhead for LAN.

## Dependencies

None. Works independently of any tracker or signaling mechanism. MAY be used alongside `kippit-tracker` — peer uses mDNS for LAN, tracker for WAN.

## Example

Two NAS devices on the same network. Runner A seeds a movie, Runner B wants to replicate:

1. B queries `_kippit._tcp.local.`
2. A responds: SRV port 9000, TXT resources=abc123...
3. B: `GET http://192.168.1.50:9000/chunks/abc123.../0` → chunk data
4. No tracker, no signaling, no WebRTC. Direct HTTP.
