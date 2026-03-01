# Pomysł: Tracker manifest — tracker jako autorytet federacji

**Status:** Aktywny — wpływa na KIP-0002

## Idea

Tracker (REGISTRY) nie jest głupim relay. Jest **autorytetem który definiuje reguły komunikacji** w swojej sieci. Federacja, nie demokracja.

Peer łączy się z trackerem i dostaje **manifest** — zestaw reguł: jakie extensiony są wymagane, jak ogłaszać zasoby, jak się łączyć z innymi peerami, push czy pull. Peer albo akceptuje reguły albo odchodzi.

## Konsekwencje

### Tracker decyduje, nie peer

- Tracker mówi: "w mojej sieci auth jest wymagany" → peer dostarcza JWT albo zostaje odrzucony
- Tracker mówi: "ogłaszaj się jako host, nie listuj plików" → peer mówi "jestem X, hostuję serwer A" zamiast wysyłać 50000 resource_ids
- Tracker mówi: "nowi peerzy dostają push o zmianach" → peer nasłuchuje
- Tracker mówi: "połączenia przez WebRTC, signaling przeze mnie" → peer używa signaling tego trackera

### Różne trackery, różne reguły

| Tracker | Reguły |
|---|---|
| Nasz (kippit.net) | Auth required, encryption required, push notifications, announce per-server |
| Publiczny (community) | No auth, no encryption, announce per-file, pull only |
| Korporacyjny (self-hosted) | Auth required, encryption required, no external peers, LAN only |
| BT bridge tracker | BT-compatible announce format, no KIP extensions |

RFC definiuje **mechanizm manifestu** (jak tracker komunikuje reguły), nie **konkretne reguły** (to decyzja operatora).

### Nie demokracja

W blockchainie peery negocjują konsensus. Tu nie. Tracker jest federatorem — jak serwer Matrix, jak instancja Mastodon. Operator trackera ustala politykę. Peer wybiera z jakim trackerem chce pracować.

RFC MOŻE definiować demokratyczne mechanizmy (np. peer voting na reguły) jako opcjonalny extension. Ale core model to federacja.

### Announce: hosty, nie pliki

Przy 50000 plików peer nie powinien ogłaszać listy plików. Zamiast tego:

```
"Jestem peer X, hostuję pliki z serwera A, B, C"
```

Lista plików to detal dostępny przez API (faza 1+) lub bezpośrednio od peera (wire protocol). Tracker wie KTO hostuje, nie CO hostuje.

Ale: tracker MOŻE wymagać per-file announce (np. publiczny tracker do sharing). To zależy od manifestu.

### Wpływ na KIP-0002

Sekcja 4.2 (hello handshake) powinna być przebudowana jako **manifest delivery**:

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

Peer odpowiada akceptacją lub rozłącza się.

## Powiązane

- [protocol-layer-model.md](protocol-layer-model.md) — model warstwowy (tracker = REGISTRY rola)
- [KIP-0002](KIP-0002-peer-discovery.md) — discovery & signaling protocol (manifest w sekcji 4.2)
