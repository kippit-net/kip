# KIP-0002: Discovery & Signaling Protocol (Layer 1 + Layer 2)

**Status:** Draft
**Version:** 0.2
**Component:** spec

## Abstract

KIP-0002 definiuje dwa interfejsy:
1. **Discovery** (warstwa 1) — jak PEER znajduje inne PEERy. Kto ma resource X?
2. **Signaling** (warstwa 2) — jak PEER nawiązuje połączenie z innym PEERem za NATem.

Spec definiuje **interfejs abstrakcyjny** + **referencyjną implementację** (KIP tracker protocol). Inne implementacje (BT tracker, mDNS, DHT, static peer list) są dopuszczalne jeśli spełniają interfejs.

## 1. Role

| Rola | Opis | Mówi protokołem |
|---|---|---|
| PEER | Ma dane lub chce dane. Klient discovery i signaling. | KIP-0001 (wire) + discovery client |
| REGISTRY | Wie kto ma co. Odpowiada na zapytania discovery. | Discovery protocol (server-side) |
| SIGNALER | Przekazuje wiadomości connection setup między PEERami. | Signaling protocol |
| RELAY | Przekazuje dane gdy P2P niemożliwy. Transparent proxy. | KIP-0001 (wire) |

Jeden node może pełnić wiele ról. Nasza implementacja: server = REGISTRY + SIGNALER + RELAY.

## 2. Discovery interface (warstwa 1)

### 2.1 Abstrakcyjny interfejs

Każda implementacja discovery MUSI dostarczyć:

```
DiscoveryProvider:
  Announce(peer_id, resources[]) → ack/error
  Want(resource_id) → PeerInfo[]
  Leave(resource_id) → ack/error

PeerInfo:
  peer_id: bytes[16]
  capabilities: string[]       // np. ["signaling", "relay", "direct-http"]
  connection_hint: string      // jak się połączyć (adres, transport type)
```

Nie narzuca transportu. Implementacja może być WebSocket, HTTP, UDP, plik na dysku, cokolwiek.

### 2.2 Implementacje discovery

| Implementacja | Transport | Scope | Uwagi |
|---|---|---|---|
| **KIP tracker** | WebSocket (sekcja 4) | WAN | Referencyjna. Nasz server. |
| **mDNS/DNS-SD** | UDP multicast | LAN | Zero-config, sekcja 5 |
| **BT tracker** | HTTP (BEP 3) | WAN | Przez bridge extension |
| **DHT** | UDP (BEP 5-like) | WAN | Przyszłe, decentralized |
| **Static peer list** | Plik / env var | Dowolny | Dev/testy |

PEER może używać wielu implementacji jednocześnie (resolver chain, US-0015).

## 3. Signaling interface (warstwa 2)

### 3.1 Abstrakcyjny interfejs

Signaling jest potrzebny gdy dwaj PEERy nie mogą połączyć się bezpośrednio (NAT). SIGNALER przekazuje wiadomości setup.

```
SignalingProvider:
  Signal(to_peer_id, payload) → ack/error

payload: opaque bytes — spec nie wie co jest w środku.
         WebRTC: SDP offer/answer, ICE candidates.
         Inny transport: cokolwiek potrzebuje.
```

Signaling jest **transport-specific**. WebRTC potrzebuje SDP/ICE exchange. TCP nie potrzebuje signaling (direct connect). In-memory pipe nie potrzebuje signaling.

### 3.2 Kiedy signaling jest potrzebny

| Transport (warstwa 2) | Wymaga signaling? |
|---|---|
| WebRTC data channel | TAK — SDP offer/answer + ICE candidates |
| TCP direct (publiczny IP) | NIE |
| HTTP LAN (mDNS) | NIE |
| In-memory pipe (testy) | NIE |
| QUIC (przyszłe) | Zależy od NAT |

Jeśli discovery zwraca `connection_hint` z wystarczającą informacją do direct connect, signaling nie jest potrzebny.

## 4. KIP Tracker Protocol (referencyjna implementacja)

Implementacja REGISTRY + SIGNALER w jednym endpoincie. To jest to czym mówi nasz server.

### 4.1 Transport

