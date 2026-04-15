# PLAN_OBSERVABILITY_FINAL.md

Piano finale per l'implementazione del modulo `system_observability` su Odoo 14.

Questo documento sostituisce l'uso operativo diretto di `PLAN_OBSERVABILITY.md` e recepisce integralmente i vincoli emersi in `CODEX_PLAN_REVIEW.md`.
L'obiettivo non e' solo costruire una dashboard, ma introdurre un modulo di osservabilita' production-safe, a impatto minimo, testabile in modo incrementale e adatto sia a uno sviluppatore umano sia a un coding agent.

---

## 1. Decisioni Architetturali Vincolanti

Queste decisioni non sono opzionali. Tutte le fasi successive devono rispettarle.

### 1.1 Fail-open obbligatorio

- Il modulo non deve mai interrompere cron esistenti, richieste HTTP business o commit applicativi.
- Ogni eccezione del modulo deve essere catturata, loggata e degradata in modo innocuo.
- Se una componente di osservabilita' fallisce, il comportamento corretto e' `dato non disponibile`, non errore bloccante.

### 1.2 Separazione tra summary data e live diagnostics

- La dashboard principale deve leggere quasi esclusivamente da snapshot e dati persistiti recenti.
- Le diagnostiche costose devono essere disponibili solo on-demand.
- E' vietato un endpoint unico che ricalcola in live tutte le metriche del sistema a ogni refresh.

### 1.3 Nessuna scrittura telemetrica nella transazione business critica

- I log di successo dei cron devono essere scritti solo dopo commit, tramite `postcommit` o meccanismo equivalente.
- I log di errore dei cron devono essere scritti con cursore separato, dopo la gestione del rollback del core.
- Nessuna write telemetrica deve condividere il destino della transazione business osservata.

### 1.4 Polling conservativo di default

- Refresh dashboard summary: default 60 secondi.
- Nessun refresh automatico se il tab browser non e' visibile.
- Nessuna RPC concorrente per la stessa sezione.
- Le sezioni live devono essere aggiornate solo su richiesta esplicita.

### 1.5 Sicurezza forte sulle azioni operative

- `pg_cancel_backend()` puo' entrare nella MVP solo con guardrail completi.
- `pg_terminate_backend()` resta disabilitato di default e fuori scope MVP.
- Il trigger manuale dei cron non entra nella MVP.
- Tutte le azioni operative devono essere auditabili.

### 1.6 Protezione dati sensibili

- Le query SQL live non devono mostrare testo raw per default.
- La UI deve mostrare fingerprint, tipo query, durata e metadati essenziali.
- L'eventuale esposizione di SQL raw deve stare dietro feature flag esplicita e warning dedicato.

### 1.7 Budget prestazionali minimi

| Ambito | Target |
|--------|--------|
| `dashboard_summary` | p95 < 200 ms |
| `recent_cron_logs` | p95 < 150 ms |
| `alerts_summary` | p95 < 100 ms |
| `live_queries` | p95 < 1000 ms |
| `live_locks` | p95 < 1000 ms |
| collector summary | p95 < 1000 ms |
| collector storage | p95 < 3000 ms |
| overhead cron instrumentation | < 5 ms p95 per esecuzione |

### 1.8 Timeout locali sulle query diagnostiche

- Tutte le query live verso PostgreSQL devono impostare `SET LOCAL statement_timeout`.
- Timeout standard: 1000 ms per query/lock live.
- Timeout storage: massimo 3000 ms.
- Un timeout deve produrre un widget `unavailable`, non una 500.

---

## 2. Scope Finale

### 2.1 In scope MVP

- logging sicuro delle esecuzioni cron;
- retention e indici sui log;
- metriche summary persistite via snapshot;
- motore alert con incidenti deduplicati;
- dashboard summary veloce e conservativa;
- recent cron logs;
- queue metrics cached;
- live diagnostics manuali e separati;
- `cancel_backend` con guardrail e audit.

### 2.2 Fuori scope MVP

- `pg_terminate_backend()`;
- trigger manuale dei cron;
- `EXPLAIN ANALYZE` integrato;
- metriche profonde su cache ORM e internals HTTP non robusti;
- analisi WAL, bloat o replication ad alta frequenza;
- qualunque metrica del connection pool locale usata come KPI globale o base di alert critico.

### 2.3 Regola di interpretazione

