# CODEX_PLAN_REVIEW.md

# Review tecnica dettagliata del piano `PLAN_OBSERVABILITY.md`

## 1. Scopo della review

Questo documento analizza in modo critico il piano contenuto in `PLAN_OBSERVABILITY.md` e valuta:

1. la correttezza tecnica delle ipotesi rispetto a Odoo 14 reale;
2. l'efficacia osservativa del modulo proposto;
3. i rischi di regressione funzionale o operativa;
4. il rischio di impatto sulle performance in produzione;
5. le correzioni necessarie per rendere il modulo realmente sicuro da introdurre in un sistema Odoo gia' in esercizio.

La review e' volutamente severa: il requisito aggiuntivo espresso e' che il modulo di osservabilita' non debba in alcun modo compromettere il funzionamento del sistema e non debba introdurre degrado prestazionale percepibile in produzione.

Per questo motivo, ogni rilievo e' accompagnato da una proposta implementativa correttiva, con priorita' operativa e impatto architetturale.

---

## 2. Giudizio sintetico

Il piano e' forte dal punto di vista della copertura funzionale. I suoi obiettivi dichiarati in `PLAN_OBSERVABILITY.md`, righe 43-47, sono validi e realistici:

- dashboard unificata;
- aggiornamento real-time;
- tracciamento storico dei cron;
- alert proattivi;
- azioni operative.

Il piano, pero', nella forma attuale e se implementato letteralmente, presenta quattro problemi strutturali:

1. confonde monitoraggio live con osservabilita' persistente;
2. introduce in alcuni punti logica troppo invasiva su componenti interni di Odoo 14;
3. propone alcune metriche e azioni che, cosi' come descritte, sarebbero inaccurate o potenzialmente pericolose;
4. concentra troppe query e troppe aggregazioni dentro il caricamento live della dashboard, con rischio di caricare proprio il sistema che intende osservare.

Conclusione netta:

- come modulo di monitoraggio operativo, il piano e' buono;
- come modulo di osservabilita' production-safe, il piano deve essere corretto in modo significativo;
- per soddisfare il vincolo "nessun impatto sul sistema", l'architettura va spostata da "dashboard che interroga tutto live" a "raccolta passiva + snapshot + diagnostica live solo on-demand".

---

## 3. Vincoli non negoziabili per una implementazione sicura

Questi vincoli dovrebbero essere esplicitati nel piano come requisiti architetturali di primo livello.

### 3.1 Fail-open obbligatorio

Ogni componente del modulo deve degradare in modo innocuo.

Se il modulo di osservabilita' fallisce:

- non deve interrompere cron esistenti;
- non deve impedire richieste HTTP normali;
- non deve bloccare commit applicativi;
- non deve introdurre rollback su transazioni business;
- non deve saturare il connection pool;
- non deve generare cascata di errori lato frontend.

Tradotto in implementazione:

- tutte le scritture di telemetria devono essere isolate da quelle business;
- le query osservative live devono avere timeout stretti;
- le eccezioni del modulo devono essere catturate e loggate senza propagarsi al chiamante.

### 3.2 Separazione tra dati live e dati campionati

Le metriche costose o aggregate non devono essere calcolate a ogni refresh della dashboard.

Devono esistere due classi di dati:

- `summary data`: metriche sintetiche pre-raccolte e lette da snapshot recenti;
- `live diagnostics`: query attive, lock graph, dettagli pesanti, disponibili solo su richiesta esplicita dell'amministratore.

### 3.3 Nessuna scrittura telemetrica nella transazione business critica

Le scritture del modulo non devono condividere il destino della transazione monitorata, salvo casi strettamente controllati.

Per i cron questo e' essenziale, perche' il core Odoo fa rollback in `odoo/addons/base/models/ir_cron.py`, riga 89.

### 3.4 Nessun polling aggressivo di default

Il default di `refresh_interval = 30` secondi proposto in `PLAN_OBSERVABILITY.md`, sezione 8.3 e sezione 5.5, e' troppo ambizioso per produzione se il payload resta monolitico.

Produzione sicura:

- dashboard summary: 60-120 secondi di default;
- dettaglio live: solo manuale o a richiesta;
- refresh sospeso quando il tab non e' visibile.

### 3.5 Nessuna azione distruttiva abilitata senza guardrail

Le azioni operative proposte in `PLAN_OBSERVABILITY.md`, righe 47 e sezione 7, devono essere considerate funzionalita' ad alto rischio.

In particolare:

- `pg_terminate_backend()`;
- esecuzione manuale di cron;
- eventuali future azioni su query, vacuum o piano di esecuzione.

Queste azioni non devono mai essere esposte come click diretto senza controlli aggiuntivi.

---

## 4. Riepilogo dei rilievi principali

| ID | Severita' | Area | Sintesi |
|----|-----------|------|---------|
| F1 | Critica | Cron instrumentation | L'override proposto di `_callback()` puo' produrre log errati e poggia su una semantica non corretta del core Odoo |
| F2 | Alta | Manual run cron | Il pulsante "Run Cron" usa `method_direct_trigger()` e non percorre il path del scheduler |
| F3 | Alta | Pool metrics | Le metriche su `odoo.sql_db._Pool` sono per-processo, non globali, quindi fuorvianti in prefork |
| F4 | Alta | Dashboard backend | Le aggregazioni proposte hanno N+1, carichi Python inutili e query pesanti ad ogni refresh |
| F5 | Alta | Queue metrics | Le metriche su `mail.mail` rischiano full scan ricorrenti e non implementano esattamente quanto dichiarato nel piano |
| F6 | Alta | Alert engine | Il motore di alert crea duplicati, non modella incidenti e ricalcola troppo |
| F7 | Alta | Frontend refresh | Il frontend puo' generare refresh concorrenti, listener residui e polling eccessivo |
| F8 | Critica | Azioni operative | `kill_query` e trigger manuali sono troppo permissivi e pericolosi se esposti cosi' |
| F9 | Media-Alta | Dati sensibili | Troncatura delle query non equivale a sanificazione dei dati sensibili |
| F10 | Media-Alta | Cron alert XML | Il cron alert punta a un modello ambiguo e la responsabilita' del metodo non e' ben definita |
| F11 | Media | Architettura osservativa | La parte time-series e snapshot e' relegata a sezione avanzata, ma dovrebbe essere core |
| F12 | Media | Correttezza dei sample code | Alcuni snippet del piano non sono production-ready cosi' come scritti |

