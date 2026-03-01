# Pomysł: Model warstwowy protokołu KIP

**Status:** Aktywny — obecny kierunek architektoniczny

## Kontekst

Próba opisania protokołu jako jednego monolitu (KIP-0001 + KIP-0002 + KIP-0003) prowadzi do pomieszania warstw. Tracker jest opisany jako "WebSocket JSON API" zamiast jako rola w protokole. Extension negotiation działa tylko między peerami, nie między peerem a trackerem. Nie wiadomo gdzie żyje WebRTC, HLS, BT — bo model nie rozróżnia warstw.

## Idea: KIP to stack protokołów, nie jeden protokół

Jak TCP/IP nie jest jednym protokołem, KIP definiuje warstwy i interfejsy między nimi.

### Warstwy

```
┌─────────────────────────────────────────────────────────────────┐
│  5. APPLICATION                                                 │
│     "Oglądam film" / "Syncuję pliki" / "Udostępniam mamie"     │
│     CLI, Web App, Mobile — NIE w spec protokołu                 │
├─────────────────────────────────────────────────────────────────┤
│  4. SEMANTICS                                                   │
│     Co dane ZNACZĄ. Extensiony żyją tutaj.                      │
│                                                                 │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│     │Streaming │ │  Sync    │ │Messaging │ │   Auth   │ ...   │
│     │HLS/m3u8  │ │op log,  │ │real-time │ │JWT, TLS  │       │
│     │sequential│ │vector   │ │channels  │ │encryption│       │
│     │priority  │ │clocks   │ │          │ │          │       │
│     └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├─────────────────────────────────────────────────────────────────┤
│  3. EXCHANGE                                                    │
│     Handshake, bitfield, request, piece, cancel.                │
│     Transport-agnostic. Binarny framing.                        │
│                                                                 │
│     Implementacje:                                              │
│     ┌────────────────┐  ┌─────────────────┐                    │
│     │ KIP wire       │  │ BT wire (bridge)│                    │
│     └────────────────┘  └─────────────────┘                    │
├─────────────────────────────────────────────────────────────────┤
│  2. CONNECTION                                                  │
│     Jak nawiązuję kanał z peerem. NAT traversal tu żyje.       │
│                                                                 │
│     ┌────────┐ ┌──────┐ ┌──────┐ ┌──────────┐ ┌────────────┐  │
│     │WebRTC  │ │ TCP  │ │ HTTP │ │ In-memory│ │ QUIC/inne  │  │
│     │data ch.│ │      │ │ LAN  │ │ (testy)  │ │ (przyszłe) │  │
│     └────────┘ └──────┘ └──────┘ └──────────┘ └────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  1. DISCOVERY                                                   │
│     Gdzie są peery? Kto ma to czego szukam?                    │
│                                                                 │
│     ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐ ┌───────────┐ │
│     │KIP      │ │BT        │ │ mDNS │ │ DHT  │ │ static    │ │
│     │tracker  │ │tracker   │ │      │ │      │ │ peer list │ │
│     └──────────┘ └──────────┘ └──────┘ └──────┘ └───────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

Każda warstwa ma wiele implementacji. Protokół definiuje interfejsy między warstwami.

### Role w protokole

Node w sieci pełni role. Rola to zestaw capability + protokół którym mówi. Rola to nie binarka.

```
PEER        Ma dane, chce dane. Mówi wire protocol (warstwa 3).
REGISTRY    Wie kto ma co. Mówi discovery protocol (warstwa 1).
SIGNALER    Przekazuje SDP/ICE. Mówi signaling protocol (warstwa 2).
RELAY       Przekazuje dane gdy P2P niemożliwy. Mówi wire protocol (warstwa 3).
```

Każdy node może pełnić dowolną kombinację ról:

| Binarka | Role | Opis |
|---|---|---|
| Runner (nasz, Go) | PEER | Seeduje/pobiera chunki, mówi wire + discovery client |
| Server (nasz, Node.js) | REGISTRY + SIGNALER + RELAY | Swarm state, SDP relay, chunk relay, API |
| Fat node (czyjaś impl) | PEER + REGISTRY | Peer który jest jednocześnie trackerem |
| BT tracker adapter | REGISTRY | Tłumaczy KIP discovery ↔ BT tracker HTTP |
| Przeglądarka | PEER | WebRTC peer, klient discovery |
| CLI tool | PEER | Testowy peer, in-memory lub TCP |

### Nasza implementacja: 2 binarki

```
runner (Go)                          server (Node.js)
┌──────────────────────┐            ┌──────────────────────┐
│ PEER                 │            │ REGISTRY             │
│  - seeduje chunki    │◄──────────►│  - swarm state       │
│  - pobiera chunki    │  discovery │  - peer matching     │
│  - wire protocol     │  protocol  │                      │
│                      │            │ SIGNALER             │
│ (extensions:)        │◄──────────►│  - SDP/ICE relay     │
│  - streaming (HLS)   │  signaling │                      │
│  - encryption (AES)  │  protocol  │ RELAY (opcjonalnie)  │
│  - auth (JWT)        │            │  - chunk forwarding  │
│  - sync              │            │                      │
│  - bt-bridge         │            │ API (poza protokołem)│
│                      │            │  - REST: auth, lib   │
└──────────────────────┘            │  - DB: PostgreSQL    │
                                    └──────────────────────┘
