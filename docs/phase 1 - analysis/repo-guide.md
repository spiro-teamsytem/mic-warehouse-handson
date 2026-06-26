# Repo Guide — MIC Warehouse Hands-on

Guida di orientamento al repository: cosa contiene ogni cartella e file, e dove guardare
per fare una determinata cosa. Pensata per chi entra oggi nel progetto.

## Cos'è questo repo

Laboratorio della **Lezione 7**: estrarre il Bounded Context **Warehouse (Magazzino)** dal
monolite PHP legacy **MIC** (un gestionale di fatturazione fittizio, ispirato a FattureInCloud)
verso un microservizio **Go pulito**, applicando il pattern **Strangler Fig** con un AI coding
agent come motore.

Il repo è organizzato in **3 fasi/checkpoint** + documentazione di riferimento. Il branch che si
clona (`lezione-7`) è il **punto di partenza senza soluzioni**; le soluzioni vivono sul branch
`lezione-7-soluzione`.

```
mic-warehouse-handson/
├── README.md / README-IT.md      ← overview lab, branch, 3 checkpoint
├── docs/
│   ├── adr/                       ← Architecture Decision Records (perché si fa così)
│   ├── page-map.md                ← (generato) mappa delle pagine SPA
│   ├── architecture.md            ← (generato) architettura + flusso ordine
│   └── repo-guide.md              ← questo file
├── phase-01-monolith/    CP1 — Capire: il monolite MIC da esplorare
├── phase-02-analysis/    CP2 — Decidere: analisi DDD, sceglie il BC Warehouse
└── phase-03-skeleton/    CP3 — Costruire: domain layer Go del Warehouse
```

---

## Livello radice

| File/Cartella | Cosa è |
|---|---|
| `README.md` / `README-IT.md` | Punto d'ingresso: spiega lab, i due branch (start vs soluzione) e i 3 checkpoint. Inglese + italiano. |
| `docs/` | Documentazione trasversale: ADR di riferimento e artefatti generati. |
| `phase-01-monolith/` | Il monolite PHP funzionante da studiare (CP1). |
| `phase-02-analysis/` | Analisi DDD strategica/tattica che nomina il BC da estrarre (CP2). |
| `phase-03-skeleton/` | Punto di partenza per costruire il domain layer Go del Warehouse (CP3). |
| `.vscode/` | Impostazioni editor locali (non versionate nel cuore del lab). |

---

## `docs/` — riferimenti

| File | Cosa è |
|---|---|
| `adr/ADR-001-strangler-fig-pattern*.md` | Strategia di migrazione Strangler Fig: estrarre BC un pezzo alla volta. |
| `adr/ADR-006-php-monolite-implementation*.md` | Perché il baseline MIC è un monolite PHP con quel design "brutto di proposito". |
| `adr/ADR-007-go-clean-architecture-implementation*.md` | Clean architecture in Go per il nuovo servizio. |
| `adr/ADR-010-event-storming-methodology*.md` | Metodo di Event Storming usato in phase-02. |
| `adr/ADR-011-ddd-building-blocks*.md` | Building block DDD (aggregate, value object, eventi, repository port). |

> Ogni ADR ha la coppia EN + `-IT`. Si leggono per capire **il perché** delle scelte, non il come.

---

## `phase-01-monolith/` — il monolite MIC (CP1)

Applicazione completa e funzionante: SPA vanilla JS + API PHP + MySQL, tutto in container.
Il design è **volutamente anti-pattern** (vedi sotto): è la palestra per l'estrazione.

### Infrastruttura / esecuzione

| File | Cosa fa |
|---|---|
| `docker-compose.yml` | Orchestra 3 servizi: `mic-app` (porta 8088), `mysql` (3306), `adminer` (8082). Monta schema+seed all'avvio. |
| `Dockerfile` | Immagine unica PHP 8.2-FPM + nginx + supervisord (entrambi nello stesso container). |
| `nginx.conf` | Serve asset statici da `public/`; instrada tutto il resto a `index.php` via FastCGI. |
| `openapi.yaml` | Contratto API legacy delle rotte MIC. |
| `README.md` / `README-IT.md` | Missione di CP1, come avviare, i 5 deliverable da produrre. |

### Database

| File | Cosa fa |
|---|---|
| `database/schema.sql` | Crea **solo 2 tabelle generiche**: `business_data` e `business_relations`. |
| `database/seed.sql` | Dati realistici deterministici (~2465 record, ~4185 relazioni). |

**Punto chiave:** non esistono tabelle `clienti`, `ordini`, `fatture`. Tutto vive in:
- `business_data` — colonne anonime `amount_1..4`, `text_1..5`, `date_1..3`, `status`, `payload_json`; il significato dipende da `record_type` (`ordine`, `articolo`, `fattura`...).
- `business_relations` — collegamenti tipizzati `source_id → target_id` (`cliente_di_ordine`, `articolo_in_riga_ordine`...).

### Backend PHP — `php-app/`

