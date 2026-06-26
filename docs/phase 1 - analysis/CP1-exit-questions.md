# CP1 — Domande di uscita (autovalutazione)

Risposte alle 3 domande di autovalutazione del [phase-01 README](../phase-01-monolith/README-IT.md),
con evidenze dagli artefatti prodotti in [docs/](.).

---

## 1. Se volessimo ritagliare per primo un pezzo di MIC, quale dipendenza tra due parti del dominio dovremmo gestire, e perché?

Il pezzo da ritagliare per primo è il **Warehouse** (`magazzino` + `movimento` + calcolo giacenze).
La dipendenza da gestire è:

> **Warehouse → Catalogo, sull'identità dell'articolo.**

**Perché.** È l'unico legame cross-dominio del Warehouse, ed è strutturale:
- Ogni movimento punta a un articolo via la relazione `movimento_di_articolo` ([MovimentoController.php](../phase-01-monolith/php-app/src/Controllers/MovimentoController.php)).
- La giacenza si calcola con un JOIN `magazzino → movimento → articolo` che legge `articolo.code`/`name` ([MagazzinoController.php:49-62](../phase-01-monolith/php-app/src/Controllers/MagazzinoController.php#L49-L62)).
- Il dato di stock è **spezzato tra due domini**: la soglia di riordino vive sull'Articolo (`amount_2`), la giacenza reale nel Warehouse.

**Cosa faremmo.** Il nuovo servizio non potrà fare JOIN sulla tabella articoli: serve un
**riferimento prodotto** proprio (copia di id/SKU) e un **anti-corruption layer** verso il Catalogo.
Questo confine va deciso prima ancora di toccare la persistenza.
*(Evidenza: [coupling-map.md](coupling-map.md), accoppiamenti C5/C6.)*

---

## 2. Quale singola scelta di design rende MIC più difficile da cambiare in sicurezza, e cosa faresti?

> **Lo schema EAV generico: tutte le entità in `business_data` + `business_relations`, senza foreign key.**

**Perché è la peggiore.** Il database non conosce il dominio — colonne anonime (`amount_1..4`,
`text_1..5`), nessun vincolo, semantica codificata *solo* nei controller
([schema.sql:29-72](../phase-01-monolith/database/schema.sql#L29-L72)). A cascata: integrità solo
applicativa (dati orfani possibili), query **N+1** per ricostruire le entità
([OrderController.php:15-19](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L15-L19)),
confini di dominio invisibili. Qualsiasi modifica rischia regressioni silenziose sui dati.

**Cosa faremmo.** Non rifattorizzare in-place, ma **estrarre i Bounded Context con Strangler Fig**:
ogni BC estratto ottiene tabelle reali, tipizzate e vincolate, con la logica consolidata in
aggregati che garantiscono gli invarianti. Si comincia dal Warehouse.
*(Evidenza: [worst-antipatterns.md](worst-antipatterns.md) #1 e #2.)*

---

## 3. Elenca i domini funzionali di MIC: dove vive ciascuno in schermate, codice e dati?

| Dominio funzionale | Schermate (SPA) | Codice (controller) | Dati (`record_type`) |
|---|---|---|---|
| **Anagrafiche** | Clienti, Fornitori, Utenti | Customer/Supplier/User/Agente | `cliente`, `fornitore`, `utente`, `agente` |
| **Catalogo** | Articoli, Categorie, Aliquote IVA, Listini | Article/Category/Iva/Listino | `articolo`, `categoria`, `aliquota_iva`, `listino`, `voce_listino` |
| **Vendita / Ordini** | Ordini, Sconti, Agenti | Order/Sconto/Agente | `ordine`, `riga_ordine`, `sconto` |
| **Magazzino (Warehouse)** ⭐ | Magazzino | Magazzino/Movimento | `magazzino`, `movimento` |
| **Fiscale** | Fatture | Invoice/NotaCredito/Pagamento | `fattura`, `nota_credito`, `pagamento` |
| **Sistema** | Dashboard | Dashboard/Audit | `audit_log` + KPI |

**Punto chiave.** I domini sono ben distinti nelle **schermate** (sidebar) e nel **codice** (un
controller per area), ma nei **dati collassano tutti** nelle stesse 2 tabelle generiche. È questa
discrepanza — confini netti in alto, indistinti in basso — a rendere l'estrazione necessaria e non
banale. ⭐ = target di estrazione, confermato dai commenti nel codice
([MagazzinoController.php:6-15](../phase-01-monolith/php-app/src/Controllers/MagazzinoController.php#L6-L15)).
