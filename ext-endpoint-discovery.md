# Extension: endpoint-discovery

**Version:** 1.0.0
**Layer:** Discovery (1)
**Roles:** peer, tracker

## Purpose

Publishes a map of available endpoints on a node — analogous to HTTP's `sitemap.xml` or API documentation. A node may serve files, catalogs, metadata, HLS manifests, key endpoints, and more. This extension provides a single entry point that lists all available endpoints, their purpose, and how to use them. Without it, clients must infer endpoints from extension configs.

## Manifest Contribution

**Config:**
```json
{
  "name": "endpoint-discovery",
  "version": "1.0.0",
  "config": {
    "endpoint": "/endpoints"
  }
}
```

- `endpoint`: HTTP path where the endpoint map is served. Default: `"/endpoints"`.

## Negotiation

Configuration-only. A node either serves an endpoint map or it doesn't.

## Communication

### Request

```
GET /endpoints
```

### Response

```json
{
  "endpoints": [
    {
      "path": "/catalog",
      "method": "GET",
      "extension": "resource-catalog",
      "description": "Browse available resources",
      "auth": false
    },
    {
      "path": "/metadata/{resource_id}",
      "method": "GET",
      "extension": "resource-metadata",
      "description": "Get resource metadata",
      "auth": false
    },
    {
      "path": "/keys/{resource_id}",
      "method": "GET",
      "extension": "key-delivery",
      "description": "Get decryption key for a resource",
      "auth": true
    },
    {
      "path": "/hls/{resource_id}/manifest.m3u8",
      "method": "GET",
      "extension": "hls-streaming",
      "description": "HLS manifest for video playback",
      "auth": false
    },
    {
      "path": "/chunks/{resource_id}/{chunk_index}",
      "method": "GET",
      "extension": "mdns-discovery",
      "description": "Direct chunk download (LAN)",
      "auth": false
    }
  ]
}
```

### Endpoint Entry Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `path` | string | YES | URL path. May contain `{parameters}`. |
| `method` | string | YES | HTTP method (`GET`, `POST`, etc.). |
| `extension` | string | NO | Extension that provides this endpoint. |
| `description` | string | NO | Human-readable description. |
| `auth` | boolean | NO | Whether authentication is required. Default: `false`. |
| `content_type` | string | NO | Response content type (e.g. `"application/json"`, `"application/vnd.apple.mpegurl"`). |

### Error Responses

| Status | Meaning |
|---|---|
| 200 | Success |
| 503 | Endpoint map not ready |

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `endpoint` | any HTTP path | `"/endpoints"` | NO | NO |

## Dependencies

- None required. Works independently of other extensions.
- Lists endpoints from any active extension that exposes HTTP paths.

## Example

### Kippit tracker endpoint map

```
GET /endpoints
```
```json
{
  "endpoints": [
    {
      "path": "/api/library",
      "method": "GET",
      "extension": "resource-catalog",
      "description": "User's file library",
      "auth": true,
      "content_type": "application/json"
    },
    {
      "path": "/api/metadata/{resource_id}",
      "method": "GET",
      "extension": "resource-metadata",
      "auth": true,
      "content_type": "application/json"
    },
    {
      "path": "/.well-known/jwks.json",
      "method": "GET",
      "extension": "jwt-auth",
      "description": "Public key set for JWT verification",
      "auth": false,
      "content_type": "application/json"
    }
  ]
}
```

### Minimal peer (LAN only)

```
GET /endpoints
```
```json
{
  "endpoints": [
    {
      "path": "/chunks/{resource_id}/{chunk_index}",
      "method": "GET",
      "description": "Direct chunk download",
      "auth": false,
      "content_type": "application/octet-stream"
    }
  ]
}
```
