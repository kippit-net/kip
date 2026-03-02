# Extension: resource-metadata

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer, tracker

## Purpose

Provides structured metadata about a specific resource — name, size, type, chunk count, and arbitrary key-value pairs. Separating metadata from the catalog allows nodes to serve lightweight listings (just resource_ids) while offering detailed information on demand.

## Manifest Contribution

**Config:**
```json
{
  "name": "resource-metadata",
  "version": "1.0.0",
  "config": {
    "endpoint": "/metadata"
  }
}
```

- `endpoint`: Base HTTP path for metadata requests. Default: `"/metadata"`. Full path: `{endpoint}/{resource_id}`.

## Negotiation

Configuration-only. A node either serves metadata or it doesn't.

## Communication

### Request

```
GET /metadata/{resource_id}
```

### Response

```json
{
  "resource_id": "a1b2c3d4e5f6a7b8",
  "name": "holiday-2024.mp4",
  "size": 2147483648,
  "type": "video/mp4",
  "chunks": 1024,
  "chunk_size": 2097152,
  "created": "2025-06-15T10:30:00Z",
  "modified": "2025-06-15T10:30:00Z",
  "extra": {}
}
```

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `resource_id` | string | YES | The resource identifier. |
| `name` | string | NO | Human-readable name. May be encrypted or absent for privacy. |
| `size` | integer | NO | Total size in bytes. |
| `type` | string | NO | MIME type. |
| `chunks` | integer | YES | Number of chunks. |
| `chunk_size` | integer | YES | Chunk size in bytes. |
| `created` | string | NO | ISO 8601 timestamp. |
| `modified` | string | NO | ISO 8601 timestamp. |
| `extra` | object | NO | Extension-specific metadata. Open object — unknown keys ignored. |

### Extra Fields Convention

Extensions and implementations MAY add fields to `extra`. Convention: prefix with extension or domain name.

```json
{
  "extra": {
    "hls": {
      "duration": 7200,
      "manifest_url": "/hls/a1b2c3d4e5f6a7b8/manifest.m3u8"
    },
    "thumbnail": {
      "url": "/thumbs/a1b2c3d4e5f6a7b8.jpg"
    },
    "kippit.net": {
      "quality": "1080p",
      "codec": "h264"
    }
  }
}
```

### Error Responses

| Status | Meaning |
|---|---|
| 200 | Success |
| 404 | Unknown resource_id |
| 401 | Auth required (if `jwt-auth` active) |

### Peer-to-Peer Metadata Exchange

When two peers are connected via `chunk-exchange`, metadata MAY also be exchanged as an extension message (type 0x80+) instead of HTTP:

```json
{
  "type": "metadata-request",
  "resource_id": "a1b2c3d4e5f6a7b8"
}
```

```json
{
  "type": "metadata-response",
  "resource_id": "a1b2c3d4e5f6a7b8",
  "name": "holiday-2024.mp4",
  "size": 2147483648,
  "type": "video/mp4",
  "chunks": 1024,
  "chunk_size": 2097152,
  "extra": {}
}
```

This allows metadata exchange without an HTTP server — useful for NAT'd peers communicating over WebRTC.

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `endpoint` | any HTTP path | `"/metadata"` | NO | NO |

**Tracker rules:**

A tracker MAY mandate metadata requirements via rules:

```json
{
  "rules": {
    "metadata_required_fields": ["name", "type", "size"]
  }
}
```

This forces peers to provide at minimum these fields for every resource they announce.

## Dependencies

- None required.
- **`chunk-exchange`** — optional. Enables peer-to-peer metadata exchange via extension messages.
- **`resource-catalog`** — optional. Catalog entries link to metadata via `metadata_url`.
- **`jwt-auth`** — optional. When active, metadata endpoint requires authentication.

## Example

### Video file metadata on a Kippit runner

```
GET /metadata/a1b2c3d4e5f6a7b8
```
```json
{
  "resource_id": "a1b2c3d4e5f6a7b8",
  "name": "holiday-2024.mp4",
  "size": 2147483648,
  "type": "video/mp4",
  "chunks": 1024,
  "chunk_size": 2097152,
  "created": "2025-06-15T10:30:00Z",
  "extra": {
    "hls": {
      "duration": 7200,
      "manifest_url": "/hls/a1b2c3d4e5f6a7b8/manifest.m3u8"
    },
    "thumbnail": {
      "url": "/thumbs/a1b2c3d4e5f6a7b8.jpg"
    }
  }
}
```

### Minimal metadata (hash-only network)

```
GET /metadata/a1b2c3d4e5f6a7b8
```
```json
{
  "resource_id": "a1b2c3d4e5f6a7b8",
  "chunks": 256,
  "chunk_size": 2097152
}
```