In caso di conflitto tra piano originale e review:

- prevalgono sicurezza operativa, fail-open e snapshot-first;
- una feature utile ma rischiosa viene rimandata;
- una metrica locale o imprecisa non diventa KPI globale.

---

## 3. Architettura Target

### 3.1 Struttura del modulo attesa

```text
addons/system_observability/
|-- __init__.py
|-- __manifest__.py
|-- controllers/
|   |-- __init__.py
|   |-- main.py
|-- models/
|   |-- __init__.py
|   |-- action_audit.py
|   |-- alert_engine.py
|   |-- alert_incident.py
|   |-- alert_rule.py
|   |-- cron_execution_log.py
|   |-- ir_cron_inherit.py
|   |-- metric_sample.py
|   |-- observability_dashboard.py
|   |-- res_config_settings.py
|-- security/
|   |-- ir.model.access.csv
|-- views/
|   |-- alert_views.xml
|   |-- dashboard_views.xml
|   |-- menu.xml
|   |-- res_config_settings_views.xml
|   |-- cron_log_views.xml
|   |-- templates.xml
|-- static/
|   |-- src/
|       |-- css/dashboard.css
|       |-- js/dashboard_main.js
|       |-- xml/dashboard_templates.xml
|-- data/
|   |-- config_data.xml
|   |-- cron_data.xml
|-- tests/
|   |-- __init__.py
|   |-- test_alert_engine.py
|   |-- test_cron_logging.py
|   |-- test_dashboard_summary.py
|   |-- test_live_diagnostics.py
|   |-- test_metric_sampling.py
|   |-- test_operational_actions.py
```

I nomi dei file possono variare se necessario, ma i contratti funzionali qui descritti non devono cambiare.

### 3.2 Modelli core

#### `system_observability.cron.log`

Scopo:

- persistenza di ogni esecuzione cron;
- storico consultabile;
- base per dashboard, alert e troubleshooting.

Campi minimi:

- `cron_id`
- `cron_name`
- `start_time`
- `end_time`
- `duration`
- `state` (`success`, `error`)
- `error_message`
- `user_id`
- `trigger_mode` (`scheduled`, `manual`, `recovered`)
- `worker_pid`
- `execution_uuid`
- `db_name`

#### `system_observability.metric.sample`

Scopo:

- snapshot leggero e periodico dei KPI sintetici;
- base per dashboard summary e trend.

Campi minimi:

- `collected_at`
- `metric_key`
- `scope` (`global`, `db`, `process`)
- `scope_ref`
- `value_float`
- `value_text`
- `payload_json`

#### `system_observability.alert.rule`

Scopo:

- definizione di metriche monitorate e soglie.

Campi minimi:

- `name`
- `metric_key`
- `active`
- `threshold_warning`
- `threshold_critical`
- `cooldown_minutes`
- `auto_resolve`
- `notify_on_recovery`

#### `system_observability.alert.incident`

Scopo:

- modellare un incidente aperto, non una sequenza di record duplicati.

Campi minimi:

- `rule_id`
- `state` (`open`, `ack`, `resolved`)
- `current_level` (`warning`, `critical`)
- `opened_at`
- `updated_at`
- `resolved_at`
- `last_metric_value`
- `message`
- `incident_key`

#### `system_observability.action.audit`

Scopo:

- audit di ogni azione operativa sensibile.

Campi minimi:

- `action_type`
- `executed_by`
- `executed_at`
- `target_pid`
- `query_fingerprint`
- `outcome`
- `reason`

### 3.3 Service layer atteso

- `ir.cron` inherit per instrumentation sicura;
- `system_observability.alert.engine` per la valutazione delle regole;
- `system_observability.dashboard` come service/read model per i payload UI;
- controller distinti per summary, live diagnostics e azioni operative.

### 3.4 Endpoint attesi

Endpoint economici e frequenti:

- `/system_observability/dashboard_summary`
- `/system_observability/alerts_summary`
- `/system_observability/recent_cron_logs`

Endpoint costosi e solo manuali:

- `/system_observability/live_queries`
- `/system_observability/live_locks`
- `/system_observability/storage_details`
- `/system_observability/cancel_backend`

Endpoint esplicitamente escluso dalla MVP:

- `/system_observability/terminate_backend`

### 3.5 Parametri di configurazione minimi