---

## 5. Rilievi dettagliati e correzioni proposte

## F1. Intercettazione dei cron: l'override proposto non e' sufficientemente sicuro

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, sezione 5.2, righe 351-495.
- Codice reale Odoo: `odoo/addons/base/models/ir_cron.py`, righe 89-120.

### Problema

Il piano descrive correttamente che `_handle_callback_exception()` esegue rollback e che per loggare gli errori serva un cursore separato. Questo e' corretto.

La criticita' reale sta pero' nell'override proposto di `_callback()` nelle righe 403-495 del piano.

L'idea descritta e':

1. chiamare `super()._callback(...)`;
2. se non ci sono eccezioni propagate, considerare l'esecuzione come riuscita;
3. scrivere il log di successo.

Questo e' fragile perche' in Odoo 14 il metodo `_callback()` del core cattura internamente l'eccezione, logga, chiama `_handle_callback_exception(...)` e non la rilancia.

Quindi il chiamante di `super()._callback(...)` non ha garanzia che l'esecuzione sia realmente andata a buon fine solo perche' la chiamata e' terminata senza eccezione.

In altre parole: il pattern del piano puo' classificare come `success` una esecuzione che in realta' e' fallita e gestita internamente dal core.

### Rischi

- falso positivo nei log cron;
- perdita di fiducia nella dashboard;
- alert basati su cron history falsati;
- debugging piu' difficile del caso senza modulo;
- rischio di doppio logging se error path e success path non sono perfettamente coordinati.

### Osservazione ulteriore

Il piano riconosce la difficolta' di misurare la durata in caso di errore e propone un `threading.local()`. L'idea e' sensata, ma resta incompleta se la semantica del successo non viene risolta a monte.

### Correzione proposta

Non usare il pattern "`super()._callback()` + log di successo dopo".

Usare invece una di queste due strategie.

#### Strategia consigliata: override near-verbatim del core con strumentazione minima

Replicare il corpo del metodo `_callback()` del core Odoo, mantenendolo il piu' aderente possibile all'upstream, ed inserire strumentazione solo in due punti:

1. subito prima della `run()` della server action;
2. nel ramo `except`, dopo il rollback gestito dal core.

Pattern raccomandato:

```python
import logging
import time
import traceback
import odoo

from odoo import api, fields, models

_logger = logging.getLogger(__name__)


class IrCron(models.Model):
    _inherit = 'ir.cron'

    @api.model
    def _obs_write_cron_log(self, values):
        try:
            db = odoo.sql_db.db_connect(self._cr.dbname)
            with db.cursor() as cr:
                env = api.Environment(cr, self.env.uid or odoo.SUPERUSER_ID, {})
                env['system_observability.cron.log'].sudo().create(values)
        except Exception:
            _logger.warning('Observability cron log write failed', exc_info=True)

    @api.model
    def _callback(self, cron_name, server_action_id, job_id):
        start_mono = time.monotonic()
        start_dt = fields.Datetime.now()
        execution_uid = self.env.uid

        try:
            if self.pool != self.pool.check_signaling():
                self.env.reset()
                self = self.env()[self._name]

            self.env['ir.actions.server'].browse(server_action_id).run()
            duration = time.monotonic() - start_mono
            end_dt = fields.Datetime.now()

            @self._cr.postcommit.add
            def _after_commit():
                self._obs_write_cron_log({
                    'cron_id': job_id,
                    'cron_name': cron_name,
                    'start_time': start_dt,
                    'end_time': end_dt,
                    'duration': duration,
                    'state': 'success',
                    'user_id': execution_uid,
                })

            self.pool.signal_changes()
        except Exception as exc:
            self.pool.reset_changes()
            _logger.exception(
                'Call from cron %s for server action #%s failed in Job #%s',
                cron_name, server_action_id, job_id,
            )
            self._handle_callback_exception(cron_name, server_action_id, job_id, exc)

            duration = time.monotonic() - start_mono
            end_dt = fields.Datetime.now()
            error_tb = ''.join(traceback.format_exception(type(exc), exc, exc.__traceback__))

            self._obs_write_cron_log({
                'cron_id': job_id,
                'cron_name': cron_name,
                'start_time': start_dt,
                'end_time': end_dt,
                'duration': duration,
                'state': 'error',
                'error_message': error_tb,
                'user_id': execution_uid,
            })
```

Perche' questo pattern e' piu' sicuro:

- il successo e' definito in modo reale, non inferito;
- il log di successo parte solo dopo commit;
- il log di errore usa un cursore separato dopo rollback;
- qualsiasi guasto del logger non si propaga.

#### Strategia alternativa: log runtime + finalizzazione

Se si vuole anche la metrica "cron attualmente in esecuzione" in modo affidabile, e' ancora meglio:

1. creare un record `runtime` all'inizio con `state='running'` e `execution_uuid`;
2. aggiornarlo a `success` o `error` a fine esecuzione;
3. pulire record `running` stantii con un job di garbage collection.

Questo approccio e' piu' preciso del tentativo di inferire i cron live da `pg_stat_activity`.

### Hardening ulteriore richiesto

- aggiungere un campo `trigger_mode` con valori `scheduled`, `manual`, `recovered`;
- aggiungere un campo `worker_pid` per poter correlare l'esecuzione al processo;
- aggiungere `db_name` se si vogliono esportazioni multi-istanza;
- aggiungere test di non regressione che confrontino il comportamento con il core Odoo.

