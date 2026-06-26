# CP1 — Comprensione del monolite MIC

## Obiettivo
Capire il monolite PHP **MIC** (gestionale di fatturazione legacy) per estrarne il BC
**Warehouse** verso un microservizio Go, con pattern **Strangler Fig**.

## Cosa abbiamo prodotto (5 artefatti, in `docs/`)
- **Page map** — 13 pagine SPA in 6 gruppi → controller → API.
- **Architecture** — architettura runtime + flusso di una richiesta di ordine nel codice.
- **Repo guide** — cosa fa ogni cartella/file.
- **Coupling map** — come sono salvati i dati e i legami cross-dominio.
- **Worst-of list** — i 3 anti-pattern peggiori.

## Com'è fatto MIC
SPA JS → nginx → PHP-FPM → `index.php` → Router → 18 Controller → 1 Repository → MySQL.

## La scoperta chiave
**Niente tabelle `clienti`/`ordini`/`fatture`.** Tutto in **2 tabelle generiche**
(`business_data` con colonne anonime + `business_relations` senza foreign key). Il significato
dei campi vive solo nei controller.

## I 3 problemi peggiori
1. **Schema EAV generico** → il DB non conosce il dominio, semantica duplicata, nessun vincolo.
2. **Zero integrità referenziale** → dati orfani, query N+1, confini invisibili.
3. **Logica di business nel posto sbagliato** → es. IVA fattura calcolata nel browser (split 82/18).

## Da dove partire
Estrarre il **Warehouse** (`magazzino` + `movimento`). Unico legame cross-dominio, ma critico:
**Warehouse → Catalogo sull'articolo** (giacenze e movimenti dipendono dall'`id` articolo via JOIN).
→ Serve un **riferimento prodotto** proprio + **anti-corruption layer**. È l'input per il CP2.
