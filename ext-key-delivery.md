# Extension: key-delivery

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer, tracker

## Purpose

Defines how encryption keys reach the node that needs them. `aes-encryption` defines what cipher is used but explicitly punts on how the key is delivered. This extension fills that gap. Different networks deliver keys differently — via API, via peer exchange, via URL, via user input. This extension defines the delivery methods and lets nodes declare which ones they support.

## Manifest Contribution

**Config:**
```json
{
  "name": "key-delivery",
  "version": "1.0.0",
  "config": {
    "methods": ["api", "url"],
    "api_endpoint": "/keys"
  }
}
```

- `methods`: Array of supported delivery methods. At least one required.
- `api_endpoint`: HTTP path for key retrieval (when method includes `"api"`). Default: `"/keys"`.

### Delivery Methods

| Method | Description |
|---|---|
| `api` | Key fetched from an HTTP endpoint. Node serves keys at `{api_endpoint}/{resource_id}`. Auth typically required. |
| `url` | Key URL embedded in metadata or HLS manifest (`#EXT-X-KEY:URI=...`). Fetcher resolves the URL. |
| `peer-exchange` | Key exchanged directly between peers via `chunk-exchange` extension message after auth. |
| `manual` | Key provided by user (password, file, USB). Not protocol-mediated. |

**Rule fields:**
```json
{
  "rules": {
    "key_delivery_required": true,
    "allowed_key_methods": ["api", "url"]
  }
}
```

- `key_delivery_required`: When `true`, peers MUST declare a key delivery method if they use `aes-encryption`. Default: `false`.
- `allowed_key_methods`: Restricts which delivery methods are permitted in the network.

## Negotiation

Peer-negotiated. When two peers both support `aes-encryption` and `key-delivery`, the seeder tells the leecher how to obtain the key.

A tracker MAY require it in `required_extensions` alongside `aes-encryption`.

## Communication

### Method: `api`

```
GET /keys/{resource_id}
Authorization: Bearer <JWT>
```

Response:
```json
{
  "resource_id": "a1b2c3d4e5f6a7b8",
  "key": "<base64-encoded-key>",
  "iv": "<base64-encoded-iv>",
  "cipher": "AES-128-CBC",
  "expires": "2025-06-15T11:30:00Z"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `resource_id` | string | YES | Resource the key unlocks. |
| `key` | string | YES | Base64-encoded encryption key. |
| `iv` | string | NO | Base64-encoded IV. If absent, derived from context (e.g. HLS segment number). |
| `cipher` | string | YES | Cipher algorithm (matches `aes-encryption` config). |
| `expires` | string | NO | ISO 8601. Key valid until this time. Absent = no expiry. |

Error responses:

| Status | Meaning |
|---|---|
| 200 | Success |
| 401 | Not authenticated |
| 403 | Not authorized for this resource |
| 404 | Unknown resource or no key available |

### Method: `url`

Key URL appears in HLS manifest or resource metadata:

```
#EXT-X-KEY:METHOD=AES-128,URI="https://api.kippit.net/keys/a1b2c3d4e5f6a7b8",IV=0x00000001
```

The URL is resolved by the player/client. Auth headers apply if `jwt-auth` is active.

### Method: `peer-exchange`

After `chunk-exchange` handshake + `jwt-auth` verification, the seeder sends the key via extension message (type 0x80+):

```json
{
  "type": "key-offer",
  "resource_id": "a1b2c3d4e5f6a7b8",
  "key": "<base64-encoded-key>",
  "cipher": "AES-128-CBC"
}
```

Leecher confirms:

```json
{
  "type": "key-ack",
  "resource_id": "a1b2c3d4e5f6a7b8"
}
```

**Security:** `peer-exchange` SHOULD only be used on authenticated connections (`jwt-auth` verified). Sending keys to unauthenticated peers defeats the purpose of encryption.

### Method: `manual`

No protocol interaction. The user provides the key through the application UI, a file, or environment variable. The node uses whatever key the user gave it.

Extension message after handshake (informational):

```json
{
  "type": "key-method",
  "method": "manual"
}
```

This tells the peer "I have the key, don't try to send it to me."

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `methods` | `api`, `url`, `peer-exchange`, `manual` | `["api"]` | YES | YES |
| `api_endpoint` | any HTTP path | `"/keys"` | NO | NO |

**Method negotiation:** When two peers support different methods, they use the first mutually supported method from the seeder's preference list. If no overlap and encryption is required → connection rejected.

## Dependencies

- **`aes-encryption`** — required. `key-delivery` without encryption has no purpose.
- **`jwt-auth`** — recommended for `api` and `peer-exchange` methods. Without auth, key delivery is insecure.
- **`chunk-exchange`** — required for `peer-exchange` method (uses extension messages).

## Example

### Kippit runner: API key delivery

Runner encrypts video with personal.key. Viewer gets the key from the API:

1. Viewer authenticates with Kippit API (JWT).
2. Viewer requests key: `GET /keys/a1b2c3d4e5f6a7b8` with JWT.
3. API verifies viewer has access (owner, share token, etc.).
4. API returns wrapped key.
5. Viewer's HLS.js uses key to decrypt segments.

Manifest fragment:
```json
{
  "extensions": [
    { "name": "aes-encryption", "version": "1.0.0", "config": { "cipher": "AES-128-CBC" } },
    { "name": "key-delivery", "version": "1.0.0", "config": { "methods": ["api", "url"], "api_endpoint": "/keys" } }
  ]
}
```

### LAN sharing: peer-exchange

Two NAS runners on the same LAN. Runner A has the key, Runner B needs it for replication:

1. B connects to A via mDNS.
2. B authenticates (jwt-auth).
3. A sends key via `peer-exchange`.
4. B stores key, decrypts chunks.

### Manual key: USB drive

User copies `personal.key` to a USB drive, plugs it into a second device. No protocol interaction — the device reads the file.