### Raccomandazione finale su F1

Questo punto deve essere considerato bloccante prima di qualsiasi sviluppo frontend.

---

## F2. Il pulsante "Run Cron" non esegue il cron come farebbe lo scheduler

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, righe 47, 990, 1110-1155 circa, tabella test funzionali in sezione 13.
- Codice reale Odoo: `odoo/addons/base/models/ir_cron.py`, righe 81-85 (`method_direct_trigger`).

### Problema

Il piano propone un pulsante frontend che chiama `ir.cron.method_direct_trigger()`.

Nel core Odoo questo metodo:

- non passa da `_callback()`;
- non usa il percorso di lock del scheduler;
- non replica il ciclo `_process_job()`;
- aggiorna `lastcall` ma non riproduce tutte le semantiche del cron runner.

Quindi il pulsante non esegue davvero "il cron" nel senso scheduler-safe, ma esegue la server action associata in modo diretto.

### Conseguenze

- la metrica cron puo' diventare incoerente;
- la durata loggata su `_callback()` non includera' i trigger manuali;
- un amministratore puo' ottenere risultati diversi da quelli del cron reale;
- il test funzionale proposto nel piano puo' dare falsa confidenza.

### Correzione proposta

Tre opzioni in ordine di sicurezza.

#### Opzione A: non esporre il trigger manuale nella MVP

E' la scelta piu' sicura per produzione.

Il modulo resta osservativo, non operativo-distruttivo.

#### Opzione B: esporre un trigger manuale ma dichiararlo per quello che e'

Etichetta UI corretta:

- non "Run Cron";
- ma "Esegui server action ora (manuale, fuori scheduler)".

In questo caso i log devono distinguere `trigger_mode='manual'`.

#### Opzione C: implementare un `action_run_now_safe()` dedicato

Questo metodo dovrebbe:

1. verificare gruppo admin;
2. acquisire lock sul record `ir.cron` con `FOR UPDATE NOWAIT`;
3. eseguire la callback nello stesso schema logico del cron runner;
4. loggare `trigger_mode='manual'`;
5. opzionalmente non alterare `nextcall` se si tratta di esecuzione diagnostica.

### Raccomandazione finale su F2

Per minimizzare il rischio, raccomando di rimuovere questa funzione dalla fase iniziale.

---

## F3. Le metriche del connection pool non rappresentano il sistema intero in modalita' prefork

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, righe 68-72, sezione 3.3, sezione 5.3 `_get_pool_data()`.
- Codice reale Odoo: `odoo/sql_db.py`, righe 561-690 e 754-764.

### Problema

Il piano presenta il pool Odoo come se fosse osservabile come entita' unica del sistema.

Nel codice reale, `_Pool` e' una variabile globale di processo. In modalita' prefork (`workers > 0`), ogni worker ha il proprio `ConnectionPool` separato.

Quindi:

- `used`, `total`, `max`, `leaked` letti da `odoo.sql_db._Pool` descrivono solo il processo corrente;
- non rappresentano l'intera istanza Odoo;
- la dashboard puo' mostrare un valore apparentemente rassicurante mentre un altro worker e' vicino all'esaurimento.

### Problema secondario: metrica `leaked`

Nel codice del pool, durante `borrow()`, le connessioni marcate `leaked` vengono gia' riciclate/ripulite.

Questo significa che il conteggio di `leaked` e':

- istantaneo;
- locale al processo;
- non storico;
- non affidabile come KPI da dashboard o base di alert globale.

### Correzione proposta

#### Correzione minima

Cambiare il significato della sezione UI da:

- "Connection Pool"

a:

- "Connection Pool del processo corrente".

Mostrare esplicitamente:

- `worker_pid`;
- modalita' server (`threaded` / `prefork`);
- avvertenza: "metrica locale al worker corrente".

#### Correzione raccomandata

Usare come KPI globale di salute connessioni solo metriche basate su PostgreSQL:

- totale connessioni sul database da `pg_stat_activity`;
- connessioni attive;
- idle in transaction;
- rapporto con `db_maxconn` configurato.

Formula utile per alert globale:

```text
db_connection_pressure_pct = total_connections_for_db / config_db_maxconn * 100
```

Questa metrica non e' perfetta in assoluto ma e' molto piu' rappresentativa del sistema rispetto al pool locale.

#### Correzione avanzata, solo se davvero necessaria

Se si vuole una vista per-worker, introdurre una tabella snapshot tipo:

- `system_observability.process_sample`

con campi:

- `pid`
- `hostname`
- `collected_at`
- `pool_used`
- `pool_total`
- `pool_max`
- `rss_mb`

e far scrivere snapshot solo da:

- cron dedicato nei processi che lo eseguono;
- oppure punti controllati di amministrazione.

Non raccomando heartbeat frequenti per worker senza una necessita' reale.

### Raccomandazione finale su F3

Non usare le metriche del pool locale come base di alert critico globale.

---

## F4. Il backend della dashboard e' troppo monolitico e contiene query/aggregazioni evitabili

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, righe 503-720.

### Problemi principali

#### Problema 1: `get_dashboard_data()` carica tutto in un singolo payload

Questo design e' semplice, ma costoso:

- cron summary;
- query attive;
- lock waits;
- top tabelle;
- queue metrics;
- system info;
- alerts.

Tutto viene prodotto sempre, anche se l'utente sta guardando solo la card iniziale.

#### Problema 2: N+1 su ultimo log per cron

In `_get_cron_data()`, per ogni cron attivo viene fatta una `Log.search(... limit=1)`.

Con molti cron, questo scala male.

#### Problema 3: caricamento in memoria di tutti i log 24h

`logs_24h = Log.search([('start_time', '>=', yesterday)])` e poi `filtered(...)` in Python.

Questo e' corretto funzionalmente, ma sbagliato architetturalmente se i log crescono.

#### Problema 4: query live pesanti in refresh periodico