Persistent bidirectional connection. Referencyjna: WebSocket (`wss://`/`ws://`). Ale spec nie wymaga WebSocket — implementacja może użyć gRPC stream, TCP z framing, SSE + POST, etc.

### 4.2 Manifest (REGISTRY → PEER)

Po nawiązaniu połączenia transportowego, REGISTRY wysyła **manifest** — zestaw reguł obowiązujących w tej sieci. REGISTRY jest autorytetem — to federacja, nie demokracja. PEER akceptuje reguły albo rozłącza się.

```json
{
  "type": "manifest",
  "version": 1,
  "rules": {
    "auth_required": false,
    "announce_mode": "per-resource",
    "signaling": "provided",
    "relay": "none",
    "push_peers": false,
    "required_extensions": [],
    "encryption_required": false
  }
}
```

PEER odpowiada akceptacją:

```json
{
  "type": "accept",
  "version": 1,
  "peer_id": "a1b2c3d4e5f6..."
}
```

#### Pola manifestu

| Pole | Wartości | Opis |
|---|---|---|
| `auth_required` | bool | Czy announce wymaga tokenu auth (KIP-0004) |
| `announce_mode` | `"per-resource"`, `"per-server"` | Czy peer ogłasza listę resource_ids czy tylko "jestem hostem serwerów X, Y" |
| `signaling` | `"provided"`, `"none"` | Czy REGISTRY oferuje relay signaling (SIGNALER rola) |
| `relay` | `"available"`, `"none"` | Czy REGISTRY oferuje chunk relay (RELAY rola) |
| `push_peers` | bool | Czy REGISTRY proaktywnie informuje o nowych peerach czy czeka na want |
| `required_extensions` | int[] | Extensiony wymagane do uczestnictwa w sieci |
| `encryption_required` | bool | Czy szyfrowanie jest obowiązkowe |

**`announce_mode`** rozwiązuje problem 50000 plików:
- `"per-resource"`: peer ogłasza listę resource_ids (jak BT tracker — dobry dla małych bibliotek, sharing)
- `"per-server"`: peer ogłasza się jako host (np. "jestem runner X z serwerami A, B" — lista plików dostępna przez API lub bezpośrednio od peera)

Różne trackery mogą mieć różne manifesty:

| Tracker | Manifest |
|---|---|
| Nasz (kippit.net, faza 1) | auth required, per-server, signaling provided, push |
| Nasz (faza 0) | no auth, per-resource, signaling provided, pull |
| Publiczny community | no auth, per-resource, no relay, pull |
| Korporacyjny self-hosted | auth required, encryption required, per-server |

Operator trackera ustala politykę. RFC definiuje mechanizm manifestu, nie konkretne reguły.

#### Governance model

Manifest MOŻE zawierać pole `governance` definiujące kto ustala reguły:

| Governance | Znaczenie |
|---|---|
| `"federated"` | REGISTRY narzuca reguły. Peer akceptuje lub odchodzi. **Default.** |
| `"democratic"` | Swarm negocjuje reguły konsensusem. REGISTRY to bootstrap node, nie autorytet. |
| `"hybrid"` | REGISTRY proponuje defaults, swarm może override na wybranych polach. |

**Demokracja może zdecydować o wszystkim** — w tym o wymaganiu auth, szyfrowania, czy przejściu na federację. Swarm głosuje "auth required" → wyznacza node który tego pilnuje → ten node de facto jest federatorem. Demokracja może zdecydować o federacji. Federator może oddać głos swarmowi. Governance jest dynamiczny.

**Ograniczenia demokracji w P2P nie dotyczą CZEGO można przegłosować, ale JAK głosować i KTO egzekwuje:**

1. **Sybil attack** — kto ma prawo głosu? Jeśli 1 peer = 1 głos, atakujący tworzy 1000 peerów i przegłosowuje. Mechanizm tożsamości (kto jest prawdziwym peerem) wymaga auth — ale auth to reguła o którą dopiero głosujemy. Problem kurczaka i jajka.

2. **Enforcement** — kto wyrzuca peera który ignoruje wynik głosowania? Jeśli nikt — reguły są niewiążące. Jeśli wyznaczony node — ten node jest de facto federatorem. Demokracja w momencie enforcement **konwerguje do federacji**.

