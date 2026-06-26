# Action Plan — Article domain layer (fasi stagne)

> Piano d'azione concreto per costruire e validare il domain layer del BC Warehouse, organizzato in
> **fasi stagne**: ogni fase ha un **ingresso**, dei **deliverable** e un **gate d'uscita binario**.
> Non si entra nella fase _n+1_ finché il gate della fase _n_ non è **verde**. Niente lavoro a
> cavallo tra fasi.
>
> Sintetizza i file di questa cartella: il modello da [`domain-model.md`](./domain-model.md), i task
> da [`implementation-plan.md`](./implementation-plan.md), le suite da [`test-plan.md`](./test-plan.md).

---

## Regole del gioco (valgono per ogni fase)

1. **Gate verde = fase chiusa.** Il gate è una condizione binaria verificabile (build ok / test
   verdi / check superato), non un giudizio soggettivo.
2. **Test insieme al codice.** I test di una fase si scrivono _dentro_ la fase, non dopo.
3. **No leak in avanti.** Ogni fase produce solo ciò che le compete; persistenza, HTTP, use case
   restano fuori (Regola #6). Una fase non anticipa lavoro della successiva.
4. **Atomicità delle invarianti.** Ogni caso di errore deve lasciare lo stato invariato e nessun
   evento registrato (vale dalla Fase 2 in poi).

Legenda dipendenze: `F0 → F1 → {F2, F3} → F4 → F5 → F6`. F2 e F3 sono indipendenti tra loro.

---

## Fase 0 — Bootstrap del modulo

| | |
|---|---|
| **Obiettivo** | Modulo Go compilabile e servizio che boota, senza dominio. |
| **Ingresso** | Cartella `phase-03-skeleton/` con il copy PHP da estrarre. |
| **Lavoro** | `go.mod` (`warehouse.local/core`, Go 1.22) · `main.go` con Echo: solo `GET /health` → `{"status":"ok"}` su `:8081`. |
| **Deliverable** | `go.mod`, `main.go`. |
| **Test** | — (smoke manuale). |
| **Gate d'uscita** | `go build ./...` ok **e** `docker compose up --build` → `curl :8081/health` = `{"status":"ok"}`. |
| **Copre** | Regola #6 (boot edge-only). |

---

## Fase 1 — Value Objects (`SKU`, `Money`)

| | |
|---|---|
| **Obiettivo** | I due VO con factory che falliscono loud e uguaglianza by value. |
| **Ingresso** | Gate F0 verde. |
| **Lavoro** | `entities/sku.go` (`NewSKU`, regex `^[A-Z0-9-]{3,32}$`) · `entities/money.go` (`NewMoney`, `cents≥0`, currency 3 maiuscole ISO). |
| **Deliverable** | `sku.go`, `money.go` + `sku_test.go`, `money_test.go`. |
| **Test** | Tabelle di [`test-plan.md` §3.1–3.2](./test-plan.md): validi, boundary (len 3/32, `cents=0`), invalidi, uguaglianza by value. |
| **Gate d'uscita** | `go test ./entities/...` verde per SKU e Money; ogni VO ha ≥1 happy path e ≥1 fallimento loud. |
| **Copre** | Regola #3. |

---

## Fase 2 — Entità `InventoryLevel`

| | |
|---|---|
| **Obiettivo** | Entità di stock con le sue invarianti, **non** mutabile dall'esterno dell'aggregato. |
| **Ingresso** | Gate F1 verde. |
| **Lavoro** | `entities/inventory.go`: `quantity≥0`, `reserved≥0`, `reserved≤quantity`, id/location non vuoti. Nessuna API pubblica di mutazione indipendente. |
| **Deliverable** | `inventory.go` + `inventory_test.go`. |
| **Test** | [`test-plan.md` §3.3](./test-plan.md): costruzione valida, negativi, `reserved>quantity`, boundary `reserved==quantity`. |
| **Gate d'uscita** | `go test ./entities/...` verde per InventoryLevel; boundary coperto; nessun setter pubblico che violi le invarianti. |
| **Copre** | Regola #2 (parte struttura). |

---

## Fase 3 — Domain Events

| | |
|---|---|
| **Obiettivo** | Interfaccia comune + i 3 eventi core, immutabili e past-tense. |
| **Ingresso** | Gate F1 verde (indipendente da F2). |
| **Lavoro** | `events/events.go`: `DomainEvent` (`EventName()`, `OccurredAt()`) · `ArticleCreated`, `InventoryAdjusted`, `StockReserved` con i payload di [`domain-model.md` §6](./domain-model.md). |
| **Deliverable** | `events.go` + `events_test.go`. |
| **Test** | [`test-plan.md` §3.4](./test-plan.md): ogni evento implementa l'interfaccia, naming past-tense, payload presenti, nessun setter. |
| **Gate d'uscita** | `go test ./events/...` verde; i 3 eventi soddisfano `DomainEvent` (check a compile-time). |
| **Copre** | Regola #4 (parte tipi). |

---

## Fase 4 — Aggregate Root `Article`

| | |
|---|---|
| **Obiettivo** | L'unico entry point: factory, behaviors, registrazione eventi. È il cuore del BC. |
| **Ingresso** | Gate F2 **e** F3 verdi. |
| **Lavoro** | `entities/article.go`: `NewArticle` (id/name non vuoti, `price>0`, registra `ArticleCreated`) · `ChangePrice` (currency immutabile, `>0`) · `AdjustInventory` · `ReserveStock` (`reserved+qty≤quantity`) · `PullEvents` (drena). Stock mutabile **solo** da qui. |
| **Deliverable** | `article.go` + `article_test.go`. |
| **Test** | [`test-plan.md` §3.5–3.6](./test-plan.md): factory (name vuoto, prezzo 0) · `ChangePrice` currency diversa → errore · stock oltre soglia → errore · `PullEvents` drena (2ª chiamata vuota) · **atomicità** su ogni errore. |
| **Gate d'uscita** | `go test ./entities/...` interamente verde; ogni errore lascia stato invariato e zero eventi; nessuna pubblicazione (solo append in memoria). |
| **Copre** | Regole #1, #2 (comportamento), #4 (recording). |

---

## Fase 5 — Porta `ArticleRepository`

| | |
|---|---|
| **Obiettivo** | Una sola porta aggregate-level, solo interfaccia. |
| **Ingresso** | Gate F4 verde. |
| **Lavoro** | `interfaces/repository.go`: `Save` (idempotente), `FindByID`, `FindBySKU`, `List`, `Delete`. **Nessun** `InventoryRepository`. |
| **Deliverable** | `repository.go` + `repository_test.go` (fake in-memory di supporto). |
| **Test** | [`test-plan.md` §3.7](./test-plan.md): compile-time check (`var _ ArticleRepository = ...`), `Save` due volte = stesso stato. |
| **Gate d'uscita** | `go test ./interfaces/...` verde; il fake soddisfa l'interfaccia; non esiste alcuna porta per `InventoryLevel`. |
| **Copre** | Regola #5. |

---

## Fase 6 — Hardening, Clean Architecture & design

| | |
|---|---|
| **Obiettivo** | Chiudere i confini, dimostrare l'isolamento del dominio, rendere esplicite le decisioni. |
| **Ingresso** | Gate F5 verde. |
| **Lavoro** | `architecture_test.go` (i package dominio non importano DB/web) · `design-decisions.md` (mezza pagina: 4 scelte di [`implementation-plan.md` §6](./implementation-plan.md)) · run completa. |
| **Deliverable** | `architecture_test.go`, `design-decisions.md`. |
| **Test** | [`test-plan.md` §3.8](./test-plan.md) + esecuzione di tutta la suite. |
| **Gate d'uscita** | **DoD completa** (sotto) verde. |
| **Copre** | Regola #6 (isolamento) + deliverable di design. |

---

## Gate finale (Definition of Done complessiva)

Replica la DoD di [`implementation-plan.md` §5](./implementation-plan.md). Tutte le caselle verdi
chiudono il piano:

- [ ] **#1** `NewArticle` rifiuta name vuoto e prezzo ≤ 0; `ChangePrice` preserva la currency. *(F4)*
- [ ] **#2** Stock con `reserved≤quantity` e `quantity≥0`, mutato solo via `Article`. *(F2+F4)*
- [ ] **#3** `SKU`/`Money` senza identità, factory loud, confronto by value. *(F1)*
- [ ] **#4** Eventi registrati su pending + `PullEvents` drena; nessuna pubblicazione. *(F3+F4)*
- [ ] **#5** Solo `ArticleRepository` aggregate-level, `Save` idempotente; nessun `InventoryRepository`. *(F5)*
- [ ] **#6** Dominio senza import DB/web; `main.go` solo `/health`. *(F0+F6)*
- [ ] Servizio boota: `docker compose up --build` → `/health` = `{"status":"ok"}`.
- [ ] `go test ./...` (o `docker compose run --rm test`) interamente verde.
- [ ] `design-decisions.md` presente con le 4 scelte motivate.

---

## Quadro sinottico

| Fase | Produce | Gate d'uscita | Regole |
|------|---------|---------------|--------|
| F0 Bootstrap | `go.mod`, `main.go` | build ok + `/health` | #6 |
| F1 Value Objects | `sku.go`, `money.go` | test VO verdi | #3 |
| F2 InventoryLevel | `inventory.go` | test stock verdi | #2 |
| F3 Events | `events.go` | eventi ⊨ `DomainEvent` | #4 |
| F4 Article | `article.go` | invarianti + atomicità verdi | #1, #2, #4 |
| F5 Repository | `repository.go` | compile-time + `Save` idempotente | #5 |
| F6 Hardening | `architecture_test.go`, `design-decisions.md` | DoD completa | #6 |
