# KIP-0001: Core Wire Protocol

**Status:** Draft
**Version:** 0.1
**Component:** spec

## Abstract

KIP-0001 definiuje wire protocol dla wymiany chunków między peerami. Transport-agnostic — działa na WebRTC data channel, TCP, in-memory pipe, czymkolwiek co ma `Send([]byte)` i `Recv() []byte`.

Protokół jest binarny, message-oriented (nie stream-oriented). Nie zawiera auth, encryption, ani semantyki aplikacyjnej (streaming, sync). Te capabilities dostarczają extensiony (KIP-0004+), negocjowane przez KIP-0003.

## 1. Terminologia

| Termin | Definicja |
|---|---|
| **Peer** | Uczestnik protokołu. Może być seeder, leecher, lub oba. |
| **Resource** | Logiczny zbiór chunków identyfikowany przez `resource_id`. Odpowiada jednemu plikowi. |
| **Chunk** | Jednostka transferu. Bajty o stałym rozmiarze (poza ostatnim). |
| **Swarm** | Grupa peerów wymieniających chunki tego samego resource'a. |
| **Bitfield** | Maska bitowa chunków które peer posiada. |

## 2. Identyfikatory

### 2.1 resource_id

**Format:** 16 bajtów (128-bit), reprezentowany jako 32-znakowy hex string w JSON/tekstowych kontekstach.

**Generowanie:** `SHA256(seed_input)[:16]`

Seed input zależy od kontekstu:
- **Standalone peer (faza 0):** `SHA256(absolute_path)[:16]` — deterministyczny, zmienia się przy rename/move
- **Registered runner (faza 1+):** `SHA256(runner_id + ":" + relative_path)[:16]` — deterministyczny w ramach runnera

**Decyzja: path-based, nie content-based.**

Uzasadnienie:
- Content-addressing (`SHA256(file_data)`) wymaga przeczytania całego pliku na starcie — niedopuszczalne dla 50GB filmów na wolnym NASie
- Rename zmienia resource_id — akceptowalne. Runner re-skanuje, tracker dostaje nowy announce, stare ID wygasa. Viewer pobiera od nowa (chunki i tak mogą być w cache)
- Dwa runnery seedujące ten sam plik = dwa różne resource_id — OK. Dedup na poziomie resource nie jest celem fazy 0-1 (dedup chunków to ewentualnie faza 5+)

**Stabilność:** resource_id jest stabilny dopóki plik nie zostanie przeniesiony, zmieniony ani usunięty. Runner_id jest persistent (generowany raz, zapisany w config).

**Co leaks:** resource_id sam w sobie nie ujawnia nazwy pliku, rozmiaru, typu ani zawartości. Tracker widzi tylko opaque ID + liczbę chunków. To spełnia US-0005.

### 2.2 peer_id

**Format:** 16 bajtów (128-bit), hex string.

**Generowanie:** `random(16)` — losowy, generowany przy starcie sesji.

**Lifecycle:** ephemeral. Nowy peer_id przy każdym połączeniu z trackerem. Peer nie jest trackable między sesjami.

**Rationale:** persistent peer_id umożliwia tracking (kto, kiedy, jak często). Ephemeral = privacy-by-default. Reconnect/resume po disconneccie nie wymaga persistent ID — tracker matchuje po WebSocket connection, nie po peer_id.

### 2.3 chunk_index

**Format:** uint32 (0-based).

**Adresowanie:** chunk jest identyfikowany przez `(resource_id, chunk_index)`. Nie jest content-addressed (brak hash per chunk w fazie 0).

**Chunk size:** konfigurowalny per resource. Default: 2 MB. Ostatni chunk może być mniejszy.

**Uwaga do przyszłości:** content-addressed chunks (hash per chunk) to potencjalny KIP extension dla integrity verification i dedup. Nie w core — core ufa transportowi.

## 3. Framing

Każda wiadomość ma stały header:

```
┌──────────┬───────────┬─────────────────────┐
│ type (1B)│ length (4B)│ payload (variable)  │
└──────────┴───────────┴─────────────────────┘
```

- **type:** uint8, identyfikuje typ wiadomości (0x00-0xFF)
- **length:** uint32 big-endian, rozmiar payload w bajtach (bez headera)
- **payload:** bajty specyficzne dla danego typu

**Max message size:** 4 MB (chunk data 2MB + overhead). Wiadomość przekraczająca limit = disconnect.

**Byte order:** big-endian (network byte order) dla wszystkich wielobajtowych pól.

## 4. Handshake

Handshake inicjuje połączenie. Oba peery wysyłają handshake jednocześnie (nie request-response).

```
Handshake message (type 0x00):
┌──────────┬───────────┬──────────┬───────────┬──────────────┬────────────┐
│ type=0x00│ length    │ version  │ peer_id   │ extensions   │ resources  │
│ (1B)     │ (4B)      │ (2B)     │ (16B)     │ (variable)   │ (variable) │
└──────────┴───────────┴──────────┴───────────┴──────────────┴────────────┘
```

