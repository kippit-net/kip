# Extension: aes-encryption

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer

## Purpose

Per-chunk encryption. Chunks are encrypted before transfer, decrypted after receipt. The encryption key is not part of the protocol — key delivery is out-of-band (API, user input, key exchange extension). This extension only defines how encrypted chunks are flagged and what cipher is used.

## Manifest Contribution

**Config:**
```json
{
  "name": "aes-encryption",
  "version": "1.0.0",
  "config": {
    "cipher": "AES-128-CBC"
  }
}
```

- `cipher`: Encryption algorithm. `"AES-128-CBC"` (HLS standard) or `"AES-256-GCM"`. Default: `"AES-128-CBC"`.

**Rule fields:**
```json
{
  "rules": {
    "encryption_required": true
  }
}
```

- `encryption_required`: When `true`, all chunk data MUST be encrypted. Unencrypted Piece messages are rejected.

## Negotiation

Peer-negotiated. Both peers must declare `aes-encryption`. After `chunk-exchange` handshake, if both support it, they exchange encryption parameters via extension message.

A tracker MAY require it in `required_extensions`.

## Communication

After `chunk-exchange` handshake, peers exchange encryption params:

Extension message payload:
```json
{
  "cipher": "AES-128-CBC",
  "key_delivery": "out-of-band"
}
```

- `key_delivery`: How the decryption key is obtained. `"out-of-band"` means the key is not exchanged in-protocol (fetched from API, derived from user password, etc.). Future versions MAY define in-band key exchange.

Piece messages (0x04) carry encrypted chunk data. The receiving peer decrypts after receipt using the key obtained out-of-band.

### Integration with HLS

When combined with `hls-streaming`, encryption follows the HLS standard:
- AES-128-CBC with PKCS7 padding
- IV derived from segment sequence number or specified in the m3u8 manifest via `#EXT-X-KEY`
- Key URL in m3u8 points to the key delivery mechanism (API endpoint, in-manifest, etc.)

## Configuration

| Parameter | Alternatives | Default | Tracker-mandatable | Negotiable |
|---|---|---|---|---|
| `cipher` | `AES-128-CBC`, `AES-256-GCM`, `ChaCha20-Poly1305` | `AES-128-CBC` | YES | YES |
| `key_delivery` | `out-of-band`, `in-band` | `out-of-band` | YES | NO |

**Cipher negotiation:** When two peers both support `aes-encryption` but with different ciphers, they use the first mutually supported cipher from the seeder's preference list. If no overlap → extension inactive for this connection, Error if encryption is required.

**Tracker override:** If the tracker mandates `"cipher": "AES-128-CBC"` in rules, all peers must use that cipher. No negotiation.

## Dependencies

- `chunk-exchange` — encrypts/decrypts Piece message data.
- `key-delivery` — recommended. Defines how the decryption key reaches the recipient. Without it, key delivery is entirely implementation-defined.

## Example

Runner encrypts video with personal.key:

1. Runner processes video → HLS segments → encrypts each segment with AES-128-CBC using personal.key
2. Runner's Piece messages contain encrypted data
3. Viewer obtains personal.key via API (out-of-band)
4. Viewer decrypts each Piece after receipt
5. HLS.js plays decrypted segments
