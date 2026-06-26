# Phase 2 — Plan BC (Article)

> Obiettivo: **pianificare** (non ancora implementare) il domain layer del Bounded Context
> **Warehouse**, partendo dall'estrazione di un **modello degli articoli** e delle **regole da
> persistere**. L'implementazione vera e propria avviene nella Phase 03 (`phase-03-skeleton/`).

Questa cartella è il ponte fra l'analisi (CP2, `phase-02-analysis/`) e la build (CP3,
`phase-03-skeleton/`). Traduce la Tactical DDD già fatta in un **piano d'azione concreto**: quali
tipi, quali file, in quale ordine, con quali criteri di accettazione.

## Le 6 regole che devono reggere

Sono il brief di questa fase (vedi screenshot allegato). Ognuna è un'invariante o un confine
architetturale che il codice della Phase 03 deve garantire:

| # | Regola | Cosa impone |
|---|--------|-------------|
| 1 | **Article guards its invariants** | `price > 0`, `name` non vuoto, `currency` immutabile dopo la creazione |
| 2 | **Stock stays sane** | `reserved ≤ quantity`, `quantity ≥ 0`; lo stock si tocca **solo** attraverso `Article` |
| 3 | **Value objects have no identity** | `SKU` e `Money` confrontano per valore, costruiti da una factory che rifiuta input invalido |
| 4 | **Record events, do not publish** | l'aggregato **registra** eventi su una lista pending; un layer successivo li drena. Niente messaging qui |
| 5 | **One aggregate, one repository** | una sola porta per l'aggregato `Article`. Nessun `InventoryLevelRepository`. `Save` idempotente |
| 6 | **Domain layer only** | niente database, niente HTTP. Solo i tipi di business e le loro regole |

## I file di questo piano

| File | Contenuto |
|------|-----------|
| [`domain-model.md`](./domain-model.md) | Il modello estratto degli articoli: aggregato, entità, value object, eventi, e la mappatura regola→invariante→tipo |
| [`implementation-plan.md`](./implementation-plan.md) | Il piano di build: layout dei package, file, signature, sequenza dei task (T0–T8), criteri di accettazione |
| [`test-plan.md`](./test-plan.md) | Il piano di scrittura dei test per validare il domain model: suite per tipo, casi minimi, tracciabilità test→regola, sequenza allineata ai task di build |
| [`action-plan.md`](./action-plan.md) | Il piano d'azione concreto in **fasi stagne** (F0–F6): ingresso, deliverable e gate d'uscita binario per ogni fase, con il gate finale (DoD) |

## Fonti (già prodotte, da non rifare)

- [`phase-02-analysis/tactical-ddd-warehouse.md`](../../phase-02-analysis/tactical-ddd-warehouse.md) — i building block tattici già mappati sul dominio
- [`phase-02-analysis/ubiquitous-language.md`](../../phase-02-analysis/ubiquitous-language.md) — il vocabolario di business
- [`phase-03-skeleton/README.md`](../../phase-03-skeleton/README.md) — il target di build e i criteri di accettazione ufficiali
- [`docs/adr/ADR-011-ddd-building-blocks.md`](../adr/ADR-011-ddd-building-blocks.md) — Aggregate / Entity / Value Object / Domain Event
- [`docs/adr/ADR-007-go-clean-architecture-implementation.md`](../adr/ADR-007-go-clean-architecture-implementation.md) — il layering Clean Architecture in Go

## Cosa NON è in questa fase

- Scrivere il codice Go (è la Phase 03).
- Persistenza MySQL, dual-write, use case, HTTP, auth, eventi sul wire (Phase 04+).
- `StockMovement` ledger e `StockReservation` come entità a sé: **descopati** in questa fase
  (la reservation è solo un contatore `Reserved` su `InventoryLevel`).
