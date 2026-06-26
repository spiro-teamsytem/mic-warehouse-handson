# Implementation Plan — Article domain layer (Phase 03)

> Piano di build per il domain layer del BC Warehouse. Esegue il modello descritto in
> [`domain-model.md`](./domain-model.md). Target: un modulo Go (`warehouse.local/core`, Go 1.22)
> con solo il domain layer + un health endpoint. **Niente DB, niente HTTP applicativo.** (Regola #6)

---

## 1. Layout dei package

```text
phase-03-skeleton/
├── go.mod                     // module warehouse.local/core, go 1.22
├── main.go                    // Echo: solo GET /health → {"status":"ok"} su :8081
├── entities/
│   ├── sku.go                 // VO SKU + NewSKU (factory)
│   ├── money.go               // VO Money + NewMoney (factory)
│   ├── inventory.go           // entità InventoryLevel + invarianti
│   └── article.go             // aggregate root Article + factory + behaviors + eventi pending
├── events/
│   └── events.go              // DomainEvent + ArticleCreated / InventoryAdjusted / StockReserved
└── interfaces/
    └── repository.go          // porta ArticleRepository (solo interfaccia)
```

Regola delle dipendenze (Clean Architecture, ADR-007): `entities` non importa nulla;
`interfaces` importa `entities`; `main.go` è l'edge e non viene importato da nessuno. Nessun package
del dominio importa driver DB o framework web.

---

## 2. Sequenza dei task

Ordinata per dipendenze: dal più interno (VO) al più esterno (porta + bootstrap). Ogni task si chiude
**verde** (test propri) prima di passare al successivo.

| # | Task | File | Output | Regola |
|---|------|------|--------|--------|
| T0 | Bootstrap modulo Go + health endpoint | `go.mod`, `main.go` | `go build` ok; `curl /health` → `{"status":"ok"}` | #6 |
| T1 | VO `SKU` con factory + validazione regex | `entities/sku.go` | `NewSKU` rifiuta input invalido; uguaglianza by value | #3 |
| T2 | VO `Money` con factory + validazione | `entities/money.go` | `NewMoney` rifiuta cents<0 / currency non-ISO; by value | #3 |
| T3 | Entità `InventoryLevel` + invarianti stock | `entities/inventory.go` | `quantity≥0`, `reserved≥0`, `reserved≤quantity` | #2 |
| T4 | `DomainEvent` + 3 eventi | `events/events.go` | tipi past-tense immutabili | #4 |
| T5 | Aggregate root `Article`: factory + invarianti | `entities/article.go` | `NewArticle` fallisce loud; registra `ArticleCreated` | #1, #4 |
| T6 | Behaviors di `Article`: `ChangePrice`, `AdjustInventory`, `ReserveStock`, `PullEvents` | `entities/article.go` | currency immutabile; stock muta solo via root; eventi registrati | #1, #2, #4 |
| T7 | Porta `ArticleRepository` (solo interfaccia) | `interfaces/repository.go` | `Save` idempotente + find/list/delete; nessun `InventoryRepository` | #5 |
| T8 | Suite di test invarianti completa + run finale | `*_test.go` | tutte le invarianti coperte e verdi; service boota | tutte |

> Suggerimento: T1→T2 sono indipendenti e possono procedere in parallelo. T5/T6 dipendono da T1-T4.

---

## 3. Signature di riferimento (orientative, non vincolanti)

> La *forma* del codice è libera (lo dice il brief di Phase 03): qui si fissano i contratti, non
> l'implementazione.

```go
// entities/sku.go
type SKU struct{ Code string }
func NewSKU(code string) (SKU, error)        // ^[A-Z0-9-]{3,32}$, altrimenti error

// entities/money.go
type Money struct{ AmountCents int64; Currency string }
func NewMoney(cents int64, currency string) (Money, error)  // cents≥0, currency 3 maiuscole
func (m Money) Equals(o Money) bool

// entities/inventory.go
type InventoryLevel struct{ ID, LocationCode string; Quantity, Reserved int }
// muta solo via Article; valida quantity≥0, reserved≥0, reserved≤quantity

// entities/article.go
type Article struct{ /* ID, SKU, Name, Description, Price, levels, pendingEvents (privati) */ }
func NewArticle(id string, sku SKU, name, desc string, price Money) (*Article, error) // price>0, name≠""
func (a *Article) ChangePrice(newPrice Money) error    // stessa currency, >0
func (a *Article) AdjustInventory(loc string, delta int, reason string) error
func (a *Article) ReserveStock(loc string, qty int /*, ...*/) error
func (a *Article) PullEvents() []events.DomainEvent     // drena pendingEvents

// events/events.go
type DomainEvent interface { EventName() string; OccurredAt() time.Time }

// interfaces/repository.go
type ArticleRepository interface {
    Save(ctx context.Context, a *entities.Article) error  // idempotente per lo stesso stato
    FindByID(ctx context.Context, id string) (*entities.Article, error)
    FindBySKU(ctx context.Context, sku entities.SKU) (*entities.Article, error)
    List(ctx context.Context) ([]*entities.Article, error)
    Delete(ctx context.Context, id string) error
}
```

---

## 4. Test plan (la spec sono le invarianti, non una suite imposta)

| Invariante / regola | Caso di test (almeno) |
|---------------------|------------------------|
| `price > 0` | `NewArticle` con prezzo 0 → errore; con prezzo >0 → ok |
| `name` non vuoto | `NewArticle` con name `""` → errore |
| currency immutabile | `ChangePrice` con currency diversa → errore; stessa currency → ok |
| SKU regex | `"AB"`, `"ab-1"`, stringa >32 → errore; `"ABC-123"` → ok |
| Money | cents `-1` → errore; currency `"eur"`/`"EURO"` → errore; `(0,"EUR")` → ok come Money |
| stock sane | `reserved > quantity` → errore; `AdjustInventory` che porta quantity <0 → errore |
| accesso solo via root | non esiste API pubblica per mutare `InventoryLevel` fuori da `Article` |
| eventi registrati | dopo `NewArticle`, `PullEvents()` contiene `ArticleCreated`; seconda chiamata → lista vuota |
| no publish | nessun import di trasporto/broker nei package dominio |
| repository | esiste solo `ArticleRepository`; check a compile-time dell'interfaccia; `Save` due volte = stesso stato |

Stile: unit test sugli entity/VO (invarianti e mutazioni), check a compile-time sulla porta. Test
scritti contro la **propria** API (nessuna suite di riferimento da rispettare).

---

## 5. Definition of Done (criteri di accettazione)

Allineata 1:1 con le 6 regole del brief e con il README di Phase 03:

- [ ] **#1** `NewArticle` rifiuta name vuoto e prezzo ≤ 0; `ChangePrice` preserva la currency.
- [ ] **#2** `InventoryLevel` garantisce `reserved ≤ quantity` e `quantity ≥ 0`; lo stock si modifica solo via `Article`.
- [ ] **#3** `SKU` e `Money` non hanno identità, hanno factory che falliscono loud, confronto by value.
- [ ] **#4** L'aggregato registra eventi su `pendingEvents`; `PullEvents()` li drena; nessuna pubblicazione/serializzazione.
- [ ] **#5** Esiste solo `ArticleRepository` (aggregate-level, `Save` idempotente); nessun `InventoryRepository`.
- [ ] **#6** I package del dominio non importano DB né web; `main.go` espone solo `GET /health`.
- [ ] Il servizio boota (`docker compose up --build` → `/health` = `{"status":"ok"}` su `:8081`).
- [ ] Tutte le invarianti hanno test verdi (`docker compose run --rm test` oppure `go test ./...`).

---

## 6. Deliverable di design (atteso a fine Phase 03)

Oltre al codice, produrre `design-decisions.md` (mezza pagina) che renda esplicite le scelte
implicite, per ciascuna: **decisione**, **regola/confine protetto**, **alternativa scartata**,
**allineamento col target Clean Architecture**. Scelte da motivare:

1. Perché `Article` è l'unico aggregate root.
2. Perché `SKU` e `Money` non hanno identità.
3. Perché una sola porta repository e **nessun** `InventoryRepository`.
4. Perché gli eventi sono **registrati** e non pubblicati qui.

---

## 7. Fuori scope (lezioni successive)

| Non ora | Arriva in |
|---------|-----------|
| Implementazione MySQL + dual-write decorator | CP4 |
| Use case + dispatch eventi | CP5 |
| Handler HTTP `/articles` | CP6 |
| Auth, policy, eventi sul wire, observability | CP7–CP10 |
| `StockReservation` come entità, ledger `StockMovement` | descopati / Phase 05+ |