La sezione database include ogni volta:

- `pg_stat_activity` completo non-idle;
- join lock waits;
- top 30 tabelle con `pg_total_relation_size()`.

Queste query sono accettabili on-demand, non come parte obbligatoria di ogni refresh.

### Correzione proposta

Separare il backend in endpoint diversi.

#### Endpoint consigliati

1. `dashboard_summary`
   - legge solo dati cached/snapshot;
   - include KPI sintetici, alert attivi, ultimi cron.

2. `dashboard_cron_details`
   - cron list paginata;
   - ultimi log;
   - aggregati su finestra temporale.

3. `dashboard_live_queries`
   - solo su refresh manuale;
   - timeout SQL stretto.

4. `dashboard_live_locks`
   - separato da query live.

5. `dashboard_storage`
   - top tabelle, vacuum, bloat;
   - idealmente da snapshot e non live.

#### Correzione dell'algoritmo cron summary

Usare SQL aggregato o `read_group` invece di caricare tutto in memoria.

Esempio per statistiche ultime 24h:

```sql
SELECT
    count(*) FILTER (WHERE state = 'success') AS success_count,
    count(*) FILTER (WHERE state = 'error') AS error_count,
    avg(duration) AS avg_duration
FROM system_observability_cron_log
WHERE start_time >= %s
```

Esempio per ultimo log per cron:

```sql
SELECT DISTINCT ON (cron_id)
    cron_id, state, duration, start_time
FROM system_observability_cron_log
WHERE cron_id = ANY(%s)
ORDER BY cron_id, start_time DESC
```

#### Correzione sul dato top tabelle

Spostare top tabelle e metriche storage in snapshot periodico ogni 15 minuti o 1 ora.

Non ha alcun valore operativo reale ricalcolarle ogni 30 secondi.

### Raccomandazione finale su F4

Il backend deve essere riprogettato in modalita' `summary-first`, non `everything-live`.

---

## F5. Le metriche sulle code di lavoro non sono coerenti con il piano e possono essere costose

### Riferimenti precisi

- Piano originale: sezione 3.4 e sezione `_get_queue_data()`.
- Codice reale Odoo: `addons/mail/models/mail_mail.py`, riga 46; `addons/mail/models/mail_notification.py`, righe 27 e 36.

### Problemi

#### Problema 1: mismatch tra piano e codice proposto

Nel piano, la tabella di sezione 3.4 dichiara:

- email inviate nelle ultime 24h.

Nel codice `_get_queue_data()` proposto, invece, viene calcolato solo:

- totale `mail.mail` con `state = 'sent'`.

Non e' la stessa metrica.

#### Problema 2: `mail.mail.state` non e' presentato come campo indicizzato nel core

Nel modello reale `mail.mail`, il campo `state` e' dichiarato, ma il piano non affronta il fatto che contare frequentemente per stato su una tabella grande puo' degenerare in full scan ripetuti.

Questo rischio e' reale soprattutto se la dashboard fa refresh ricorrente.

#### Problema 3: `mail.notification.notification_status` e' invece molto piu' adatto a metriche rapide

Il modello `mail.notification` ha campi pensati per tracking di stato, e la review ritiene che per failure e code sia una fonte piu' robusta almeno per alcuni KPI.

### Correzione proposta

#### Scelta architetturale consigliata

Non interrogare `mail.mail` live ad ogni refresh.

Fare invece:

1. snapshot periodico ogni 5-15 minuti per KPI mail globali;
2. dettaglio live solo a richiesta;
3. alert su backlog solo da snapshot cached.

#### Se serve la metrica "sent last 24h"

Hai due opzioni corrette.

##### Opzione A: metrica approssimata ma economica

Usare snapshot differenziali.

Esempio:

- ogni 5 minuti salvi il conteggio totale `sent`;
- il valore 24h e' differenza tra snapshot corrente e quello di 24h prima.

Questa soluzione evita query storiche pesanti sulla tabella live.

##### Opzione B: metrica precisa ma piu' costosa

Aggiungere un timestamp esplicito sul momento di invio tramite inherit su `mail.mail` o hook post-send.

In assenza di questo campo, usare `write_date` non e' semanticamente pulito.

#### Opzione DBA per installazioni grandi

