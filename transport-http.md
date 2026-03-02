# Transport: http

**Layer:** Transport
**Type identifier:** `http`

## Description

HTTP/HTTPS request-response. The universal transport — works through firewalls, proxies, CDNs. Supports both plain HTTP and TLS-encrypted HTTPS. Request-response model, not persistent (though HTTP/2 multiplexes and keep-alive reuse connections).

## Connection Establishment

1. **Client** sends an HTTP request to the URL or `host:port` from the transport config.
2. **Server** processes the request and returns an HTTP response.
3. Connection may be kept alive for subsequent requests (HTTP/1.1 keep-alive, HTTP/2 multiplexing) but each exchange is a discrete request-response.

For HTTPS, TLS handshake occurs before the first HTTP request. Certificate verification follows standard rules.

## Transport Config Fields

```json
{ "type": "http", "host": "192.168.1.50", "port": 9000 }
```

```json
{ "type": "http", "url": "https://api.kippit.net" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | YES | `"http"` |
| `host` | string | NO | Hostname or IP. |
| `port` | integer | NO | Port number. Default: 80 (HTTP) or 443 (HTTPS). |
| `url` | string | NO | Full base URL. If provided, `host` and `port` are derived from it. |

Either `url` or `host`+`port` must be provided.

## Characteristics

| Property | Value |
|---|---|
| Reliable | YES — TCP underneath |
| NAT traversal | YES — outbound HTTP works through most NATs and firewalls |
| Browser support | YES — fetch API, XMLHttpRequest |
| Bidirectional | NO — request-response only (client initiates). Use SSE for server push. |
| Stream/datagram | Request-response (discrete messages) |
| Encryption | Optional — HTTPS (TLS) provides transport-level encryption |

## Compatible Extensions

| Extension | Compatible | Notes |
|---|---|---|
| `hls-streaming` | YES | m3u8 manifest and segments served over HTTP |
| `resource-catalog` | YES | `GET /catalog` |
| `resource-metadata` | YES | `GET /metadata/{resource_id}` |
| `endpoint-discovery` | YES | `GET /endpoints` |
| `key-delivery` | YES | `GET /keys/{resource_id}` (api method) |
| `bt-tracker` | YES | `GET /announce` (BEP 3 HTTP tracker) |
| `jwt-auth` | YES | `Authorization: Bearer <token>` header |
| `mdns-discovery` | YES | LAN HTTP chunks endpoint |
| `chunk-exchange` | NO | Binary stream protocol, not request-response |
| `bt-bridge` | NO | BT wire protocol is stream-oriented |
| `sync` | NO | Requires chunk-exchange (stream transport) |

## Shutdown

HTTP connections are closed by the server or client. Keep-alive connections time out. No special KIP-level shutdown needed.

## Example

LAN video server:

```json
{
  "id": "lan",
  "role": "peer",
  "transport": [
    { "type": "http", "host": "192.168.1.50", "port": 9000 }
  ],
  "extensions": [
    { "name": "hls-streaming", "version": "1.0.0" },
    { "name": "resource-catalog", "version": "1.0.0" },
    { "name": "resource-metadata", "version": "1.0.0" },
    { "name": "endpoint-discovery", "version": "1.0.0" }
  ]
}
```