- **version:** uint16, protocol version. Obecna: `0x0001`.
- **peer_id:** 16 bajtów, ephemeral ID tego peera.
- **extensions:** lista extensionów — format zdefiniowany w KIP-0003.
- **resources:** lista resource_id które peer seeduje (może być pusta dla leechera).

**Wersja niezgodna:** peer z nieobsługiwaną wersją wysyła `Error` (0x09) z kodem `VERSION_MISMATCH` i rozłącza się.

## 5. Typy wiadomości

### 5.1 Tabela typów

| Type | Hex | Nazwa | Kierunek | Opis |
|---|---|---|---|---|
| 0 | 0x00 | Handshake | ↔ | Inicjalizacja połączenia |
| 1 | 0x01 | Bitfield | ↔ | Maska posiadanych chunków |
| 2 | 0x02 | Have | → | "Mam nowy chunk" |
| 3 | 0x03 | Request | → | "Daj mi chunk X" |
| 4 | 0x04 | Piece | → | "Oto chunk X" |
| 5 | 0x05 | Cancel | → | "Nie potrzebuję chunka X" |
| 6 | 0x06 | KeepAlive | ↔ | Podtrzymanie połączenia |
| 7 | 0x07 | ResourceUpdate | → | "Mam nowy/usunięty resource" |
| 8 | 0x08 | Interested | ↔ | "Mam interes w twoich zasobach" |
| 9 | 0x09 | Error | ↔ | Błąd + disconnect |
| 128+ | 0x80+ | Extension | ↔ | Zarezerwowane dla KIP extensions |

### 5.2 Bitfield (0x01)

Wysyłany po handshake. Jeden bitfield per resource.

```
┌──────────────────┬───────────────┬───────────────┐
│ resource_id (16B)│ total_chunks  │ bitfield       │
│                  │ (4B, uint32)  │ (ceil(N/8) B)  │
└──────────────────┴───────────────┴───────────────┘
```

Bit 0 = chunk 0, bit 1 = chunk 1, itd. Bit ustawiony (1) = chunk posiadany.

Peer może wysłać wiele Bitfield wiadomości (jeden per resource).

### 5.3 Have (0x02)

Informuje o nowym chuneku (incremental update po Bitfield).

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

### 5.4 Request (0x03)

Żądanie chunka.

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

Peer POWINIEN odpowiedzieć wiadomością Piece lub Error. Brak odpowiedzi w ciągu 30 sekund = timeout, requestujący może ponowić lub wybrać innego peera.

**Request queue:** peer może mieć max 16 oczekujących requestów od jednego peera. Nowe requesty ponad limit = Error z kodem `TOO_MANY_REQUESTS`.

### 5.5 Piece (0x04)

Odpowiedź na Request.

```
┌──────────────────┬───────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │ data           │
│                  │ (4B, uint32)  │ (variable)     │
└──────────────────┴───────────────┴───────────────┘
```

**Data length:** wynika z `length` w framing header minus 20 bajtów (resource_id + chunk_index).

### 5.6 Cancel (0x05)

Anuluje poprzedni Request (peer znalazł chunk u innego peera).

```
┌──────────────────┬───────────────┐
│ resource_id (16B)│ chunk_index   │
│                  │ (4B, uint32)  │
└──────────────────┴───────────────┘
```

Peer POWINIEN przestać wysyłać Piece dla anulowanego chunka. Jeśli Piece jest już w trakcie wysyłania, Cancel może zostać zignorowany.

### 5.7 KeepAlive (0x06)

Pusty payload. Wysyłany co 30 sekund jeśli nie ma innego ruchu. Brak KeepAlive przez 90 sekund = peer uznany za martwego → disconnect.

### 5.8 ResourceUpdate (0x07)

Dynamic resource management — peer informuje o nowym lub usuniętym resource.

```
┌───────┬──────────────────┬───────────────┐
│ action│ resource_id (16B)│ total_chunks  │
│ (1B)  │                  │ (4B, uint32)  │
└───────┴──────────────────┴───────────────┘
```

- **action:** `0x01` = ADD (nowy resource), `0x02` = REMOVE (przestaję seedować)
- **total_chunks:** przy ADD — ile chunków ma resource. Przy REMOVE — ignorowane (0).

Po ADD, peer wysyła Bitfield dla nowego resource'a.

### 5.9 Interested (0x08)

Deklaracja zainteresowania resource'ami peera. Peer nie wysyła danych dopóki drugi peer nie wyśle Interested.

```
┌──────────────────┐
│ resource_id (16B)│
└──────────────────┘
```

Peer może wysłać wiele Interested (jeden per resource).

### 5.10 Error (0x09)

```
┌───────────┬──────────────────────┐
│ code (2B) │ message (UTF-8, var) │
└───────────┴──────────────────────┘
```

**Kody błędów:**

