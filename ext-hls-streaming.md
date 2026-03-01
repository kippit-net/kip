# Extension: hls-streaming

**Version:** 1.0.0
**Layer:** Semantics (4)
**Roles:** peer

## Purpose

Video streaming via HLS. Generates m3u8 manifests from chunked resources, overrides chunk selection to sequential (for playback buffering), and exposes HTTP endpoints for manifest delivery. Transforms chunk-exchange from "download a file" into "stream a video."

## Manifest Contribution

**Config:**
```json
{
  "name": "hls-streaming",
  "version": "1.0.0"
}
```

**Network entries:** A peer serving video lists its HLS endpoint:
```json
{
  "url": "http://192.168.1.50:9000/hls",
  "roles": [],
  "description": "HLS manifest endpoint"
}
```

## Negotiation

Peer-negotiated. Both peers must declare `hls-streaming`. When active, chunk selection switches from random to sequential.

If only the seeder has it, the extension is inactive — the leecher gets chunks in whatever order it requests (still playable, just not optimized for streaming).

## Communication

### HLS Manifest Endpoint

A peer with `hls-streaming` serves m3u8 manifests over HTTP:

```
GET /hls/{resource_id}/manifest.m3u8
→ 200 OK
→ Content-Type: application/vnd.apple.mpegurl
```

The manifest maps resource chunks to HLS segments:

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-KEY:METHOD=AES-128,URI="key://personal.key"
#EXTINF:10.0,
chunk://resource_id/0
#EXTINF:10.0,
chunk://resource_id/1
...
```

The `chunk://` scheme is a convention — the player resolves these via `chunk-exchange` (P2P) or direct HTTP (LAN). The key URI scheme depends on the key delivery mechanism.

### Chunk Selection Override

When `hls-streaming` is active on a connection, the leecher requests chunks sequentially starting from the playback position. The seeder prioritizes sequential responses.

Extension message after handshake:
```json
{
  "playback_position": 0,
  "buffer_ahead_chunks": 15
}
```

- `playback_position`: Starting chunk index.
- `buffer_ahead_chunks`: How many chunks ahead to buffer. Default: 15 (~30 seconds at 2MB chunks / 10s segments).

The leecher MAY update playback position during the session (seek).

### Integration with p2p-media-loader

In a browser context, p2p-media-loader intercepts HLS.js segment requests and routes them through `chunk-exchange` via WebRTC. The m3u8 manifest is fetched from the peer's HTTP endpoint, segments are fetched P2P.

## Dependencies

- `chunk-exchange` — provides the underlying chunk transfer mechanism.
- `aes-encryption` (optional) — when active, chunks are encrypted HLS segments.

## Example

Watching a movie from a NAS:

1. Runner processes video → HLS segments → maps to chunks
2. Viewer fetches `http://nas.local:9000/hls/abc123/manifest.m3u8`
3. m3u8 lists chunk references
4. p2p-media-loader resolves chunks via WebRTC data channel (chunk-exchange)
5. HLS.js decrypts (if aes-encryption active) and plays
6. Sequential chunk priority — no buffering interruptions