3. **Bootstrap** — na starcie nie ma peerów, nie ma kto głosować. Ktoś musi postawić pierwszy node z jakimiś początkowymi regułami.

W praktyce demokracja P2P ma dwie formy enforcement:
- **Mutual enforcement** — każdy peer wymusza reguły na swoich połączeniach (np. "nie gadam z niezaszyfrowanymi peerami"). Brak centralnego egzekutora. Działa dla szyfrowania, bandwidth, chunk verification.
- **Delegated enforcement** — swarm wyznacza node do pilnowania reguł. Ten node staje się federatorem z mandatem demokratycznym. Działa dla auth, moderation, commerce.

RFC definiuje mechanizm governance — wybór modelu to decyzja operatora/społeczności. Governance może się zmieniać w czasie życia sieci.

### 4.3 Discovery messages

#### announce (PEER → REGISTRY)

```json
{
  "type": "announce",
  "resources": [
    {
      "resource_id": "deadbeef01234567...",
      "chunk_count": 2048,
      "bitfield": "//////////8="
    }
  ]
}
```

- **resources:** lista seedowanych resource'ów.
- **bitfield:** base64-encoded. Opcjonalny — brak = 100% (pełny seed).
- **peer_id:** z handshake, nie powtarzany.

PEER POWINIEN wysłać announce po handshake. PEER MOŻE wysyłać ponownie z aktualizacjami.

#### want (PEER → REGISTRY)

```json
{
  "type": "want",
  "resource_id": "deadbeef01234567..."
}
```

REGISTRY odpowiada `peers`.

#### peers (REGISTRY → PEER)

```json
{
  "type": "peers",
  "resource_id": "deadbeef01234567...",
  "peers": [
    {
      "peer_id": "f6e5d4c3b2a1...",
      "connection_hint": "webrtc:signal-required"
    }
  ]
}
```

- **connection_hint:** jak połączyć się z peerem. Format zależy od transportu:
  - `"webrtc:signal-required"` — wymaga signaling przez SIGNALER
  - `"http:192.168.1.50:9000"` — direct HTTP (LAN)
  - `"tcp:1.2.3.4:6881"` — direct TCP (publiczny IP)

#### leave (PEER → REGISTRY)

```json
{
  "type": "leave",
  "resource_id": "deadbeef01234567..."
}
```

Opcjonalny — disconnect też czyści peera.

### 4.4 Signaling messages

#### signal (PEER → SIGNALER → PEER)

```json
{
  "type": "signal",
  "to": "f6e5d4c3b2a1...",
  "payload": { }
}
```

SIGNALER:
1. Dopisuje `"from": "<sender_peer_id>"` do message
2. Przekazuje do docelowego PEERa bez inspekcji payloadu
3. Jeśli docelowy PEER nie istnieje → error `PEER_NOT_FOUND`

Payload jest **opaque** — spec nie wie co jest w środku. Dla WebRTC to SDP/ICE. Dla innego transportu to cokolwiek connection setup potrzebuje.

### 4.5 Error messages

```json
{
  "type": "error",
  "code": "PEER_NOT_FOUND",
  "message": "Target peer is not connected"
}
```

| Code | Opis |
|---|---|
| PEER_NOT_FOUND | Docelowy peer signaling nie istnieje |
| INVALID_MESSAGE | Malformed message lub brak wymaganych pól |
| RATE_LIMITED | Za dużo wiadomości |
| AUTH_REQUIRED | REGISTRY wymaga auth, peer nie dostarczył |
| UNSUPPORTED_VERSION | Wersja protokołu nieobsługiwana |

### 4.6 State management

REGISTRY trzyma state **wyłącznie w pamięci**. Restart = utrata stanu. PEERy re-announce'ują po reconnect.

```
State:
  swarms: Map<resource_id, Set<PeerInfo>>
  connections: Map<peer_id, Connection>

PeerInfo:
  peer_id: bytes[16]
  resources: resource_id[]
  bitfield: Map<resource_id, Uint8Array>
  last_seen: timestamp
  connection_hint: string
```

**Eviction:** PEER bez aktywności przez 90 sekund = usunięty.

**KeepAlive:** REGISTRY wysyła ping co 30 sekund. Brak pong = disconnect.

