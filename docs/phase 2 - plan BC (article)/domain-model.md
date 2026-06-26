# Domain Model — Article (Warehouse BC)

> Modello estratto per la pianificazione del domain layer. È la traduzione operativa di
> [`tactical-ddd-warehouse.md`](../../phase-02-analysis/tactical-ddd-warehouse.md) focalizzata
> sull'**aggregato Article** e sulle **regole da persistere** (le 6 regole del brief).

---

## 1. Vista d'insieme

```text
            Article  (Aggregate Root, identity = ID)
            ├── SKU            value object   (no identity, by value)
            ├── Money price    value object   (no identity, by value)
            ├── name, description   primitivi
            ├── []InventoryLevel    entità interne (identity = ID, solo via root)
            └── pendingEvents []DomainEvent   (registrati, non pubblicati)

   interfaces.ArticleRepository  ← unica porta di persistenza (aggregate-level)
```

`Article` è l'**unico** aggregato esposto dal BC. `InventoryLevel` vive dentro di esso: non viene
mai caricato, salvato o modificato in autonomia. (Regole #2, #5)

---

## 2. Aggregato — `Article`

| Aspetto | Valore |
|---------|--------|
| Tipo | Aggregate Root |
| Identità | `ID` (UUID/string non vuota), uguaglianza **by identity** |
| Possiede | `SKU`, `Money price`, `name`, `description`, `[]InventoryLevel`, `pendingEvents` |
| Factory | `NewArticle(...)` — fallisce loud su input invalido |

**Invarianti (sempre vere, ad ogni istante):**

- `ID` non vuoto
- `name` non vuoto
- `price > 0` (più stretta di `Money`: zero è un `Money` valido, ma non un prezzo valido)
- `price.Currency` **non cambia mai** dopo la costruzione

**Comportamenti (l'unico punto d'ingresso per i chiamanti):**

| Metodo | Effetto | Evento registrato |
|--------|---------|-------------------|
| `NewArticle(id, sku, name, desc, price)` | costruisce + valida tutte le invarianti | `ArticleCreated` |
| `ChangePrice(newPrice)` | aggiorna il prezzo **mantenendo la currency** (un cambio di valuta è un flusso di migrazione separato); rifiuta `≤ 0` e currency diversa | `ArticlePriceChanged` *(pianificato)* |
| `AdjustInventory(location, delta, reason)` | varia `quantity` di un `InventoryLevel` mantenendo le invarianti di stock | `InventoryAdjusted` |
| `ReserveStock(location, qty, ...)` | incrementa `reserved` se `reserved+qty ≤ quantity` | `StockReserved` |
| `PullEvents()` | restituisce e svuota `pendingEvents` (drena, non pubblica) | — |

> Le mutazioni di stock passano **tutte** da metodi di `Article`, mai da `InventoryLevel`
> direttamente. (Regola #2)

---

## 3. Entità interna — `InventoryLevel`

| Aspetto | Valore |
|---------|--------|
| Tipo | Entity (dentro l'aggregato `Article`) |
| Identità | `ID`, uguaglianza by identity |
| Cardinalità | uno per coppia `(Article, location)` |
| Repository proprio | **nessuno** — caricato/salvato solo via `Article` |

**Invarianti:**

- `ID` e identificatori di articolo/location non vuoti
- `quantity ≥ 0`
- `reserved ≥ 0`
- `reserved ≤ quantity`

> Nota di scope: `StockReservation` resta un semplice contatore `Reserved` su `InventoryLevel`
> in questa fase (promosso a entità a sé in Phase 05+).

---

## 4. Value Object — `SKU`

| Aspetto | Valore |
|---------|--------|
| Dati | `Code string` |
| Identità | nessuna — uguaglianza **by value** |
| Validazione (factory `NewSKU`) | `^[A-Z0-9-]{3,32}$` (maiuscole, cifre, trattini; lunghezza 3–32) |

La factory rifiuta tutto il resto. Un `SKU` in mano all'aggregato è valido per costruzione:
ri-validare al punto d'uso è un *code smell*. (Regola #3)

---

## 5. Value Object — `Money`

| Aspetto | Valore |
|---------|--------|
| Dati | `AmountCents int64`, `Currency string` |
| Identità | nessuna — uguaglianza **by value** |
| Validazione (factory `NewMoney`) | `AmountCents ≥ 0`; `Currency` = 3 lettere maiuscole (ISO-4217) |

`Money(0, "EUR")` è valido come `Money`; un `Article` con prezzo zero **no** — la regola più stretta
vive sull'aggregato. (Regole #1, #3)

---

## 6. Domain Events

Fatti al **passato**, immutabili. L'aggregato li **registra**; un layer successivo (CP5) li drena e
pubblica. Qui non si pubblica né si serializza. (Regola #4)

Tutti implementano un'interfaccia comune `DomainEvent` (es. `EventName() string`,
`OccurredAt() time.Time`).

| Evento | Emesso quando | Payload canonico |
|--------|---------------|------------------|
| `ArticleCreated` | nuovo `Article` pubblicato | `articleId, sku, articleName, priceCents, currency, occurredAt` |
| `InventoryAdjusted` | `InventoryLevel.Quantity` cambia | `articleId, locationCode, delta, newQuantity, reason, occurredAt` |
| `StockReserved` | reservation creata su un `InventoryLevel` | `articleId, locationCode, quantity, reservationId, orderId, occurredAt` |
| `ArticlePriceChanged` *(pianificato)* | il prezzo cambia (currency invariata) | `articleId, oldPriceCents, newPriceCents, currency, occurredAt` |

I primi tre sono il set MVP della Phase 03; `ArticlePriceChanged` (e in futuro
`StockReservationReleased`) seguono la stessa forma.

---

## 7. Porta di persistenza — `ArticleRepository`

Una **sola** porta, solo operazioni aggregate-level. Nessun `InventoryLevelRepository`: spezzerebbe
il confine dell'aggregato. (Regola #5)

```text
interfaces.ArticleRepository
  Save(ctx, *Article) error        // idempotente per lo stesso stato
  FindByID(ctx, id) (*Article, error)
  FindBySKU(ctx, sku) (*Article, error)
  List(ctx, ...) ([]*Article, error)
  Delete(ctx, id) error
```

Solo l'**interfaccia**: nessuna implementazione nel domain layer. (Regola #6)

---

## 8. Tracciabilità: regola del brief → invariante → tipo

| Regola (screenshot) | Si concretizza in | Dove vive |
|---------------------|-------------------|-----------|
| #1 Article guards invariants | `price>0`, `name≠""`, currency immutabile | `entities/article.go` (factory + `ChangePrice`) |
| #2 Stock stays sane | `reserved ≤ quantity`, `quantity ≥ 0`, accesso solo via root | `entities/inventory.go` + metodi di `Article` |
| #3 VO have no identity | factory che rifiutano input; uguaglianza by value | `entities/sku.go`, `entities/money.go` |
| #4 Record events, don't publish | `pendingEvents` + `PullEvents()` | `entities/article.go`, `events/events.go` |
| #5 One aggregate, one repository | porta unica `ArticleRepository`, `Save` idempotente | `interfaces/repository.go` |
| #6 Domain layer only | nessun import di DB/web; `main.go` solo `/health` | tutti i package `entities/`, `events/`, `interfaces/` |

---

## 9. Origine legacy (per la futura migrazione, non per ora)

Dal monolite PHP (`business_data` con `record_type='articolo'`):

| Colonna legacy | Mappa a |
|----------------|---------|
| `code` | `Article.SKU.Code` |
| `name` | `Article.Name` |
| `description` | `Article.Description` |
| `amount_1` (prezzo listino) | `Article.Price.AmountCents` (× 100) |
| `categoria`, `iva`, `unita`, `peso` | **non migrati** (Catalog/Pricing BC) |

Lo schema legacy non distingue reserved vs available: `InventoryLevel.Reserved` è un concetto nuovo
introdotto dal BC Go. (Vedi `tactical-ddd-warehouse.md` §Mapping per il dettaglio completo.)