Se il cliente pretende metriche mail live su grande volume, offrire una migrazione SQL separata e opzionale con indice dedicato eseguito dal DBA fuori dalla transazione Odoo:

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS mail_mail_state_create_date_idx
ON mail_mail (state, create_date);
```

Nota importante: non va messo come SQL init nel modulo Odoo standard, perche' `CREATE INDEX CONCURRENTLY` non puo' stare dentro la transazione di installazione.

### Raccomandazione finale su F5

Le queue metrics devono passare da "live every refresh" a "snapshot by default, live only on demand".

---

## F6. Il motore di alert non modella incidenti, solo superamenti di soglia

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, righe 827-927 e sezione 8.

### Problema

L'algoritmo `_check_alerts()` proposto crea un record ogni volta che trova una metrica sopra soglia.

Se una metrica resta sopra soglia per 2 ore e il cron gira ogni 5 minuti, verranno creati molti record ridondanti.

Inoltre:

- non esiste cooldown;
- non esiste deduplica;
- non esiste concetto di incidente aperto;
- `is_resolved` e `resolved_date` esistono nel modello ma non sono davvero usati dalla logica;
- il motore ricalcola l'intera dashboard solo per valutare alcune metriche.

### Conseguenze

- rumore operativo;
- spam di notifiche bus;
- scritture inutili sul database;
- carico evitabile;
- impossibilita' di capire il numero reale di incidenti distinti.

### Correzione proposta

#### Modello corretto: incident state machine

La regola non deve generare un record ogni volta. Deve gestire un incidente aperto.

Proposta minima:

Nel modello `system_observability.alert.rule` aggiungere:

- `last_level`;
- `last_evaluated_at`;
- `last_triggered_at`;
- `cooldown_minutes`;
- `auto_resolve`;
- `notify_on_recovery`.

Nel modello `system_observability.alert.log` aggiungere:

- `state` con valori `open`, `ack`, `resolved`;
- `opened_at`;
- `closed_at`;
- `incident_key`;
- `last_metric_value`.

#### Algoritmo corretto

Per ogni regola:

1. calcolare la metrica necessaria, non l'intera dashboard;
2. determinare `current_level`;
3. leggere eventuale incidente aperto per la regola;
4. se non c'e' incidente e `current_level` e' warning/critical, aprire incidente;
5. se c'e' incidente e il livello peggiora, aggiornare incidente e notificare transizione;
6. se c'e' incidente e il livello resta invariato, aggiornare solo heartbeat interno o nulla;
7. se la metrica rientra, chiudere incidente e marcare `resolved`.

#### Ottimizzazione del calcolo metriche

Non chiamare `dashboard.get_dashboard_data()` dal cron alert.

Invece implementare un registry di resolver:

```python
METRIC_RESOLVERS = {
    'active_connections': _metric_active_connections,
    'failed_crons_24h': _metric_failed_crons_24h,
    'mail_queue_stuck': _metric_mail_queue_stuck,
}
```

e calcolare solo le metriche effettivamente usate dalle regole attive.

#### Uso corretto di `bus.bus`

`bus.bus.sendone()` va usato solo per:

- apertura incidente;
- escalation warning -> critical;
- recovery, se abilitata.

Mai per ogni ciclo di cron sopra soglia.

### Raccomandazione finale su F6

Il motore alert deve essere ripensato come motore incident, non come semplice threshold logger.

---

## F7. Il frontend puo' introdurre carico superfluo e listener residui

### Riferimenti precisi

- Piano originale: `PLAN_OBSERVABILITY.md`, righe 978-1172.
- Codice reale bus frontend: `addons/bus/static/src/js/longpolling_bus.js`, righe 75, 115 e 208; `addons/bus/static/src/js/services/bus_service.js`, riga 95.

### Problemi

#### Problema 1: refresh periodico pieno + refresh su bus notification

La dashboard fa `_fetchData()`:

- all'avvio;
- a intervallo fisso;
- di nuovo ad ogni notifica bus.

Questo puo' generare burst di richieste concorrenti o ravvicinate.

#### Problema 2: nessun controllo di concorrenza sulle RPC

Se una chiamata e' lenta e il timer scatta di nuovo, o arriva una notifica bus, si possono avere fetch multipli contemporanei.

#### Problema 3: cleanup incompleto

Nel `destroy()` proposto viene fermato il timer, ma non viene rimossa esplicitamente la subscription bus e non viene rimosso il canale.

In una SPA backend Odoo questo puo' lasciare listener accumulati alla riapertura dell'action.

#### Problema 4: polling anche quando il tab non e' visibile

Il piano non prevede nessuna sospensione del refresh quando la tab e' in background.

### Correzione proposta

#### Correzione minima JS

Implementare:

- un flag `_isFetching`;
- un `_pendingRefresh`;
- una singola funzione `_scheduleRefresh()` che coalesca richieste multiple.

Schema:

```javascript
_refreshPromise: null,

_refreshDashboard: function () {
    if (this._refreshPromise) {
        return this._refreshPromise;
    }
    this._refreshPromise = this._fetchData()
        .then(this._renderContent.bind(this))
        .always(() => {
            this._refreshPromise = null;
        });
    return this._refreshPromise;
}
```

#### Cleanup corretto

Nel `destroy()`:

- `clearInterval`;
- `this.call('bus_service', 'deleteChannel', channel)`;
- rimozione handler `notification`.

#### Refresh policy corretta

- default 60 o 120 secondi;
- nessun refresh automatico se `document.hidden === true`;
- backoff se il backend risponde con errore;
- dettaglio live con pulsante manuale `Refresh now`.

#### Split del frontend

La schermata iniziale deve mostrare solo summary card e incidenti.

Le tabelle pesanti devono essere caricate solo quando l'utente apre la rispettiva sezione.

### Raccomandazione finale su F7

Il frontend deve essere progettato per ridurre il numero di RPC, non solo per aggiornarsi spesso.

---

## F8. Le azioni operative proposte sono troppo pericolose senza limitazioni forti

### Riferimenti precisi

- Piano originale: righe 47 e sezione 7 (`kill_query`, `cancel_backend`).
- Codice reale di confronto: `odoo/service/db.py`, riga 165 usa `pg_terminate_backend` in uno scenario di amministrazione DB, non di normale backend observability.

### Problema

Gli endpoint proposti permettono di terminare backend PostgreSQL a partire da un PID arbitrario, dopo il solo controllo `base.group_system` e un cast a `int()`.

Questo non basta.

Rischi concreti:

- uccidere il backend della richiesta corrente;
- uccidere processi non legati alla problematica reale;
- uccidere autovacuum o altre attivita' essenziali;
- interrompere altri amministratori o cron;
- esporre un bottone troppo facile da usare in modo improprio;
- fallire comunque in produzione se l'utente DB non ha privilegi adeguati.

### Correzione proposta

#### Regola di base

`pg_cancel_backend()` deve essere l'azione di default.

`pg_terminate_backend()` deve essere:

- opzionale;
- disabilitata di default;
- protetta da conferma esplicita;
- auditable.

#### Guardrail server-side obbligatori

Prima di cancellare o terminare un PID, il controller deve verificare:

1. che il PID esista davvero in `pg_stat_activity`;
2. che appartenga al `current_database()`;
3. che non sia `pg_backend_pid()`;
4. che `backend_type` non sia tra quelli vietati;
5. che il query age superi una soglia minima o che l'utente confermi l'eccezione;
6. che l'azione venga loggata in un audit model dedicato.

Esempio di filtro:

```sql
SELECT pid, backend_type, datname, usename, state, query
FROM pg_stat_activity
WHERE pid = %s
  AND datname = current_database()
  AND pid != pg_backend_pid()
  AND backend_type NOT IN ('autovacuum launcher', 'autovacuum worker', 'walwriter', 'checkpointer')
