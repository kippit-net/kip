# KIP-0003: Extension Negotiation

**Status:** Draft
**Version:** 0.2
**Component:** spec

## Abstract

KIP-0003 defines extension negotiation at **two levels**:

1. **Layer 3 (PEER ↔ PEER):** Negotiation in the wire protocol handshake. A peer declares a list of extensions, the other peer responds. Communication is extended only for extensions accepted by both.

2. **Layer 1 (PEER ↔ REGISTRY):** Capability advertisement in the discovery protocol handshake. The REGISTRY announces requirements and offerings. The PEER decides whether to participate.

These two mechanisms are **independent** — different connection, different handshake, different capabilities.

## 1. Handshake Extension Field

In the Handshake message (KIP-0001, type 0x00) the `extensions` field contains a list of extensions:

```
extensions field:
┌──────────────────┬─────────────────────────────────┐
│ ext_count (2B)   │ extension entries (variable)    │
└──────────────────┴─────────────────────────────────┘

extension entry:
┌──────────────┬───────────────┬───────────────────┐
│ ext_id (2B)  │ version (2B)  │ data_len (2B)     │
│              │               │ + data (variable) │
└──────────────┴───────────────┴───────────────────┘
```

- **ext_count:** uint16, number of extensions in the list.
- **ext_id:** uint16, KIP extension number (e.g. 4 = KIP-0004 Auth, 5 = KIP-0005 Encryption).
- **version:** uint16, extension version supported by the peer.
- **data_len:** uint16, size of optional extension configuration data.
- **data:** optional bytes specific to each extension (e.g. list of supported cipher suites in KIP-0005).

## 2. Negotiation

### 2.1 Algorithm

