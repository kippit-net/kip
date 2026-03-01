# KIP-0003: Extension Negotiation

**Status:** Draft
**Version:** 0.2
**Component:** spec

## Abstract

KIP-0003 definiuje extension negotiation na **dwóch poziomach**:

1. **Warstwa 3 (PEER ↔ PEER):** Negocjacja w wire protocol handshake. Peer deklaruje listę extensionów, drugi peer odpowiada. Komunikacja rozszerzona tylko dla extensionów zaakceptowanych przez obu.

2. **Warstwa 1 (PEER ↔ REGISTRY):** Capability advertisement w discovery protocol handshake. REGISTRY ogłasza wymagania i offerings. PEER decyduje czy chce gadać.

Te dwa mechanizmy są **niezależne** — inne połączenie, inny handshake, inne capabilities.

## 1. Handshake extension field

W Handshake message (KIP-0001, type 0x00) pole `extensions` zawiera listę extensionów:

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

- **ext_count:** uint16, liczba extensionów w liście.
- **ext_id:** uint16, numer KIP extensionu (np. 4 = KIP-0004 Auth, 5 = KIP-0005 Encryption).
- **version:** uint16, wersja extensionu obsługiwana przez peera.
- **data_len:** uint16, rozmiar opcjonalnych danych konfiguracyjnych extensionu.
- **data:** opcjonalne bajty specyficzne per extension (np. lista obsługiwanych cipher suites w KIP-0005).

## 2. Negocjacja

### 2.1 Algorytm

1. Peer A wysyła Handshake z `extensions: [{ext_id: 4, v: 1}, {ext_id: 5, v: 1}, {ext_id: 6, v: 1}]`
2. Peer B wysyła Handshake z `extensions: [{ext_id: 4, v: 1}, {ext_id: 6, v: 2}]`
3. Obie strony obliczają **intersection**:
   - KIP-0004 v1: oba obsługują → **aktywny**
   - KIP-0005 v1: tylko A → **nieaktywny** (B nie umie)
   - KIP-0006: A ma v1, B ma v2 → **aktywny z niższą wersją (v1)**, o ile extension definiuje backward compat. Jeśli nie — nieaktywny.

### 2.2 Reguły

1. Extension jest aktywny dla połączenia **tylko jeśli oba peery go zadeklarowały**.
2. Wersjonowanie: jeśli wersje się różnią, obie strony używają **min(version_A, version_B)**. Extension MOŻE definiować, że pewne wersje nie są backward-compatible (wtedy traktowane jako niezgodne).
3. Kolejność w liście nie ma znaczenia.
4. Nieznany ext_id = ignorowany (graceful degradation). Peer NIE rozłącza się z powodu nieznanego extensionu.
5. Brak extensionów (`ext_count: 0`) jest valid = core-only connection.

## 3. Extension messages

Extensiony komunikują się przez message types **0x80-0xFF** (128-255) w KIP-0001.

Mapping ext_id → message types:

```
Extension message:
┌──────────────┬───────────┬──────────────────┐
│ type (1B)    │ length    │ payload           │
│ 0x80-0xFF    │ (4B)      │ (extension-spec)  │
└──────────────┴───────────┴──────────────────┘
```

- **type 0x80:** Generic extension message. Payload zaczyna się od `ext_id (2B)` — router do właściwego extensionu.
- **type 0x81-0xFF:** Mogą być zarezerwowane przez konkretne extensiony (jeśli extension potrzebuje dedykowanego message type dla performance).

### 3.1 Generic extension message (0x80)

```
┌──────────┬──────────┬───────────┬──────────────────┐
│ type=0x80│ length   │ ext_id    │ ext_payload       │
│ (1B)     │ (4B)     │ (2B)      │ (variable)        │
└──────────┴──────────┴───────────┴──────────────────┘
```

Peer MUSI ignorować extension messages dla ext_id który nie jest aktywny dla tego połączenia.

## 4. Reserved extension IDs