| Code | Hex | Nazwa | Opis |
|---|---|---|---|
| 1 | 0x0001 | VERSION_MISMATCH | Nieobsługiwana wersja protokołu |
| 2 | 0x0002 | UNKNOWN_RESOURCE | Requested resource nie istnieje u peera |
| 3 | 0x0003 | CHUNK_NOT_AVAILABLE | Chunk istnieje ale jeszcze nie gotowy |
| 4 | 0x0004 | TOO_MANY_REQUESTS | Przekroczony limit concurrent requests |
| 5 | 0x0005 | INVALID_MESSAGE | Malformed message |
| 6 | 0x0006 | INTERNAL_ERROR | Wewnętrzny błąd peera |
| 100+ | 0x0064+ | Extension errors | Zarezerwowane dla KIP extensions |

**Po Error:** połączenie jest zamykane. Error to terminal message.

## 6. Przepływ połączenia

```
Peer A (seeder)                              Peer B (leecher)
     │                                            │
     ├── Handshake {v1, peer_id_A, [res1]} ──────►│
     │◄── Handshake {v1, peer_id_B, []} ──────────┤
     │                                            │
     │   (extension negotiation — KIP-0003)       │
     │                                            │
     ├── Bitfield {res1, 100 chunks, 0xFF...} ───►│
     │                                            │
     │◄── Interested {res1} ──────────────────────┤
     │                                            │
     │◄── Request {res1, chunk 0} ────────────────┤
     ├── Piece {res1, chunk 0, <data>} ──────────►│
     │                                            │
     │◄── Request {res1, chunk 1} ────────────────┤
     │◄── Request {res1, chunk 2} ────────────────┤  (pipelining)
     ├── Piece {res1, chunk 1, <data>} ──────────►│
     ├── Piece {res1, chunk 2, <data>} ──────────►│
     │                                            │
     │   ... (repeat until all chunks) ...        │
     │                                            │
     ├── Have {res1, chunk 100} (new chunk) ─────►│  (incremental)
     │                                            │
```

## 7. Chunk size

**Default:** 2 MB (2,097,152 bytes).

**Rationale:**
- Za małe (64KB jak BT) = za dużo overhead na message framing + bitfield explosion dla dużych plików
- Za duże (16MB) = zbyt duża granularność, wolny start playbacku
- 2MB = sweet spot: 4GB film = 2000 chunków, bitfield = 250 bajtów, sensowna granularność

**Chunk size jest per-resource** i komunikowany w Bitfield message (total_chunks + resource size pozwala wyznaczyć chunk size). W fazie 0 jest stały (2MB). Konfigurowalny chunk size to potencjalny KIP extension.

## 8. Transport interface

Protokół jest transport-agnostic. Implementacja musi dostarczyć:

```go
type Transport interface {
    Send(msg []byte) error
    Recv() ([]byte, error)
    Close() error
}
```

Konkretne transporty to osobne komponenty:
- **WebRTC data channel** (US-0012) — domyślny, NAT traversal
- **HTTP** (LAN) — localhost/mDNS
- **In-memory pipe** (US-0014) — testy

Protokół NIE zarządza transportem. Nie wie jak połączenie powstało (mDNS? STUN? relay?). Dostaje Transport interface i wymienia wiadomości.

## 9. Zachowania obowiązkowe (MUST)

1. Peer MUSI wysłać Handshake jako pierwszą wiadomość.
2. Peer MUSI wysłać Bitfield dla każdego resource_id zadeklarowanego w Handshake.
3. Peer NIE MOŻE wysyłać Piece bez uprzedniego Request od drugiej strony.
4. Peer NIE MOŻE wysyłać Request dla resource, dla którego nie wysłał Interested.
5. Peer MUSI odpowiadać na Request w ciągu 30 sekund (Piece lub Error).
6. Peer MUSI wysyłać KeepAlive co 30s przy braku innego ruchu.
7. Peer MUSI zamknąć połączenie po wysłaniu Error.
8. Peer MUSI ignorować nieznane message types (nie Error, nie disconnect — umożliwia forward compat).
9. Peer MUSI respektować max 16 concurrent requests limit.

## 10. Zachowania zalecane (SHOULD)

1. Peer POWINIEN stosować pipelining (wiele Request zanim przyjdzie Piece).
2. Peer POWINIEN randomizować kolejność chunk requests (unikanie hot spots).
3. Peer POWINIEN trackować bandwidth per peer i preferować szybszych.
4. Peer POWINIEN reagować na Cancel przerywając wysyłanie Piece.

## 11. Poza scope KIP-0001

- **Peer discovery** → KIP-0002
- **Extension negotiation** → KIP-0003
- **Auth, encryption, streaming** → KIP-0004+
- **Chunk integrity verification** (hashing) → potencjalny KIP extension
- **Choking/unchoking** (BT-style bandwidth allocation) → nie potrzebne przy 1-5 peerów, ewentualnie KIP extension w przyszłości
- **Piece selection strategy** (rarest-first, sequential) → implementacja, nie spec. Streaming plugin (KIP-0006) overriduje na sequential.

## Pytania otwarte

1. Czy chunk size powinien być w Handshake (per-connection) czy w Bitfield (per-resource)?
2. Czy ResourceUpdate REMOVE powinien natychmiast anulować in-flight Requests dla tego resource'a?
3. Czy Error powinien mieć opcjonalny resource_id field (error per-resource vs per-connection)?