1. Peer A sends Handshake with `extensions: [{ext_id: 4, v: 1}, {ext_id: 5, v: 1}, {ext_id: 6, v: 1}]`
2. Peer B sends Handshake with `extensions: [{ext_id: 4, v: 1}, {ext_id: 6, v: 2}]`
3. Both sides compute the **intersection**:
   - KIP-0004 v1: both support → **active**
   - KIP-0005 v1: only A → **inactive** (B doesn't support it)
   - KIP-0006: A has v1, B has v2 → **active with the lower version (v1)**, provided the extension defines backward compatibility. If not — inactive.

### 2.2 Rules

1. An extension is active for a connection **only if both peers declared it**.
2. Versioning: if versions differ, both sides use **min(version_A, version_B)**. An extension MAY define that certain versions are not backward-compatible (then treated as incompatible).
3. Order in the list does not matter.
4. Unknown ext_id = ignored (graceful degradation). A peer MUST NOT disconnect because of an unknown extension.
5. No extensions (`ext_count: 0`) is valid = core-only connection.

## 3. Extension Messages

Extensions communicate via message types **0x80-0xFF** (128-255) in KIP-0001.

Mapping ext_id → message types:

```
Extension message:
┌──────────────┬───────────┬──────────────────┐
│ type (1B)    │ length    │ payload           │
│ 0x80-0xFF    │ (4B)      │ (extension-spec)  │
└──────────────┴───────────┴──────────────────┘
```

- **type 0x80:** Generic extension message. Payload starts with `ext_id (2B)` — router to the appropriate extension.
- **type 0x81-0xFF:** May be reserved by specific extensions (if an extension needs a dedicated message type for performance).

### 3.1 Generic Extension Message (0x80)

```
┌──────────┬──────────┬───────────┬──────────────────┐
│ type=0x80│ length   │ ext_id    │ ext_payload       │
│ (1B)     │ (4B)     │ (2B)      │ (variable)        │
└──────────┴──────────┴───────────┴──────────────────┘
```

A peer MUST ignore extension messages for an ext_id that is not active for this connection.

## 4. Reserved Extension IDs

| ext_id | KIP | Name | Description |
|---|---|---|---|
| 0x0001-0x0003 | - | Reserved | Core protocol (not extensions) |
| 0x0004 | KIP-0004 | Auth | JWT authentication |
| 0x0005 | KIP-0005 | Encryption | Per-chunk encryption (AES-128-CBC, AES-256-GCM) |
| 0x0006 | KIP-0006 | Streaming | Sequential priority, HLS manifest hints |
| 0x0007 | KIP-0007 | Sync | Operation log, vector clocks |
| 0x0008 | KIP-0008 | Messaging | Real-time small data |
| 0x0009 | KIP-0009 | BT Bridge | BitTorrent wire protocol compatibility |
| 0x000A-0x0063 | - | Reserved | Future official extensions |
| 0x0064-0xFFFF | - | Community | Community-defined extensions |

## 5. Extension Lifecycle

```
         Handshake
             │
             ▼
     ┌───────────────┐
     │ Negotiate      │ ← intersection of both peers' lists
     │ active exts    │
     └───────┬───────┘
             │
             ▼
     ┌───────────────┐
     │ Extension init │ ← each active extension runs OnActivate()
     │ (per ext)      │    may exchange initial extension messages
     └───────┬───────┘
             │
             ▼
     ┌───────────────┐
     │ Normal         │ ← extension messages flow alongside core messages
     │ operation      │
     └───────┬───────┘
             │
             ▼
     ┌───────────────┐
     │ Disconnect     │ ← each active extension runs OnDeactivate()
     └───────────────┘
```

An extension MAY exchange additional messages after negotiation but before normal operation (e.g. KIP-0005 must exchange key parameters before chunks are sent encrypted).

## 6. Example: Full Session with Extensions

```
Peer A (runner, seeding a movie):
  extensions: [KIP-0004 v1, KIP-0005 v1, KIP-0006 v1]

Peer B (web app, wants to watch):
  extensions: [KIP-0004 v1, KIP-0005 v1, KIP-0006 v1]

Negotiated: KIP-0004, KIP-0005, KIP-0006 — all active.

Flow:
1. Handshake (both sides, KIP-0001)
2. KIP-0004: A sends JWT challenge, B responds with signed JWT → auth OK
3. KIP-0005: A sends encryption params {method: AES-128-CBC, key_delivery: "ext-x-key"}
4. KIP-0006: B sends streaming hints {playback_position: 0, buffer_ahead: 30s}
5. Bitfield (KIP-0001) — A has all chunks
6. Interested (KIP-0001) — B interested in resource
7. Request/Piece (KIP-0001) — sequential order (KIP-0006 overrides default)
   Pieces are encrypted (KIP-0005) — B decrypts after receiving
```

```
Peer C (CLI tool, phase 0):
  extensions: []

Peer D (CLI tool, phase 0):
  extensions: []

Negotiated: nothing. Core-only.

Flow:
1. Handshake (both sides)
2. Bitfield
3. Request/Piece — raw bytes, no encryption, no auth
```

## 7. Discovery Manifest (Layer 1)

A separate mechanism from wire protocol extension negotiation (sections 1-6). Described in KIP-0002 section 4.2.

### 7.1 Differences vs Wire Protocol Negotiation

| Aspect | Wire (PEER ↔ PEER) | Discovery (PEER ↔ REGISTRY) |
|---|---|---|
| Handshake | Binary (KIP-0001 type 0x00) | JSON `hello` message (KIP-0002) |
| Negotiation | Intersection (both must support it) | Manifest (REGISTRY imposes rules, PEER accepts or leaves) |
| Format | ext_id + version + data | capabilities object |
| Symmetry | Symmetric (both peers are equal) | Asymmetric (REGISTRY has requirements, PEER meets them or doesn't) |
| Example | KIP-0005 encryption: both peers negotiate cipher | Auth required: REGISTRY requires JWT, PEER provides or leaves |

### 7.2 Shared Extension IDs

Extension IDs (section 4) are **shared** across layers. KIP-0004 (Auth) has the same ext_id (0x0004) regardless of whether it's negotiated peer-to-peer or required by REGISTRY. But extension behavior may differ per layer:

- **KIP-0004 on wire (PEER ↔ PEER):** JWT challenge-response after handshake
- **KIP-0004 on discovery (PEER ↔ REGISTRY):** JWT attached to announce message

An extension spec (e.g. KIP-0004) MUST define behavior on both layers if it applies to both.

## 8. Out of Scope for KIP-0003

- Details of individual extensions (KIP-0004, KIP-0005, ...) — each has its own spec.
- Discovery of extensions (which extensions exist) — docs/website, not protocol.
- Extension dependencies (e.g. KIP-0006 requires KIP-0005) — defined in the extension's spec, not in core negotiation.

## Open Questions

1. Should wire extension negotiation be a separate message type after Handshake, or part of the Handshake? Current: part of Handshake (simpler).
2. Should an extension be able to reject a connection? (E.g. KIP-0004 requires auth, peer without auth = disconnect.) Current: the extension decides in OnActivate(), may send Error.
3. Is message type 0x80 (generic) sufficient, or should extensions have dedicated message types?
4. Should the discovery manifest use the same binary format as wire extensions, or is JSON fine? Current decision: JSON (because the discovery protocol is JSON-based).
