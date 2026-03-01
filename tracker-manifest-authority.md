# Idea: Tracker Manifest — Tracker as Federation Authority

**Status:** Active — influences KIP-0002

## Idea

The tracker (REGISTRY) is not a dumb relay. It is an **authority that defines the rules of communication** in its network. Federation, not democracy.

A peer connects to a tracker and receives a **manifest** — a set of rules: which extensions are required, how to announce resources, how to connect to other peers, push or pull. The peer either accepts the rules or leaves.

## Consequences

### The Tracker Decides, Not the Peer

- Tracker says: "auth is required in my network" → peer provides a JWT or gets rejected
- Tracker says: "announce yourself as a host, don't list files" → peer says "I'm X, I host server A" instead of sending 50,000 resource_ids
- Tracker says: "new peers get push notifications about changes" → peer listens
- Tracker says: "connections via WebRTC, signaling through me" → peer uses this tracker's signaling

### Different Trackers, Different Rules

| Tracker | Rules |
|---|---|
| Ours (kippit.net) | Auth required, encryption required, push notifications, announce per-server |
| Public (community) | No auth, no encryption, announce per-file, pull only |
| Corporate (self-hosted) | Auth required, encryption required, no external peers, LAN only |
| BT bridge tracker | BT-compatible announce format, no KIP extensions |

The RFC defines the **manifest mechanism** (how the tracker communicates rules), not the **specific rules** (that's the operator's decision).

### Not a Democracy

In a blockchain, peers negotiate consensus. Not here. The tracker is a federator — like a Matrix server, like a Mastodon instance. The tracker operator sets the policy. The peer chooses which tracker to work with.

The RFC MAY define democratic mechanisms (e.g. peer voting on rules) as an optional extension. But the core model is federation.

### Announce: Hosts, Not Files

With 50,000 files, a peer should not announce the file list. Instead:

```
"I'm peer X, I host files from server A, B, C"
```

The file list is a detail available through the API (phase 1+) or directly from the peer (wire protocol). The tracker knows WHO hosts, not WHAT they host.

However: the tracker MAY require per-file announce (e.g. a public tracker for sharing). It depends on the manifest.

### Impact on KIP-0002

Section 4.2 (hello handshake) should be rebuilt as **manifest delivery**:

```json
{
  "type": "manifest",
  "version": 1,
  "rules": {
    "auth_required": true,
    "auth_extension": 4,
    "announce_mode": "per-server",
    "signaling": "provided",
    "relay": "available",
    "push_peers": true,
    "required_extensions": [4, 5],
    "encryption_required": true
  }
}
```

The peer responds with acceptance or disconnects.

## Related

- [protocol-layer-model.md](protocol-layer-model.md) — layer model (tracker = REGISTRY role)
- [KIP-0002](KIP-0002-peer-discovery.md) — discovery & signaling protocol (manifest in section 4.2)
