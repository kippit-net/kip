# Extension: jwt-auth

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** tracker, peer

## Purpose

JWT-based authentication. Nodes prove their identity by presenting signed JWTs. The tracker MAY require auth to join the network. Peers MAY require auth before exchanging data.

## Manifest Contribution

**Config:**
```json
{
  "name": "jwt-auth",
  "version": "1.0.0",
  "config": {
    "method": "ES256",
    "jwks_url": "https://kippit.net/.well-known/jwks.json"
  }
}
```

- `method`: Signing algorithm. `"ES256"` (ECDSA P-256) for Kippit. Extensions or implementations MAY support others.
- `jwks_url`: URL to fetch public keys for JWT verification.

**Rule fields:**
```json
{
  "rules": {
    "required_extensions": ["jwt-auth"],
    "auth_required": true
  }
}
```

- `auth_required`: When `true`, the tracker rejects unauthenticated announces and peers reject unauthenticated chunk requests.

## Negotiation

**Tracker-mandated:** Tracker lists `jwt-auth` in `required_extensions`. Peers without it can't join.

**Peer-negotiated:** When two peers connect, if both declare `jwt-auth`, they exchange JWTs after the `chunk-exchange` handshake. If only one has it, auth is skipped (extension inactive for that connection).

## Communication

### Tracker Auth

Peer includes JWT in the `accept` message (from `kippit-tracker`):

```json
{
  "type": "accept",
  "version": 1,
  "peer_id": "a1b2c3d4e5f6...",
  "auth": {
    "token": "eyJhbGciOiJFUzI1NiIs..."
  }
}
```

Tracker verifies the JWT against the JWKS. Invalid or missing token when `auth_required` = error `AUTH_REQUIRED`.

### Peer-to-Peer Auth

After `chunk-exchange` handshake, if both peers have `jwt-auth` active:

1. Both peers send an extension message (type 0x80) with a JWT
2. Each peer verifies the other's JWT
3. If verification fails, the verifying peer sends Error and disconnects

Extension message payload:
```
┌──────────────┬──────────────────────┐
│ ext_id: auth │ jwt (UTF-8, var)     │
│ (2B)         │                      │
└──────────────┴──────────────────────┘
```

### JWT Claims

Kippit JWTs contain at minimum:

| Claim | Description |
|---|---|
| `sub` | Subject (user ID or runner ID) |
| `iss` | Issuer (e.g. `kippit.net`) |
| `exp` | Expiration (1 hour from issuance) |
| `iat` | Issued at |
| `type` | `"identity"` (user) or `"nas"` (runner) |

Extensions MAY define additional claims (e.g. share tokens, permissions).

## Dependencies

None. Can be used with or without any other extension. When combined with `kippit-tracker`, auth applies to tracker communication. When combined with `chunk-exchange`, auth applies to peer connections.

## Example

Runner joins kippit.net (auth required):

1. Runner has a NAS license JWT (type: `"nas"`, long-lived, revokable)
2. Runner fetches tracker manifest: `required_extensions: ["jwt-auth"]`
3. Runner connects WebSocket, receives manifest
4. Runner sends accept with `auth.token` = NAS JWT
5. Tracker verifies against JWKS → accept
6. If invalid → error `AUTH_REQUIRED`, connection closed
