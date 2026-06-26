# Page Map — MIC Warehouse (soluzione `phase-01-monolith`)

Mappa delle pagine della SPA (`php-app/public`) con le rotte hash-based, i controller PHP
e gli endpoint API REST che ciascuna pagina consuma.

- **Shell SPA**: [index.html](../phase-01-monolith/php-app/public/index.html) + [app.js](../phase-01-monolith/php-app/public/js/app.js) (router hash-based, fetch wrapper, modali/toast, tabella riusabile).
- **Front controller / API**: [index.php](../phase-01-monolith/php-app/index.php) → [Router.php](../phase-01-monolith/php-app/src/Router.php) → Controllers.
- **Pattern API**: ogni risorsa espone un set CRUD standard:
  `GET /base` · `GET /base/:id` · `POST /base` · `PUT /base/:id` · `DELETE /base/:id`.

```
MIC SPA
│
├─ 📊 Dashboard            #/dashboard      → DashboardController
│
├─ ANAGRAFICHE
│   ├─ 📦 Articoli         #/articles       → ArticleController
│   ├─ 👥 Clienti          #/customers      → CustomerController
│   └─ 🏭 Fornitori        #/suppliers      → SupplierController
│
├─ CATALOGO / VENDITA
│   ├─ 📋 Listini          #/listini        → ListinoController
│   ├─ 🛒 Ordini           #/orders         → OrderController
│   ├─ 💰 Sconti           #/sconti         → ScontoController
│   └─ 👨‍💼 Agenti          #/agenti         → AgenteController
│
├─ LOGISTICA
│   └─ 📦 Magazzino        #/magazzino      → MagazzinoController + MovimentoController
│
├─ FISCALE
│   └─ 🧾 Fatture          #/invoices       → InvoiceController (+ PagamentoController, NotaCreditoController)
│
└─ IMPOSTAZIONI
    ├─ 🏷️ Categorie        #/categories     → CategoryController
    ├─ 📑 Aliquote IVA     #/iva            → IvaController
    └─ ⚙️ Utenti           #/users          → UserController
```

## Dettaglio pagine → endpoint

| Pagina (route) | View JS | Controller | Endpoint principali | Dipendenze da altre risorse |
|---|---|---|---|---|
| **Dashboard** `#/dashboard` | [dashboard.js](../phase-01-monolith/php-app/public/js/dashboard.js) | `DashboardController` | `GET /api/dashboard/kpi` | — |
| **Articoli** `#/articles` | [articles.js](../phase-01-monolith/php-app/public/js/articles.js) | `ArticleController` | CRUD `/api/articles` | `GET /api/categories`, `GET /api/iva-rates` (select) |
| **Clienti** `#/customers` | [customers.js](../phase-01-monolith/php-app/public/js/customers.js) | `CustomerController` | CRUD `/api/customers` · `GET /api/customers/:id/orders` · `GET /api/customers/:id/invoices` | — |
| **Fornitori** `#/suppliers` | [suppliers.js](../phase-01-monolith/php-app/public/js/suppliers.js) | `SupplierController` | CRUD `/api/suppliers` | — |
| **Listini** `#/listini` | [listini.js](../phase-01-monolith/php-app/public/js/listini.js) | `ListinoController` | CRUD `/api/listini` · `GET\|POST /api/listini/:id/voci` | `GET /api/articles` (voci) |
| **Ordini** `#/orders` | [orders.js](../phase-01-monolith/php-app/public/js/orders.js) | `OrderController` | CRUD `/api/orders` · `GET\|POST /api/orders/:id/righe` | `customers`, `listini`, `agenti`, `articles`, `invoices` |
| **Sconti** `#/sconti` | [sconti.js](../phase-01-monolith/php-app/public/js/sconti.js) | `ScontoController` | CRUD `/api/sconti` | — |
| **Agenti** `#/agenti` | [agenti.js](../phase-01-monolith/php-app/public/js/agenti.js) | `AgenteController` | CRUD `/api/agenti` | — |
| **Magazzino** `#/magazzino` | [magazzino.js](../phase-01-monolith/php-app/public/js/magazzino.js) | `MagazzinoController` · `MovimentoController` | CRUD `/api/magazzini` · `GET /api/magazzini/:id/giacenze` · CRUD `/api/movimenti` | `GET /api/articles` |
| **Fatture** `#/invoices` | [invoices.js](../phase-01-monolith/php-app/public/js/invoices.js) | `InvoiceController` | CRUD `/api/invoices` · `POST /api/invoices/:id/invia-sdi` · `GET\|POST /api/pagamenti` | `GET /api/customers` |
| **Categorie** `#/categories` | [settings.js](../phase-01-monolith/php-app/public/js/settings.js) | `CategoryController` | CRUD `/api/categories` | — |
| **Aliquote IVA** `#/iva` | [settings.js](../phase-01-monolith/php-app/public/js/settings.js) | `IvaController` | CRUD `/api/iva-rates` | — |
| **Utenti** `#/users` | [settings.js](../phase-01-monolith/php-app/public/js/settings.js) | `UserController` | CRUD `/api/users` | — |

## Endpoint senza pagina dedicata

Esposti dall'API ma usati come supporto / non collegati al menu di navigazione:

- `NotaCreditoController` → `/api/note-credito` (CRUD) — collegato al dominio Fatture.
- `AuditController` → `/api/audit-log` (CRUD) — log di sistema.

## Note sul routing

- Router SPA: hash-based in [app.js](../phase-01-monolith/php-app/public/js/app.js#L136) (`#/route/...`); default `#/dashboard`.
- Le view si registrano con `MIC.registerView(name, title, renderer)` e vengono iniettate in `#view`.
- La search globale in topbar instrada verso `articles/customers/suppliers/orders/invoices/listini` in base alle keyword.