| Chiave | Default | Note |
|-------|---------|------|
| `system_observability.summary_refresh_interval` | `60` | secondi |
| `system_observability.details_refresh_interval` | `0` | `0` = manuale |
| `system_observability.log_retention_days` | `30` | cron log e sample |
| `system_observability.summary_sample_interval_minutes` | `5` | collector leggero |
| `system_observability.storage_sample_interval_minutes` | `60` | collector storage |
| `system_observability.long_query_threshold` | `60` | secondi |
| `system_observability.enable_live_diagnostics` | `False` | manuale/admin only |
| `system_observability.enable_backend_cancel` | `True` | guardrail obbligatori |
| `system_observability.enable_backend_terminate` | `False` | fuori MVP |
| `system_observability.show_raw_sql_text` | `False` | opt-in esplicito |

---

## 4. Piano di Implementazione Per Fasi

Le fasi sono sequenziali salvo esplicita indicazione contraria. Non si passa alla fase successiva senza avere chiuso i test di accettazione della fase corrente.

## Fase 0 - Guardrail architetturali e scaffolding

### Prerequisiti

- Nessuno.

### Obiettivi

- Formalizzare i contratti architetturali del modulo.
- Preparare lo scheletro del modulo senza introdurre ancora logica costosa.
- Definire feature flag, budget e confini di responsabilita'.

### Dettagli implementativi

- Creare la struttura base del modulo `system_observability`.
- Preparare `__manifest__.py` con dipendenze minime: `base`, `bus`, `mail`.
- Definire in `res.config.settings` e `ir.config_parameter` tutte le feature flag minime.
- Formalizzare nel codice una distinzione netta tra:
  - dati summary da snapshot;
  - dati live on-demand;
  - azioni operative.
- Stabilire il contratto di degradazione dei payload UI:

```json
{
  "available": false,
  "error": "statement timeout"
}
```

- Stabilire che i controller summary non devono eseguire query diagnostiche pesanti.
- Definire da subito i file test placeholder per ogni area del modulo.

### Come testare

1. Installazione a vuoto del modulo su database di test senza errori di manifest o import.
2. Verifica che tutte le chiavi di configurazione di default vengano create correttamente.
3. Verifica che un controller summary non implementato restituisca errore gestito o placeholder, mai traceback utente.
4. Revisione tecnica del codice per confermare che nessuna feature fuori scope MVP sia esposta.

## Fase 1 - Passive instrumentation: cron log e audit

### Prerequisiti

- Fase 0 completata.
- Conferma del contratto fail-open.

### Obiettivi

- Tracciare in modo affidabile successi e fallimenti dei cron.
- Rendere persistente lo storico cron con retention e indici.
- Introdurre il modello di audit per future azioni operative.

### Dettagli implementativi

- Implementare `system_observability.cron.log` con indici su:
  - `(cron_id, start_time desc)`
  - `(state, start_time desc)`
- Implementare retention automatica dei log con `@api.autovacuum` o cron dedicato.
- Fare override near-verbatim del core di `ir.cron._callback()` invece del pattern `super() + inferenza successo`.
- Regola obbligatoria:
  - log `success` scritto solo dopo commit;
  - log `error` scritto con cursore separato dopo rollback.
- Salvare `duration` con `time.monotonic()`.
- Salvare `worker_pid`, `execution_uuid`, `db_name` e `trigger_mode`.
- Introdurre `system_observability.action.audit` anche se ancora non usato dalla UI, per avere il modello gia' disponibile.
- Tutta la strumentazione deve catturare e loggare internamente ogni eccezione senza propagarla.

### Come testare

1. Cron di test che termina correttamente: deve generare un log `success` con `duration > 0`.
2. Cron di test che solleva eccezione: deve generare un log `error` con traceback valorizzato.
3. Simulazione di failure sulla tabella `cron.log`: il cron business deve continuare a eseguirsi.
4. Verifica che in multi-worker il logging resti consistente e non rompa il runner.
5. Verifica di non regressione confrontando il comportamento del cron con il core Odoo 14.

## Fase 2 - Snapshot layer e collector periodici

### Prerequisiti

- Fase 1 completata.
- Disponibilita' del modello `metric.sample` e delle configurazioni di retention.

### Obiettivi

- Spostare i KPI sintetici su raccolta periodica e non su calcolo live.
- Preparare la base dati per dashboard veloce e trend futuri.
- Rendere economiche le metriche su code e database.