| File | Cosa fa |
|---|---|
| `index.php` | Front controller: autoloader PSR, serve SPA/asset, registra le rotte API e fa il dispatch. |
| `src/Router.php` | Mini router con pattern regex (`/api/foo/:id`). Ritorna `[status, body]`. |
| `src/Database.php` | Wrapper PDO singleton con retry sul cold start di MySQL. |
| `src/Repository.php` | **Cuore generico**: CRUD su `business_data` + gestione `business_relations`. Tutto passa di qui. |
| `src/Models/BusinessData.php` | Envelope tipato di una riga generica (`amount1`, `text1`...) + `fromRow`/`toArray`. |
| `src/Models/BusinessRelation.php` | Envelope di una relazione. |
| `src/Controllers/BaseController.php` | Plumbing CRUD condiviso (`index/show/create/update/destroy`) + `fromPayload`/`toDto` astratti. |
| `src/Controllers/*Controller.php` | Un controller per dominio funzionale: traduce le colonne generiche nel significato di quel record_type. |

**I controller (18)** raggruppati per dominio funzionale:

- **Anagrafiche:** `CustomerController`, `SupplierController`, `UserController`
- **Catalogo:** `ArticleController`, `CategoryController`, `IvaController`, `ListinoController`
- **Vendita:** `OrderController`, `ScontoController`, `AgenteController`
- **Logistica/Magazzino:** `MagazzinoController`, `MovimentoController`
- **Fiscale:** `InvoiceController`, `NotaCreditoController`, `PagamentoController`
- **Sistema/Cross:** `DashboardController`, `AuditController`

> Ogni controller "sa" che per il suo `record_type` `amount_1` significa una cosa specifica. Questa
> conoscenza sparsa nei controller è esattamente l'accoppiamento che l'estrazione deve sciogliere.

### Frontend SPA — `php-app/public/`

| File | Cosa fa |
|---|---|
| `index.html` | Shell della SPA: sidebar di navigazione, topbar, contenitori modali/toast. |
| `css/style.css` | Stili dell'app. |
| `js/app.js` | **Core SPA**: router hash-based, wrapper `fetch`, modali/toast, tabella paginata riusabile, form builder. |
| `js/dashboard.js` | View Dashboard (KPI + grafici Chart.js). |
| `js/articles.js`, `customers.js`, `suppliers.js`, `listini.js`, `orders.js`, `invoices.js`, `magazzino.js`, `sconti.js`, `agenti.js` | Una view per area funzionale (lista + form + dettaglio). |
| `js/settings.js` | Registra 3 view: Categorie, Aliquote IVA, Utenti. |

> Mappa pagine ↔ endpoint in [page-map.md](page-map.md); flusso di una richiesta in [architecture.md](architecture.md).

---

## `phase-02-analysis/` — analisi DDD (CP2)

Documenti (nessun codice) che trasformano l'esplorazione di CP1 in confini di aggregato e in un
piano di estrazione. Concludono scegliendo il BC **Warehouse**.

| File | Cosa è |
|---|---|
| `README.md` / `README-IT.md` | Obiettivi e percorso di CP2. |
| `event-storming-canvas.md` | Canvas di Event Storming (eventi di dominio, comandi, attori). |
| `ubiquitous-language.md` | Linguaggio ubiquo: termini di dominio condivisi. |
| `context-mapping.md` | Context mapping: relazioni tra i bounded context. |
| `tactical-ddd-warehouse.md` | DDD tattico applicato al Warehouse (aggregate, VO, eventi). |
| `DEPENDENCY-MAP.md` | Mappa delle dipendenze tra le aree del dominio. |
| `solutions/` | Versioni di riferimento (popolate sul branch soluzione). |

---

## `phase-03-skeleton/` — domain layer Go (CP3)

Qui si scrive il **primo codice del nuovo servizio**: solo il domain layer in Go (aggregate, value
object, eventi, repository port) — **niente DB, niente HTTP, niente framework**.

| File/Cartella | Cosa è |
|---|---|
| `README.md` / `README-IT.md` | Missione di CP3: cosa costruire e gli invarianti da garantire. |
| `php-app/`, `database/`, `docker-compose.yml`, `Dockerfile`, `nginx.conf`, `openapi.yaml` | Copia del monolite MIC: resta in piedi come sistema legacy da cui si estrae (contesto Strangler Fig). |

> Sul branch `lezione-7` non c'è scheletro Go da leggere: la struttura del codice Go la decidi tu.
> La soluzione di riferimento vive sul branch `lezione-7-soluzione`.

---

## Dove guardo se voglio…

| Voglio… | Vado in… |
|---|---|
| Avviare l'app | `phase-01-monolith/` → `docker compose up --build` (GUI su :8088, Adminer su :8082) |
| Capire come è fatta una richiesta API | `index.php` → `Router.php` → `*Controller.php` → `Repository.php` |
| Capire come sono salvati i dati | `database/schema.sql` (solo `business_data` + `business_relations`) |
| Vedere le schermate | `php-app/public/index.html` + `js/app.js` e le view per dominio |
| Capire le scelte architetturali | `docs/adr/` |
| Vedere l'analisi DDD e il BC scelto | `phase-02-analysis/` |
| Costruire il nuovo servizio | `phase-03-skeleton/` |
