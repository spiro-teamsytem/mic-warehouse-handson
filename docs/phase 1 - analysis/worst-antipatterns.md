# Worst-of List — Top 3 anti-pattern di MIC (`phase-01-monolith`)

Le tre scelte architetturali che fanno più male, in ordine di gravità. Per ciascuna: **cos'è**,
**perché fa male**, **quanto costa continuare a conviverci**. Tutte con evidenza nel codice.

---

## #1 — Schema EAV generico: la semantica vive solo nei controller

**Cos'è.** Tutti i ~16 tipi di entità (cliente, articolo, ordine, fattura, magazzino, ...) sono
salvati in un'unica tabella `business_data` con colonne anonime `amount_1..4`, `text_1..5`,
`date_1..3`, `payload_json` ([schema.sql:29-72](../phase-01-monolith/database/schema.sql#L29-L72)). Cosa significhi `amount_1`
dipende dal `record_type` e quella conoscenza è scritta solo nei `toDto`/`fromPayload` di ogni
controller — per l'ordine `amount_1`=imponibile ([OrderController.php:45](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L45)), per l'articolo `amount_1`=prezzo
([ArticleController.php:45](../phase-01-monolith/php-app/src/Controllers/ArticleController.php#L45)).

**Perché fa male.**
- Il database non sa nulla del dominio: nessun tipo, nessun `NOT NULL` significativo, nessun vincolo. Un bug in un controller scrive silenziosamente dati incoerenti.
- La semantica è duplicata e non condivisa: leggere i dati fuori dall'app (report, BI, un altro servizio) richiede di reimplementare la mappa "colonna → significato".
- Impossibile evolvere lo schema in sicurezza: aggiungere un campo significa riusare una colonna generica o infilarlo in `payload_json`, perdendo query/indici.

**Costo del non-fare.** Ogni nuova feature paga una "tassa di traduzione" e rischia regressioni
silenziose sui dati. Onboarding lento (nessuno schema leggibile). Più si accumulano record, più
diventa costoso e rischioso introdurre tabelle reali: è il debito che cresce a interesse composto.

---

## #2 — Nessuna integrità referenziale: relazioni e gerarchie "soft"

**Cos'è.** Tutte le relazioni N-N/1-N stanno in `business_relations` (`source_id → target_id` +
`relation_type`) **senza foreign key** ([schema.sql:92-104](../phase-01-monolith/database/schema.sql#L92-L104)); le composizioni padre-figlio
usano i campi polimorfici `parent_id`/`parent_type` (es. `riga_ordine`→`ordine`,
[OrderController.php:157-158](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L157-L158)). I legami cross-dominio sono solo id non vincolati: una
riga d'ordine "punta" a un articolo via `articolo_in_riga_ordine` ([OrderController.php:166](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L166)), una
fattura al cliente via `cliente_di_fattura`, e così via (vedi [coupling-map.md](coupling-map.md)).

**Perché fa male.**
- L'integrità esiste solo se il codice si ricorda di mantenerla. Cancellare un articolo non impedisce di lasciare movimenti/righe orfani; `delete()` ripulisce le relazioni ma non i riferimenti logici altrove ([Repository.php:165-172](../phase-01-monolith/php-app/src/Repository.php#L165-L172)).
- Ogni lettura "ricostruisce" le entità con query separate sulle relazioni → **N+1**: `OrderController::toDto` fa 3 query per riga di lista ([OrderController.php:15-19](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L15-L19)), `righe()` 1 query per articolo per riga ([OrderController.php:95-100](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L95-L100)).
- I confini di dominio sono invisibili: qualsiasi controller può creare una relazione verso qualsiasi record, quindi gli accoppiamenti proliferano senza che nessuno se ne accorga.

**Costo del non-fare.** Dati che si corrompono lentamente (orfani, dangling reference) difficili da
diagnosticare; performance che degradano con i volumi; ogni tentativo di estrarre un pezzo (es. il
Warehouse) deve prima ricostruire a mano quali legami esistono davvero, perché lo schema non li
dichiara.

---

## #3 — Logica di business sparsa e nel posto sbagliato (anche nel frontend)

**Cos'è.** Le regole di dominio non hanno una casa autorevole. Esempi concreti:
- Il calcolo fiscale di una fattura è fatto **nel browser** con uno split fisso 82/18 imponibile/IVA ([orders.js:139-153](../phase-01-monolith/php-app/public/js/orders.js#L139-L153)) — un client può inviare totali arbitrari.
- L'aliquota IVA è risolta per **stringa-codice** (`articolo.text_2 = 'IVA22'`) con lookup e fallback a 22% hard-coded ([OrderController.php:137-142](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L137-L142)); nessun vincolo che il codice esista.
- Il calcolo di riga e il ricalcolo totali ordine vivono nel controller ([OrderController.php:135-144](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L135-L144), [178-199](../phase-01-monolith/php-app/src/Controllers/OrderController.php#L178-L199)); la giacenza è una `SUM` opportunistica in un altro controller ([MagazzinoController.php:49-62](../phase-01-monolith/php-app/src/Controllers/MagazzinoController.php#L49-L62)).
- La categoria dell'articolo è **duplicata**: relazione `categoria_di_articolo` + stringa su `text_1` ([ArticleController.php:29](../phase-01-monolith/php-app/src/Controllers/ArticleController.php#L29)), due fonti di verità che possono divergere.

**Perché fa male.**
- Nessun invariante è garantito in un solo punto: la stessa regola (totali, IVA) può dare risultati diversi a seconda di chi la calcola (client vs server) e rischia di non essere applicata affatto.
- La logica nel frontend non è riusabile né testabile dal backend; chiunque chiami l'API la bypassa.
- Le fonti di verità duplicate (categoria, stock spezzato tra Articolo e Magazzino — vedi [coupling-map.md](coupling-map.md)) rendono impossibile fidarsi di un singolo valore.

**Costo del non-fare.** Rischio di **dati finanziari errati** (fatture con importi incoerenti) e di
regole applicate a macchia di leopardo; ogni nuovo canale (API pubblica, import, microservizio)
deve reimplementare o aggirare regole che dovrebbero stare nel dominio. È esattamente la logica che
l'estrazione DDD deve consolidare in aggregati con invarianti — finché resta sparsa, ogni estrazione
parte da una base inaffidabile.

---

## In una riga

1. **EAV generico** → il DB non conosce il dominio; la semantica è duplicata nei controller.
2. **Zero integrità referenziale** → relazioni soft, dati che marciscono, N+1 ovunque, confini invisibili.
3. **Logica di dominio dispersa** (anche nel browser) → nessun invariante garantito, fonti di verità duplicate, rischio su dati fiscali.

> Insieme spiegano perché MIC è difficile da cambiare in sicurezza e perché conviene estrarre i
> Bounded Context con Strangler Fig invece di rifattorizzare in-place.