```

Ale spec nie wie o Go ani Node.js. Spec mówi jak PEER gada z REGISTRY i jak PEER gada z PEER.

### Interfejsy do zdefiniowania w spec

| Interfejs | Kto ↔ Kto | KIP |
|---|---|---|
| Wire protocol | PEER ↔ PEER | KIP-0001 (istnieje, draft) |
| Discovery protocol | PEER ↔ REGISTRY | KIP-0002 (do przepisania) |
| Signaling protocol | PEER ↔ SIGNALER | Część KIP-0002 lub osobny KIP |
| Extension negotiation | PEER ↔ PEER, PEER ↔ REGISTRY | KIP-0003 (do rozszerzenia) |
| Transport interface | Wewnętrzny | Nie spec — interfejs implementacyjny |

RELAY nie potrzebuje osobnego spec — jest transparent proxy na wire protocol.

### Jak istniejące standardy wchodzą w model

```
Standard         Warstwa    Rola w systemie
─────────        ───────    ───────────────
WebRTC           2          Transport (data channel) + signaling (ICE/STUN)
TCP              2          Transport (BT compat, LAN)
HTTP             2          Transport (LAN chunks) + API (poza protokołem)
HLS/m3u8         4          Semantics — streaming extension generuje manifest
JWT              4          Semantics — auth extension weryfikuje tokeny
AES              4          Semantics — encryption extension szyfruje chunki
BT wire          3          Exchange — bridge extension tłumaczy ↔ KIP wire
BT tracker       1          Discovery — bridge tłumaczy ↔ KIP discovery
mDNS             1          Discovery — LAN, zero-config
STUN/TURN        2          Connection — NAT traversal
```

Żaden z nich nie jest "konkurencją" — działają na różnych warstwach.

### Scenariusze end-to-end

**"Oglądam film z NASa poza domem":**
```
Discovery:    KIP tracker → "Runner A ma resource X"
Connection:   WebRTC (ICE/STUN) → data channel
Exchange:     KIP wire → request chunks
Semantics:    Streaming ext (sequential) + Auth ext (JWT) + Encryption ext (AES)
Application:  HLS.js w przeglądarce
```

**"Pobieram torrent":**
```
Discovery:    BT tracker → "Peers B,C,D"
Connection:   TCP
Exchange:     BT wire → bridge ext → KIP internal
Semantics:    brak — raw download
Application:  plik na dysku
```

**"Syncuję 2 NASy":**
```
Discovery:    KIP tracker → "Runner B online"
Connection:   WebRTC lub LAN HTTP
Exchange:     KIP wire
Semantics:    Sync ext (op log) + Auth ext + Encryption ext
Application:  Runner B ma kopię
```

### Otwarte pytania

1. **Discovery + signaling: razem czy osobno w spec?** W naszej implementacji to jeden WebSocket. Ale koncepcyjnie to dwie role (REGISTRY vs SIGNALER). Czy jeden KIP czy dwa?

2. **Discovery protocol wire format:** Definiujemy konkretny format (jak KIP-0001 definiuje binarny framing) czy abstrakcyjny interfejs + referencyjna implementacja?

3. **Capability advertisement:** REGISTRY powinno w handshake powiedzieć peerowi: "wymagam auth, oferuję relay, nie oferuję signaling". Peer decyduje czy chce gadać.

4. **Extension negotiation na discovery protocol:** KIP-0003 teraz działa tylko peer ↔ peer. Czy REGISTRY też negocjuje extensiony? Czy to osobny mechanizm (capability advertisement)?

## Powiązane

- [tracker-manifest-authority.md](tracker-manifest-authority.md) — tracker jako autorytet federacji
- KIP drafty: [KIP-0001](KIP-0001-wire-protocol.md), [KIP-0002](KIP-0002-peer-discovery.md), [KIP-0003](KIP-0003-extension-negotiation.md)