| ext_id | KIP | Nazwa | Opis |
|---|---|---|---|
| 0x0001-0x0003 | - | Reserved | Core protocol (nie extensiony) |
| 0x0004 | KIP-0004 | Auth | JWT authentication |
| 0x0005 | KIP-0005 | Encryption | Per-chunk encryption (AES-128-CBC, AES-256-GCM) |
| 0x0006 | KIP-0006 | Streaming | Sequential priority, HLS manifest hints |
| 0x0007 | KIP-0007 | Sync | Operation log, vector clocks |
| 0x0008 | KIP-0008 | Messaging | Real-time small data |
| 0x0009 | KIP-0009 | BT Bridge | BitTorrent wire protocol compatibility |
| 0x000A-0x0063 | - | Reserved | Przyszłe oficjalne extensiony |
| 0x0064-0xFFFF | - | Community | Community-defined extensiony |

## 5. Extension lifecycle

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

Extension MOŻE wymienić dodatkowe wiadomości po negocjacji ale przed normalną operacją (np. KIP-0005 musi wymienić key parameters zanim chunki polecą zaszyfrowane).

## 6. Przykład: pełna sesja z extensionami

```
Peer A (runner, seeduje film):
  extensions: [KIP-0004 v1, KIP-0005 v1, KIP-0006 v1]

Peer B (web app, chce oglądać):
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
Peer C (CLI tool, faza 0):
  extensions: []

Peer D (CLI tool, faza 0):
  extensions: []

Negotiated: nothing. Core-only.

Flow:
1. Handshake (both sides)
2. Bitfield
3. Request/Piece — raw bytes, no encryption, no auth
```

## 7. Discovery manifest (warstwa 1)

Osobny mechanizm od wire protocol extension negotiation (sekcje 1-6). Opisany w KIP-0002 sekcja 4.2.

### 7.1 Różnice vs wire protocol negotiation

| Aspekt | Wire (PEER ↔ PEER) | Discovery (PEER ↔ REGISTRY) |
|---|---|---|
| Handshake | Binarny (KIP-0001 type 0x00) | JSON `hello` message (KIP-0002) |
| Negocjacja | Intersection (oba muszą umieć) | Manifest (REGISTRY narzuca reguły, PEER akceptuje lub odchodzi) |
| Format | ext_id + version + data | capabilities object |
| Symetria | Symetryczny (oba peery równe) | Asymetryczny (REGISTRY ma wymagania, PEER je spełnia lub nie) |
| Przykład | KIP-0005 encryption: oba peery negocjują cipher | Auth required: REGISTRY wymaga JWT, PEER dostarcza lub odchodzi |

### 7.2 Shared extension IDs

Extension IDs (sekcja 4) są **współdzielone** między warstwami. KIP-0004 (Auth) ma ten sam ext_id (0x0004) niezależnie czy jest negocjowany peer-to-peer czy wymagany przez REGISTRY. Ale zachowanie extensionu może się różnić per warstwa:

- **KIP-0004 na wire (PEER ↔ PEER):** JWT challenge-response po handshake
- **KIP-0004 na discovery (PEER ↔ REGISTRY):** JWT dołączany do announce message

Extension spec (np. KIP-0004) MUSI definiować zachowanie na obu warstwach jeśli dotyczy obu.

## 8. Poza scope KIP-0003

- Szczegóły poszczególnych extensionów (KIP-0004, KIP-0005, ...) — każdy ma własny spec.
- Discovery extensionów (jakie extensiony istnieją) — docs/website, nie protocol.
- Extension dependencies (np. KIP-0006 wymaga KIP-0005) — definiowane w spec extensionu, nie w core negotiation.

## Pytania otwarte

1. Czy wire extension negotiation powinna być osobnym message type po Handshake, czy częścią Handshake? Obecnie: część Handshake (prostsze).
2. Czy extension powinna móc odrzucić połączenie? (Np. KIP-0004 wymaga auth, peer bez auth = disconnect.) Obecnie: extension decyduje w OnActivate(), może wysłać Error.
3. Czy message type 0x80 (generic) jest wystarczający, czy extensiony powinny mieć dedykowane message types?
4. Czy discovery manifest powinien mieć ten sam binarny format co wire extensions, czy JSON jest OK? Obecna decyzja: JSON (bo discovery protocol jest JSON-based).
