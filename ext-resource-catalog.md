# Extension: resource-catalog

**Version:** 1.0.0
**Layer:** Discovery (1)
**Roles:** peer, tracker

## Purpose

Enables a node to publish a browseable list of resources it holds or knows about. Different networks need different catalog styles — a NAS lists files by name, a torrent tracker lists info hashes, a CDN lists content IDs. This extension defines the catalog endpoint and response format without mandating what's in it.

## Manifest Contribution

**Config:**
```json
{
  "name": "resource-catalog",
  "version": "1.0.0",
  "config": {
    "endpoint": "/catalog",
    "format": "json",
    "searchable": false
  }
}
```

- `endpoint`: HTTP path where the catalog is served. Default: `"/catalog"`.
- `format`: Response format. `"json"` (structured) or `"text"` (plain list). Default: `"json"`.
- `searchable`: Whether the endpoint accepts query parameters for filtering. Default: `false`.

**Network entries:**
```json
{
  "network": [
    { "url": "https://nas.local:9000/catalog", "roles": ["peer"], "protocol": "HTTP", "description": "File catalog" }
  ]
}
```

## Negotiation

Configuration-only. A node either serves a catalog or it doesn't. No runtime negotiation — the catalog endpoint is public (or behind auth if `jwt-auth` is active).

## Communication

### Listing

```
GET /catalog
GET /catalog?type=video
GET /catalog?q=holiday
GET /catalog?offset=0&limit=50
```

Response (`format: "json"`):
```json
{
  "total": 142,
  "offset": 0,
  "limit": 50,
  "resources": [
    {
      "resource_id": "a1b2c3d4e5f6a7b8",
      "metadata_url": "/metadata/a1b2c3d4e5f6a7b8"
    },
    {
      "resource_id": "f8e7d6c5b4a39281",
      "metadata_url": "/metadata/f8e7d6c5b4a39281"
    }
  ]
}
```

Response (`format: "text"`):
```
a1b2c3d4e5f6a7b8
f8e7d6c5b4a39281
c3d4e5f6a7b8a1b2
```

### Query Parameters (when `searchable: true`)

| Parameter | Type | Description |
|---|---|---|
| `q` | string | Free-text search. What it searches is implementation-defined. |
| `type` | string | Filter by resource type (if `resource-metadata` provides types). |
| `offset` | integer | Pagination offset. Default: 0. |
| `limit` | integer | Max results. Default: 50. Max: 500. |

When `searchable: false`, the endpoint ignores query parameters and returns the full list (paginated).

### Error Responses

| Status | Meaning |
|---|---|
| 200 | Success |
| 401 | Auth required (if `jwt-auth` active) |
| 503 | Catalog not ready (node still scanning) |

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `endpoint` | any HTTP path | `"/catalog"` | NO | NO |
| `format` | `json`, `text` | `json` | YES | NO |
| `searchable` | `true`, `false` | `false` | NO | NO |

## Dependencies

- None required.
- **`resource-metadata`** — optional. When available, catalog entries include `metadata_url`. Without it, catalog returns bare resource_ids.
- **`jwt-auth`** — optional. When active, catalog endpoint requires authentication.

## Example

### NAS runner listing video files

```
GET /catalog?type=video&limit=10
```
```json
{
  "total": 47,
  "offset": 0,
  "limit": 10,
  "resources": [
    {
      "resource_id": "a1b2c3d4e5f6a7b8",
      "metadata_url": "/metadata/a1b2c3d4e5f6a7b8"
    }
  ]
}
```

### Public hash listing (no metadata)

```
GET /catalog
```
```
a1b2c3d4e5f6a7b8
f8e7d6c5b4a39281
```

### Torrent-style tracker listing

```
GET /catalog?limit=100
```
```json
{
  "total": 12483,
  "offset": 0,
  "limit": 100,
  "resources": [
    {
      "resource_id": "a1b2c3d4e5f6a7b8",
      "metadata_url": "/metadata/a1b2c3d4e5f6a7b8"
    }
  ]
}
```