### Dettagli implementativi

- Implementare `system_observability.metric.sample` con indice su `(metric_key, collected_at desc)`.
- Introdurre due collector distinti:
  - collector summary ogni 5 minuti;
  - collector storage ogni 60 minuti.
- Il collector summary deve salvare almeno:
  - `db_connections_total`
  - `db_connections_active`
  - `db_connections_idle_in_transaction`
  - `db_connection_pressure_pct`
  - `cron_failed_24h`
  - `cron_overdue_count`
  - `mail_outgoing_count`
  - `mail_exception_count`
  - `notification_exception_count`
  - `db_size_bytes`
- Il collector storage deve salvare almeno:
  - top tabelle per dimensione;
  - dead tuple ratio;
  - ultimo vacuum / analyze;
  - storage pressure sintetica.
- Le metriche mail e notification devono essere raccolte via snapshot, non via full refresh continuo della dashboard.
- Le metriche del connection pool Odoo, se mostrate, devono essere etichettate come `current process only` e mai usate per alert globale.
- Introdurre retention per i sample coerente con il piano: default 30 giorni.

### Come testare

1. Esecuzione manuale del collector summary: deve creare sample coerenti per tutte le chiavi minime.
2. Esecuzione manuale del collector storage: deve creare payload non vuoti e leggibili.
3. Query sull'ultimo sample per metrica: p95 < 50 ms su dataset di test.
4. Verifica che il collector non esegua query storage pesanti nel ciclo summary.
5. Verifica retention sample: dati oltre la finestra configurata vengono rimossi senza impatto funzionale.

## Fase 3 - Alert engine a incidenti deduplicati

### Prerequisiti

- Fase 1 completata.
- Fase 2 completata.
- Elenco finale delle metriche effettivamente allertabili.

### Obiettivi

- Trasformare gli alert da semplici threshold log in incidenti gestiti correttamente.
- Evitare spam, duplicati e notifiche inutili.
- Valutare solo le metriche necessarie alle regole attive.

### Dettagli implementativi

- Implementare `system_observability.alert.rule` e `system_observability.alert.incident`.
- Implementare `system_observability.alert.engine` come service model dedicato.
- Il cron degli alert deve puntare all'engine, non a `alert.log` o ad altri modelli di storage.
- Implementare resolver per singola metrica, ad esempio:
  - `_metric_active_connections()`
  - `_metric_failed_crons_24h()`
  - `_metric_mail_queue_stuck()`
- L'engine non deve chiamare `get_dashboard_data()` completo.
- Algoritmo minimo obbligatorio per ogni regola:
  1. calcolo valore corrente;
  2. determinazione livello corrente;
  3. ricerca incidente aperto;
  4. apertura incidente se assente e sopra soglia;
  5. aggiornamento solo se cambia stato o severita';
  6. chiusura se rientra sotto soglia.
- Le notifiche bus devono essere inviate solo su transizioni:
  - apertura;
  - escalation warning -> critical;
  - recovery, se abilitata.

### Come testare

1. Regola sopra soglia per 30 minuti: deve esistere un solo incidente aperto, non record duplicati a ogni run.
2. Escalation warning -> critical: incidente aggiornato e singola notifica bus.
3. Recovery: incidente chiuso con `resolved_at` valorizzato.
4. Cooldown attivo: nessuna nuova notifica entro la finestra configurata.
5. Verifica che l'engine non carichi l'intero payload dashboard per valutare una sola metrica.

## Fase 4 - Dashboard summary, viste e UX conservativa

### Prerequisiti

- Fase 2 completata.
- Fase 3 completata.

### Obiettivi

- Esporre una dashboard veloce, stabile e utile per l'operativita' quotidiana.
- Presentare summary, incidenti e ultimi cron senza dipendere da query live pesanti.
- Introdurre le viste e la navigazione del modulo in modo coerente con Odoo.

### Dettagli implementativi

- Implementare `system_observability.dashboard` come read model orientato a summary.
- Implementare almeno questi endpoint separati:
  - `/system_observability/dashboard_summary`
  - `/system_observability/alerts_summary`
  - `/system_observability/recent_cron_logs`
- Il payload summary deve includere solo:
  - ultimi sample per KPI principali;
  - incidenti aperti;
  - ultimi log cron;
  - configurazione di refresh.