## 5. mDNS Discovery (LAN)

Implementacja discovery interface (sekcja 2) dla sieci lokalnej. Nie wymaga REGISTRY — PEERy ogłaszają się bezpośrednio.

### 5.1 Service announcement

```
Service Type:  _kippit._tcp
Instance Name: <peer_id_hex_first_8_chars>
Port:          zdefiniowany w config (default 9000)
TXT Records:
  v=1                          (protocol version)
  resources=<res_id1>,<res_id2> (comma-separated, max 10)
```

### 5.2 Discovery flow

1. PEER wysyła mDNS query: `_kippit._tcp.local.`
2. LAN PEERy odpowiadają SRV + TXT records
3. Discoverer sprawdza `resources` w TXT
4. Jeśli match: direct HTTP connection (brak signaling, brak wire protocol overhead)

### 5.3 LAN chunk endpoint

```
GET /chunks/{resource_id}/{chunk_index}
→ 200 OK, body = raw chunk data
→ 404 Not Found
```

Shortcut — nie wymaga wire protocol handshake (KIP-0001). Prostszy, mniej overhead dla LAN.

## 6. Combined flow (wszystkie warstwy)

```
PEER A chce resource R:

Warstwa 1 (DISCOVERY):
  1. mDNS cache → peer B w LAN? → TAK → skip to warstwa 2 direct
  2. REGISTRY: want {resource_id: R}
     REGISTRY: peers {R, [{peer_id: B, hint: "webrtc:signal-required"}]}

Warstwa 2 (CONNECTION):
  Jeśli LAN:
    3a. HTTP GET http://192.168.1.50:9000/chunks/R/0 → done (shortcut)

  Jeśli remote (WebRTC):
    3b. SIGNALER: signal {to: B, payload: {type: "offer", sdp: ...}}
        SIGNALER → B: signal {from: A, payload: {type: "offer", sdp: ...}}
        B → SIGNALER: signal {to: A, payload: {type: "answer", sdp: ...}}
        SIGNALER → A: signal {from: B, payload: ...}
        (+ ICE candidates)
    4. WebRTC data channel established → Transport interface ready

Warstwa 3 (EXCHANGE):
  5. KIP-0001 handshake → Bitfield → Request → Piece → ...

Warstwa 4 (SEMANTICS):
  6. Extensions: auth verification, decryption, sequential playback, ...
```

## 7. Extensions na discovery protocol

KIP-0003 definiuje extension negotiation dla warstwy 3 (PEER ↔ PEER). Discovery protocol ma **osobny** mechanizm: **capability advertisement** w handshake (sekcja 4.2).

Przykłady discovery extensions:

| Extension | Co robi | Handshake |
|---|---|---|
| KIP-0004 (Auth) | REGISTRY wymaga JWT w announce | `auth_required: true` |
| Relay | REGISTRY oferuje chunk relay | `relay_available: true` |
| Push notifications | REGISTRY informuje o nowych peerach | `push_peers: true` |
| Federation | REGISTRY synchronizuje z innymi REGISTRY | `federation: true` |

Discovery extensions ≠ wire protocol extensions. To osobne namespace'y, osobna negocjacja.

## 8. Poza scope KIP-0002

- **Wire protocol** → KIP-0001 (warstwa 3)
- **Extension negotiation peer ↔ peer** → KIP-0003 (warstwa 3)
- **Auth details** (JWT format, verification) → KIP-0004
- **Chunk relay implementation** → osobny spec lub część KIP-0004/tracker docs
- **Transport implementations** (WebRTC, TCP, QUIC) → poza spec, pluggable

## Pytania otwarte

1. **`want` batch:** czy wspierać `want` z wieloma resource_ids?
2. **Push vs pull:** czy REGISTRY proaktywnie informuje o nowych peerach (push) czy tylko na `want` (pull)?
3. **Resource limit:** max resources per peer w announce? Peer z 10000 plików = duży message.
4. **Discovery + signaling razem czy osobno?** Obecny draft: razem w jednym connection (prostsze). Alternatywa: osobne endpointy (czystszy separation of concerns). Nasza implementacja i tak ma jedno połączenie.
5. **connection_hint format:** formalizować czy free-form string?
