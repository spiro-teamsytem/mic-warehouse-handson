# Test Plan — Article domain layer (Phase 03)

> Piano di scrittura dei test per **validare** il domain layer descritto in
> [`domain-model.md`](./domain-model.md) e costruito secondo
> [`implementation-plan.md`](./implementation-plan.md). La **spec sono le invarianti**: ogni regola
> del brief deve avere almeno un test che la dimostra verde e uno che dimostra il fallimento loud.

---

## 1. Strategia

Coerente con la testing strategy di [ADR-007](../adr/ADR-007-go-clean-architecture-implementation.md):
i layer interni si testano **senza** adapter esterni.

- **Unit test** su entity e value object: invarianti di costruzione e comportamento di mutazione.
- **Compile-time check** sulla porta `ArticleRepository` (e un fake in-memory di supporto ai test
  d'aggregato, non un'implementazione di produzione).
- **Nessun** test che tocchi DB o HTTP: non esistono in questa fase. (Regola #6)
- **Table-driven tests** (idioma Go) per i casi di validazione: una tabella di `{nome, input, wantErr}`.
- Approccio **TDD-friendly**: i test di ogni tipo si scrivono insieme al task che lo crea (T1→T8),
  non tutti alla fine.

> Forma libera: il brief di Phase 03 dice esplicitamente che non c'è una suite di riferimento da
> rispettare. Qui si fissano **quali comportamenti** vanno coperti, non i nomi esatti delle funzioni.

---

## 2. Layout dei file di test

```text
phase-03-skeleton/
├── entities/
│   ├── sku_test.go         // T1
│   ├── money_test.go       // T2
│   ├── inventory_test.go   // T3
│   └── article_test.go     // T5 + T6 (factory e behaviors)
├── events/
│   └── events_test.go      // T4
├── interfaces/
│   └── repository_test.go  // T7 — compile-time check + fake in-memory
└── architecture_test.go    // cross-cutting: nessun import vietato (Regola #6)
```

---

## 3. Suite per tipo (casi minimi da coprire)

### 3.1 `SKU` — `sku_test.go` (T1, Regola #3)

| Caso | Input | Atteso |
|------|-------|--------|
| valido tipico | `"ABC-123"` | ok |
| lunghezza minima | `"ABC"` (3) | ok |
| lunghezza massima | 32 char validi | ok |
| troppo corto | `"AB"` | errore |
| troppo lungo | 33 char | errore |
| minuscole | `"abc-1"` | errore |
| caratteri illegali | `"ABC_1"`, `"ABC 1"`, `"ABÇ"` | errore |
| vuoto | `""` | errore |
| **uguaglianza by value** | due `NewSKU("ABC-123")` | `==` / `Equals` true; istanze diverse, valore uguale |

### 3.2 `Money` — `money_test.go` (T2, Regola #3)

| Caso | Input | Atteso |
|------|-------|--------|
| valido | `(1099, "EUR")` | ok |
| **zero è valido come Money** | `(0, "EUR")` | ok (la regola prezzo>0 sta sull'aggregato) |
| cents negativi | `(-1, "EUR")` | errore |
| currency minuscola | `(100, "eur")` | errore |
| currency lunghezza ≠ 3 | `(100, "EU")`, `(100, "EURO")` | errore |
| currency non lettere | `(100, "E1R")` | errore |
| **uguaglianza by value** | due `NewMoney(100,"EUR")` | uguali; `(100,"EUR") != (100,"USD")` |

### 3.3 `InventoryLevel` — `inventory_test.go` (T3, Regola #2)

| Caso | Scenario | Atteso |
|------|----------|--------|
| costruzione valida | `quantity=10, reserved=3` | ok |
| quantity negativa | `quantity=-1` | errore |
| reserved negativa | `reserved=-1` | errore |
| reserved > quantity | `quantity=5, reserved=6` | errore |
| reserved == quantity | `quantity=5, reserved=5` | ok (boundary) |
| id/location vuoti | `ID=""` o location `""` | errore |

> Verifica anche che **non esista** un'API pubblica per costruire/mutare un `InventoryLevel`
> fuori da `Article` se la scelta di design è quella (vedi §3.5).

### 3.4 Domain Events — `events_test.go` (T4, Regola #4)

| Caso | Atteso |
|------|--------|
| ogni evento implementa `DomainEvent` | `EventName()` non vuoto, `OccurredAt()` valorizzato |
| naming past-tense | `ArticleCreated`, `InventoryAdjusted`, `StockReserved` |
| immutabilità | i campi sono valorizzati alla costruzione e non mutati (nessun setter) |
| payload | i campi canonici di [`domain-model.md` §6](./domain-model.md) sono presenti |

### 3.5 `Article` factory — `article_test.go` (T5, Regola #1)

| Caso | Input | Atteso |
|------|-------|--------|
| creazione valida | id, sku, name, price>0 | ok |
| name vuoto | `name=""` | errore |
| id vuoto | `id=""` | errore |
| **prezzo zero** | `price = Money(0,"EUR")` | errore (più stretto di Money) |
| prezzo valido | `price = Money(1,"EUR")` | ok |
| evento registrato | dopo `NewArticle` | `PullEvents()` contiene `ArticleCreated` |

### 3.6 `Article` behaviors — `article_test.go` (T6, Regole #1, #2, #4)

| Comportamento | Caso | Atteso |
|---------------|------|--------|
| `ChangePrice` | stessa currency, valore >0 | ok; registra `ArticlePriceChanged` *(se implementato)* |
| `ChangePrice` | **currency diversa** | errore (currency immutabile) |
| `ChangePrice` | valore ≤ 0 | errore |
| `AdjustInventory` | delta positivo | quantity aumenta; registra `InventoryAdjusted` |
| `AdjustInventory` | delta che porta quantity < 0 | errore, stato invariato |
| `ReserveStock` | `reserved+qty ≤ quantity` | ok; registra `StockReserved` |
| `ReserveStock` | `reserved+qty > quantity` | errore, stato invariato |
| accesso solo via root | mutazione stock | passa solo da metodi di `Article`, mai da `InventoryLevel` esposto |
| `PullEvents` (drena) | seconda chiamata consecutiva | prima ritorna gli eventi, seconda lista vuota |
| nessun publish | esecuzione dei behaviors | nessuna chiamata a broker/serializzazione (solo append in memoria) |

> **Atomicità delle invarianti**: per ogni caso di errore, asserire anche che lo **stato non è
> cambiato** (no mutazione parziale) e che **nessun evento** è stato registrato.

### 3.7 `ArticleRepository` — `repository_test.go` (T7, Regola #5)

| Caso | Atteso |
|------|--------|
| compile-time check | un fake in-memory soddisfa l'interfaccia (`var _ ArticleRepository = (*fakeRepo)(nil)`) |
| operazioni aggregate-level | esistono `Save`, `FindByID`, `FindBySKU`, `List`, `Delete` |
| **nessun `InventoryRepository`** | non esiste alcuna porta per `InventoryLevel` (verifica statica/di review) |
| `Save` idempotente | salvare due volte lo **stesso stato** → un solo articolo, stesso risultato |

### 3.8 Clean Architecture — `architecture_test.go` (Regola #6)

| Caso | Atteso |
|------|--------|
| import vietati | i package `entities/`, `events/`, `interfaces/` non importano driver DB né framework web |
| verifica | test che ispeziona gli import (es. `go/packages` o lista import) **oppure** check documentato in review |

---

## 4. Tracciabilità test → invariante → regola

| Regola brief | File/test che la copre |
|--------------|------------------------|
| #1 Article guards invariants | `article_test.go` §3.5 + `ChangePrice` §3.6 |
| #2 Stock stays sane | `inventory_test.go` §3.3 + `AdjustInventory`/`ReserveStock` §3.6 |
| #3 VO no identity | `sku_test.go` §3.1 + `money_test.go` §3.2 (uguaglianza by value, factory loud) |
| #4 Record events, don't publish | `events_test.go` §3.4 + `PullEvents`/no-publish §3.6 |
| #5 One aggregate, one repository | `repository_test.go` §3.7 |
| #6 Domain layer only | `architecture_test.go` §3.8 |

---

## 5. Sequenza di scrittura (allineata ai task T1–T8)

1. **T1/T2** — scrivere `sku_test.go` e `money_test.go` insieme alle factory dei VO.
2. **T3** — `inventory_test.go` con i casi di boundary (`reserved == quantity`).
3. **T4** — `events_test.go` appena definita l'interfaccia `DomainEvent`.
4. **T5** — casi factory di `article_test.go` (rifiuto loud + `ArticleCreated`).
5. **T6** — casi behavior di `article_test.go` (currency immutabile, stock, drain eventi, atomicità).
6. **T7** — `repository_test.go` (compile-time + fake + idempotenza di `Save`).
7. **T8** — `architecture_test.go` + run completa.

---

## 6. Definition of Done dei test

- [ ] Ogni regola #1–#6 ha **almeno** un test "happy path" verde e un test di **fallimento loud**.
- [ ] Ogni caso di errore asserisce **stato invariato** e **nessun evento registrato**.
- [ ] I boundary sono coperti (lunghezze 3/32 SKU, `cents=0`, `reserved==quantity`).
- [ ] La porta è verificata a compile-time; non esiste `InventoryRepository`.
- [ ] Nessun test importa DB/HTTP.
- [ ] `go test ./...` (o `docker compose run --rm test`) tutto verde, con il servizio che boota
      (`/health` = `{"status":"ok"}`).

---

## 7. Come eseguirli

```bash
cd phase-03-skeleton
docker compose run --rm test          # nel container
# oppure, con Go locale 1.22+:
go test ./... -v
go test ./... -cover                  # per vedere la copertura delle invarianti
```