- Implementare client action JS con queste regole:
  - una sola fetch in corso per volta;
  - refresh coalescente;
  - stop refresh quando `document.hidden === true`;
  - cleanup esplicito di timer e listener bus in `destroy()`.
- Le notifiche bus devono aggiornare banner/toast e schedulare un refresh summary, non forzare burst di chiamate concorrenti.
- Creare menu, action, viste tree/form/search per:
  - cron log;
  - alert rule;
  - alert incident;
  - settings.
- Le ACL devono essere limitate a `base.group_system`.

### Come testare

1. `dashboard_summary` con p95 < 200 ms su ambiente di test realistico.
2. Apertura dashboard con 10 admin concorrenti: nessun accumulo anomalo di RPC o connessioni.
3. Cambio tab browser: il refresh automatico deve fermarsi e riprendere solo al ritorno in foreground.
4. Apertura e chiusura ripetuta della dashboard: nessun listener bus residuo.
5. Utente non admin: accesso negato a dashboard, log, alert e settings.

## Fase 5 - Live diagnostics on-demand

### Prerequisiti

- Fase 4 completata.
- Feature flag `enable_live_diagnostics` disponibile.

### Obiettivi

- Fornire diagnostica live utile senza caricare inutilmente il sistema.
- Separare chiaramente overview e troubleshooting.
- Proteggere i dati sensibili presenti nelle query SQL.

### Dettagli implementativi

- Implementare endpoint separati e manuali:
  - `/system_observability/live_queries`
  - `/system_observability/live_locks`
  - `/system_observability/storage_details`
- Ogni endpoint deve:
  - impostare `SET LOCAL statement_timeout`;
  - usare solo query read-only;
  - restituire struttura `available/error` in caso di timeout o failure controllata.
- `live_queries` deve restituire per default:
  - `pid`
  - `state`
  - `backend_type`
  - `usename`
  - `duration_sec`
  - `query_type`
  - `query_fingerprint`
  - `is_long`
  - `is_blocked`
- Nessun testo raw completo della query deve essere mostrato di default.
- `live_locks` deve esporre il grafo minimo bloccato/bloccante, non un payload monolitico generico.
- `storage_details` deve essere caricata solo quando l'utente apre la sezione dedicata o clicca refresh.
- La UI deve avere pulsante `Refresh now` per ogni sezione live.

### Come testare

1. Endpoint `live_queries` e `live_locks` con p95 < 1000 ms in ambiente di test.
2. Forzando `statement_timeout`, la UI deve mostrare `Data unavailable` senza 500.
3. Verifica che le query mostrate siano fingerprintate e prive di literal sensibili.
4. Verifica che nessuna sezione live parta in automatico all'apertura dashboard.
5. Verifica che le query storage non vengano eseguite finche' la sezione non viene richiesta.

## Fase 6 - Azioni operative guardate

### Prerequisiti

- Fase 4 completata.
- Fase 5 completata.
- Modello `action.audit` disponibile e testato.

### Obiettivi

- Abilitare una sola azione operativa nella MVP: `cancel_backend`.
- Rendere ogni azione sicura, auditabile e chiaramente distinta dalla sola osservazione.
- Evitare esposizione prematura di funzionalita' distruttive.

### Dettagli implementativi

- Implementare solo `/system_observability/cancel_backend` nella MVP.
- Guardrail server-side obbligatori prima della cancellazione:
  - PID esistente;
  - PID appartenente a `current_database()`;
  - PID diverso da `pg_backend_pid()`;
  - `backend_type` non appartenente a lista vietata;
  - controllo permessi su `base.group_system`;
  - feature flag `enable_backend_cancel` attiva.
- Eseguire audit write su `system_observability.action.audit` per ogni tentativo, riuscito o fallito.
- UI con conferma esplicita e visualizzazione di fingerprint query e durata.
- `pg_terminate_backend()` non deve essere esposto nella UI MVP.
- Il trigger manuale dei cron deve restare non implementato nella MVP.

### Come testare

1. Tentativo di cancellare il PID corrente: rifiuto esplicito.
2. Tentativo di cancellare backend di tipo vietato: rifiuto esplicito.
3. PID inesistente: errore gestito e audit creato.
4. Cancellazione valida: query interrotta e audit valorizzato.
5. Con feature flag disattiva: endpoint non operativo anche per admin.