```

#### Guardrail UI obbligatori

- conferma modale con query fingerprint e durata;
- distinzione visiva tra `cancel` e `terminate`;
- etichetta danger reale;
- eventuale secondo click per terminate.

#### Audit operativo

Creare `system_observability.action.audit` con:

- utente;
- timestamp;
- azione;
- pid;
- query fingerprint;
- esito;
- motivazione libera opzionale.

### Raccomandazione finale su F8

Per la MVP, esporre solo `cancel_backend` e lasciare `terminate_backend` dietro flag di configurazione esplicita.

---

## F9. La protezione dei dati sensibili nelle query SQL e' insufficiente

### Riferimenti precisi

- Piano originale: sezione 9.2 e 9.3.

### Problema

Il piano considera sufficiente troncare `pg_stat_activity.query` a 500 o 100 caratteri.

Questo non elimina il problema principale: i segreti e i dati personali possono trovarsi all'inizio della query.

Esempi reali:

- insert con email;
- update con token;
- literal SQL con valori sensibili;
- query generate da moduli custom.

### Correzione proposta

#### Default sicuro

Mostrare per default solo:

- `pid`;
- `state`;
- durata;
- utente DB;
- fingerprint della query;
- query type (`SELECT`, `UPDATE`, `INSERT`, `DELETE`);
- lunghezza del testo originale.

#### Fingerprinting semplice

Applicare una normalizzazione lato Python:

- sostituire literal string e numeri con placeholder;
- ridurre whitespace;
- tenere solo i primi token significativi.

Esempio:

```text
UPDATE res_users SET password = ? WHERE id = ?
```

#### Raw SQL solo opt-in

L'esposizione del testo completo deve richiedere:

- configurazione specifica attivata in settings;
- gruppo admin dedicato o permesso extra;
- banner di warning in UI.

### Raccomandazione finale su F9

La dashboard non deve mostrare raw SQL per default in produzione.

---

## F10. Il cron degli alert ha un model target ambiguo e la responsabilita' applicativa non e' ben definita

### Riferimenti precisi

- Piano originale: `_check_alerts()` nelle righe 887-927;
- `data/cron_data.xml` nella sezione 8.2.

### Problema

Il cron XML punta a:

- `model_id = model_system_observability_alert_log`

e invoca:

- `model._check_alerts()`.

Nel piano, pero', `_check_alerts()` non e' formalmente incapsulato in una classe chiaramente identificata come `system_observability.alert.log`.

Anche se si potesse implementare cosi', la scelta architetturale resta sbagliata: il modello `alert.log` dovrebbe rappresentare eventi/log, non la logica di valutazione delle regole.

### Correzione proposta

Separare i ruoli.

#### Opzione raccomandata

Creare un model service dedicato:

- `system_observability.alert.engine`

con responsabilita' unica:

- valutare regole;
- aprire/aggiornare/chiudere incidenti;
- emettere notifiche bus.

Il cron XML deve puntare a quel model.

#### Alternativa accettabile

Mettere `_check_alerts()` su `system_observability.alert.rule`.

Non raccomando metterlo su `alert.log`.

### Raccomandazione finale su F10

La responsabilita' di scheduling e valutazione va isolata in un engine model dedicato.

---

## F11. La parte davvero osservativa del sistema e' trattata come opzionale, ma dovrebbe essere nel core del progetto

### Riferimenti precisi

- Piano originale: sezione 10.2 `HTTP Request Metrics`, sezione 11.1 `Trend Analysis su Serie Temporali`, sezione 12 `Piano di Implementazione`.

### Problema

Il piano mette nel corpo principale soprattutto metriche live e dashboard interattiva.

Le componenti che trasformano il modulo in vera osservabilita' vengono spostate piu' avanti:

- HTTP stats;
- snapshot temporali;
- trend analysis;
- anomaly detection.

Questo porta a un rischio pratico: implementare prima la dashboard live e rimandare indefinitamente la raccolta storica, ottenendo un modulo molto visibile ma non abbastanza utile per capire regressioni progressive.

### Correzione proposta

Promuovere `snapshot` e `metric sample` a fase core del progetto.

#### Architettura consigliata

##### Layer A. Passive instrumentation

- cron logs;
- alert incident logs;
- action audit logs.

##### Layer B. Sample collector

Cron leggero che ogni N minuti salva un insieme ridotto di metriche summary in una tabella dedicata.

Esempio:

- active connections;
- idle in transaction;
- long-running query count;
- failed cron 24h;
- mail queue size;
- DB size;
- vacuum pressure.

##### Layer C. Dashboard summary

Legge principalmente l'ultimo sample disponibile, non il database live.

##### Layer D. Live diagnostics

Disponibili solo quando l'amministratore apre il dettaglio e clicca refresh.

### Benefici

- carico prevedibile;
- trend analysis reale;
- alert piu' stabili;
- dashboard veloce;
- separazione pulita tra overview e diagnostica.

### Raccomandazione finale su F11

La tabella snapshot della sezione 11.1 non e' un extra: deve diventare parte della baseline architetturale.

---

## F12. Alcuni snippet del piano non sono pronti per essere implementati cosi' come sono

### Riferimenti precisi

- `_get_system_info()` in `PLAN_OBSERVABILITY.md`, sezione 5.3.
- sezione 10.2 `HTTP Request Metrics`.
- sezione 11.2 `Query Plan Analysis`.
- sezione 9.1 ACL su `system_observability.dashboard`.

### Osservazioni

#### Punto 1: `_get_system_info()` usa `platform` ma non lo importa nello snippet

E' un dettaglio minore, ma segnala che i sample code vanno rivisti prima di essere usati come base di implementazione.

#### Punto 2: `HTTP Request Metrics` accede a dettagli privati e specifici del server mode

La proposta in sezione 10.2 vale solo in condizioni limitate e non deve essere considerata una metrica robusta di primo livello.

#### Punto 3: `EXPLAIN ANALYZE` e' troppo rischioso per essere integrato in un modulo production-safe

Anche con guardrail, e' una funzionalita' da tenere fuori dalla prima versione.

#### Punto 4: ACL su `system_observability.dashboard`

Essendo un `AbstractModel`, la gestione ACL va verificata attentamente. Potrebbe funzionare, ma non va data per scontata come design necessario. Il vero enforcement deve stare su controller e metodi server-side.

### Correzione proposta

- trattare gli snippet del piano come pseudocode, non come specifica eseguibile;
- introdurre una fase di "contract review" prima della scrittura del modulo;
- validare ogni sample code contro il core Odoo reale del repository.

---

## 6. Proposta architetturale corretta: modulo osservativo a impatto minimo

Per rispettare il vincolo di non intaccare il sistema, propongo questa architettura target.

## 6.1 Principio guida

Il modulo non deve interrogare in modo costoso il sistema ogni volta che un amministratore apre una schermata.

Deve invece:

1. raccogliere passivamente il minimo indispensabile;
2. campionare periodicamente pochi KPI sintetici;
3. leggere da snapshot per la dashboard summary;
4. lanciare diagnostica live solo quando richiesta manualmente.

## 6.2 Modelli consigliati

### `system_observability.cron.log`

Persistente, con retention.

Campi aggiuntivi consigliati oltre a quelli gia' previsti:

- `trigger_mode`;
- `worker_pid`;
- `execution_uuid`;
- `db_name`.

### `system_observability.metric.sample`

Nuovo modello core del progetto.

Campi consigliati:

- `collected_at`;
- `metric_key`;
- `scope` (`global`, `process`, `db`);
- `scope_ref` (es. `pid` o `dbname`);
- `value_float`;
- `value_text`;
- `payload_json` opzionale.

### `system_observability.alert.incident`

Meglio di usare solo `alert.log`.

Campi:

- `rule_id`;
- `state` (`open`, `ack`, `resolved`);
- `current_level`;
- `opened_at`;
- `updated_at`;
- `resolved_at`;
- `last_metric_value`;
- `message`.

### `system_observability.action.audit`

Per tutte le azioni operative sensibili.

## 6.3 Endpoint API consigliati

### Economici e frequenti

- `/system_observability/dashboard_summary`
- `/system_observability/alerts_summary`
- `/system_observability/recent_cron_logs`

### Costosi e solo manuali

- `/system_observability/live_queries`
- `/system_observability/live_locks`
- `/system_observability/storage_details`
- `/system_observability/cancel_backend`
- `/system_observability/terminate_backend`

## 6.4 Politica di raccolta

### Summary snapshot every 5 minutes

Include solo KPI sintetici.

### Storage snapshot every 30-60 minutes

Include:

- db size;
- top tables;
- vacuum pressure;
- dead tuple ratios.

### Live diagnostics on demand

Include:

- active queries;
- blocking graph;
- backend details.

---

## 7. Revisione proposta del piano di implementazione

Il piano originale in sezione 12 e' funzionalmente sensato, ma andrebbe riordinato.

## Fase 0. Guardrail architetturali

Prima di tutto:

1. definire fail-open policy;
2. definire metriche consentite live vs snapshot;
3. definire time budget per endpoint e cron interni;
4. definire feature flags per azioni distruttive e raw SQL.

## Fase 1. Passive instrumentation minima

1. `cron.log` con logging sicuro postcommit / separate cursor;
2. `action.audit`;
3. retention e indici;
4. test di non regressione su cron.

## Fase 2. Snapshot layer

1. `metric.sample`;
2. collector cron leggero;
3. collector storage separato e meno frequente;
4. test di costo e latenza.

## Fase 3. Alert engine corretto

1. `alert.rule`;
2. `alert.incident` / `alert.log` rivisti;
3. deduplica, cooldown, recovery;
4. bus notifications solo su transizioni.

## Fase 4. Dashboard summary

1. summary endpoint da snapshot;
2. frontend con refresh conservativo;
3. no live diagnostics automatici.

## Fase 5. Live diagnostics on demand

1. live queries;
2. live locks;
3. storage details;
4. timeout e read-only safeguards.

## Fase 6. Azioni operative guardate

1. `cancel_backend`;
2. `terminate_backend` opzionale e protetto;
3. eventuale trigger manuale sicuro dei cron.

---

## 8. Requisiti prestazionali obbligatori

Per rispettare il vincolo di impatto nullo o trascurabile in produzione, raccomando di fissare questi target prima di scrivere il codice.

## 8.1 Budget di latenza

- `dashboard_summary`: p95 < 200 ms
- `alerts_summary`: p95 < 100 ms
- `recent_cron_logs`: p95 < 150 ms
- `live_queries`: p95 < 1000 ms
- `live_locks`: p95 < 1000 ms
- alert cron: p95 < 1000 ms
- snapshot cron summary: p95 < 1000 ms
- snapshot cron storage: p95 < 3000 ms, con frequenza bassa

## 8.2 Budget di overhead cron instrumentation

Per ogni esecuzione cron monitorata:

- overhead CPU osservabilita' < 5 ms p95;
- zero retry automatici su log write failure;
- massimo una write di successo postcommit e una write di errore separata.

## 8.3 Budget di carico DB

- nessuna query `pg_stat_activity` continua da frontend in background;
- nessun full scan ricorrente di `mail.mail` lato dashboard live;
- nessuna query di storage pesante ogni 30 secondi;
- nessuna notifica bus per sample periodici non incidentali.

## 8.4 Budget frontend

- nessun refresh automatico se tab nascosto;
- nessuna RPC concorrente per la stessa dashboard section;
- nessuna leak di listener bus dopo destroy dell'action.

---

## 9. Safeguard tecnici concreti da imporre nel modulo

Questa sezione e' la piu' importante rispetto al requisito del tuo messaggio.

## 9.1 Tutte le query diagnostiche live devono avere timeout locale

Per ogni cursore usato nelle query di diagnostica live:

```sql
SET LOCAL statement_timeout = '1000ms';
```

Per query particolarmente pesanti:

```sql
SET LOCAL statement_timeout = '3000ms';
```

Il timeout va impostato prima della query e il fallimento va gestito come dato assente, non come errore utente.

## 9.2 Ogni errore del modulo deve degradare a "widget non disponibile"

Esempio di payload summary corretto:

```json
{
  "database": {
    "available": false,
    "error": "statement timeout"
  }
}
```

Non deve mai salire una 500 all'utente per colpa dell'osservabilita'.

## 9.3 Nessuna dipendenza dal modulo per il comportamento del sistema

Il modulo deve essere puramente additivo.

Non deve:

- modificare semantica di commit business;
- alterare scheduling dei cron esistenti;
- aggiungere patch che cambino i percorsi HTTP standard;
- imporre lock aggiuntivi su tabelle core durante richieste normali.

## 9.4 Feature flags di sicurezza

Configurazioni da prevedere in `res.config.settings`:

- `enable_live_diagnostics`
- `enable_backend_cancel`
- `enable_backend_terminate`
- `show_raw_sql_text`
- `summary_refresh_interval`
- `details_refresh_interval`

Default raccomandati:

- live diagnostics: `False` o `True` solo per admin e manuale;
- terminate backend: `False`;
- raw SQL text: `False`;
- summary refresh: `60` o `120`.

## 9.5 Nessuna write ripetitiva non necessaria

Bus notifications solo per transizioni di alert.

Non per:

- ogni sample;
- ogni refresh dashboard;
- ogni cron riuscito.

---

## 10. Test aggiuntivi indispensabili rispetto al piano originale

La sezione 13 del piano va estesa con test piu' mirati alla sicurezza operativa.

## 10.1 Test di fail-open

1. rompere temporaneamente la tabella `system_observability_cron_log` e verificare che i cron business continuino a funzionare;
2. forzare timeout sulle query live e verificare che la dashboard mostri widget unavailable invece di fallire;
3. simulare errore del bus e verificare che la dashboard continui a funzionare con refresh manuale.

## 10.2 Test in modalita' multi-worker

1. avviare Odoo con `--workers > 0`;
2. verificare che il pool locale venga mostrato come locale;
3. verificare che i KPI globali vengano da PostgreSQL, non dal pool di un singolo worker.

## 10.3 Test di leak frontend

1. aprire e chiudere la dashboard ripetutamente;
2. verificare assenza di accumulo listener bus;
3. verificare assenza di refresh multipli concorrenti.

## 10.4 Test di carico controllato

1. 10 admin con dashboard aperta;
2. misurare p95 degli endpoint summary;
3. misurare incremento su query DB e connessioni;
4. verificare che il modulo non alteri il throughput del sistema.

## 10.5 Test di sicurezza azioni operative

1. tentare `cancel_backend` su proprio PID e verificare rifiuto;
2. tentare `terminate_backend` su backend_type vietato e verificare rifiuto;
3. verificare creazione audit log per ogni azione;
4. verificare conferma lato UI.

## 10.6 Test di coerenza osservativa

1. cron successo -> log `success` con durata > 0;
2. cron errore -> log `error` con traceback e durata valorizzata;
3. trigger manuale, se esposto -> `trigger_mode='manual'`;
4. stesso alert sopra soglia per 30 minuti -> un incidente aperto, non 6 record duplicati.

---

## 11. Decisioni consigliate per la MVP

Per massimizzare valore e minimizzare rischio, la prima release dovrebbe includere solo questo sottoinsieme.

## Da includere subito

- cron execution log sicuro;
- metric sample summary;
- dashboard summary da snapshot;
- alert engine con incident deduplicati;
- recent cron logs;
- queue metrics cached;
- live query list solo manuale;
- `cancel_backend` con guardrail.

## Da rimandare

- `terminate_backend`;
- run cron manuale;
- explain analyze;
- ORM cache deep metrics;
- HTTP internals non robusti;
- WAL e bloat analysis live ad alta frequenza.

---

## 12. Conclusione finale

Il piano originale e' valido come base concettuale e mostra buona comprensione di Odoo 14, soprattutto nella lettura dei cron, del bus e delle fonti PostgreSQL.

Tuttavia, se lo si implementa alla lettera, il modulo rischia di:

- loggare in modo non sempre corretto le esecuzioni cron;
- fornire KPI fuorvianti sul pool in modalita' prefork;
- introdurre query troppo pesanti nel caricamento live della dashboard;
- generare alert rumorosi e ridondanti;
- esporre azioni operative troppo pericolose;
- avere un impatto non trascurabile sulle performance in presenza di refresh frequenti.

La correzione di fondo e' questa:

**spostare il progetto da una dashboard live monolitica a una architettura production-safe fatta di instrumentation passiva, snapshot periodici, diagnostica live solo on-demand e azioni operative fortemente guardate.**

Se questa correzione viene adottata, il modulo puo' diventare realmente utile in produzione senza alterare il comportamento del sistema.

Se non viene adottata, il rischio e' costruire un modulo molto ricco in UI ma troppo invasivo, costoso o impreciso proprio nei momenti in cui servirebbe di piu'.

---

## 13. Priorita' esecutive raccomandate

Ordine di lavoro consigliato:

1. correggere il design dell'instrumentation cron;
2. introdurre lo strato snapshot summary;
3. ridisegnare alert come incident engine deduplicato;
4. separare summary endpoint da live diagnostics;
5. ridurre aggressivita' del frontend;
6. blindare azioni operative;
7. solo dopo, completare UI e metriche avanzate.

Questa priorita' massimizza robustezza e minimizza rischio sistemico.