## Fase 7 - Hardening finale, test end-to-end e readiness al rollout

### Prerequisiti

- Fasi 0-6 completate.

### Obiettivi

- Consolidare il modulo prima del rollout reale.
- Verificare in modo esplicito che l'osservabilita' non alteri il comportamento del sistema.
- Chiudere documentazione, test e criteri di accettazione finali.

### Dettagli implementativi

- Completare la suite test del modulo su:
  - cron instrumentation;
  - sample collector;
  - alert engine;
  - dashboard summary;
  - live diagnostics;
  - cancel backend;
  - ACL e settings.
- Aggiungere test di fail-open dedicati.
- Aggiungere test multi-worker.
- Aggiungere test di leak frontend e concorrenza refresh.
- Aggiungere test di carico controllato con piu' admin concorrenti.
- Verificare compatibilita' di tutte le query PostgreSQL con il target supportato dal progetto.
- Documentare chiaramente in README interno:
  - cosa entra nella MVP;
  - cosa resta off di default;
  - come leggere i KPI;
  - limiti noti.

### Come testare

1. Installazione modulo: `./odoo-bin -d testdb -i system_observability --test-enable --stop-after-init` senza errori.
2. Test fail-open: rottura controllata della write telemetrica senza impatto sui cron business.
3. Test multi-worker: KPI globali da PostgreSQL corretti, pool locale etichettato come locale.
4. Test frontend: nessun refresh concorrente, nessun listener residuo, nessun polling su tab nascosto.
5. Test di carico: dashboard summary e alert engine entro i budget definiti.

---

## 5. Ordine di esecuzione consigliato

Ordine obbligatorio:

1. Fase 0
2. Fase 1
3. Fase 2
4. Fase 3
5. Fase 4
6. Fase 5
7. Fase 6
8. Fase 7

Parallelizzazione consentita dopo Fase 1:

- parte della Fase 2 e preparazione della Fase 3 possono procedere in parallelo;
- la UI summary non parte prima che Fase 2 e Fase 3 siano stabili.

Motivazione:

- prima si rende affidabile la raccolta passiva;
- poi si costruisce il livello snapshot;
- solo dopo si espone la UI e la diagnostica live.

---

## 6. Criteri di accettazione globali

Il modulo puo' essere considerato pronto per la fase implementativa completa solo se tutte le condizioni seguenti sono vere.

### 6.1 Correttezza osservativa

- un cron riuscito genera un log `success` corretto;
- un cron fallito genera un log `error` corretto;
- gli alert non duplicano incidenti a ogni ciclo;
- i KPI globali non derivano da metriche locali fuorvianti.

### 6.2 Sicurezza operativa

- nessuna feature distruttiva non protetta e' esposta;
- nessun raw SQL sensibile e' mostrato di default;
- ogni azione operativa e' auditata;
- gli utenti non admin non possono accedere al modulo.

### 6.3 Prestazioni

- i target di latenza della sezione 1.7 sono rispettati;
- non esistono query pesanti in refresh automatico summary;
- non esistono refresh concorrenti lato frontend;
- il modulo non introduce degrado percepibile nel throughput del sistema.

### 6.4 Robustezza

- il modulo degrada a `widget unavailable` in caso di problemi locali;
- i cron business restano funzionanti anche in caso di failure del logging telemetrico;
- i collector non saturano connection pool o DB.

---

## 7. Backlog post-MVP

Queste voci sono ammesse solo dopo il completamento delle fasi MVP e dopo una nuova review tecnica.

- `pg_terminate_backend()` con doppia conferma e guardrail estesi;
- trigger manuale cron sicuro con semantica esplicita;
- trend analysis avanzata e report storici piu' ricchi;
- metriche opzionali su cache ORM;
- monitoring filestore, replication, WAL e bloat in modalita' dedicata;
- esposizione opzionale del raw SQL solo per contesti altamente controllati.

---

## 8. Decisione finale

La baseline implementativa approvata e' questa:

- instrumentation passiva sicura;
- snapshot-first per summary e KPI;
- incident engine deduplicato;
- dashboard rapida e conservativa;
- live diagnostics solo manuali;
- azioni operative ridotte al minimo e fortemente guardate.

Qualunque implementazione che torni a un modello `dashboard monolitica live`, che scriva telemetria dentro transazioni business o che esponga azioni distruttive senza guardrail non e' conforme a questo piano finale.