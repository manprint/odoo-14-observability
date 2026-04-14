# PLAN_OBSERVABILITY.md

# Modulo Odoo 14: System Observability

Piano tecnico dettagliato per la realizzazione di un modulo di osservabilita' completa del sistema.

---

## Indice

1. [Visione e Obiettivi](#1-visione-e-obiettivi)
2. [Panoramica Architetturale](#2-panoramica-architetturale)
3. [Aree di Monitoraggio](#3-aree-di-monitoraggio)
4. [Struttura del Modulo](#4-struttura-del-modulo)
5. [Dettaglio Tecnico dei Modelli](#5-dettaglio-tecnico-dei-modelli)
6. [Dashboard Frontend](#6-dashboard-frontend)
7. [Controller e API](#7-controller-e-api)
8. [Sistema di Alert e Notifiche](#8-sistema-di-alert-e-notifiche)
9. [Sicurezza e Permessi](#9-sicurezza-e-permessi)
10. [Idee Aggiuntive di Monitoraggio](#10-idee-aggiuntive-di-monitoraggio)
11. [Tecniche di Analisi Avanzate](#11-tecniche-di-analisi-avanzate)
12. [Piano di Implementazione](#12-piano-di-implementazione)
13. [Testing e Verifica](#13-testing-e-verifica)

---

## 1. Visione e Obiettivi

### Il Problema

Odoo 14 non dispone di un sistema di osservabilita' integrato. Le informazioni critiche sul funzionamento del sistema sono distribuite tra:

- **Log del server** (file di testo, difficili da consultare in tempo reale)
- **Cataloghi di sistema PostgreSQL** (accessibili solo via psql o pgAdmin)
- **Stato interno dell'applicazione** (connection pool, registry, cache — non esposti nell'interfaccia)

Questo significa che problemi come cron bloccati, deadlock nel database, esaurimento del connection pool, query lente o code di email accumulate vengono scoperti solo quando hanno gia' causato un impatto visibile sugli utenti.

### L'Obiettivo

Creare un **modulo Odoo nativo** che fornisca:

- **Dashboard unificata** — un singolo punto di ingresso per monitorare lo stato dell'intero sistema
- **Monitoraggio real-time** — aggiornamento automatico dei dati con push notifications via bus.bus
- **Tracciamento storico** — log persistente delle esecuzioni cron con durata e stato
- **Alert proattivi** — soglie configurabili che generano avvisi prima che i problemi diventino critici
- **Azioni operative** — possibilita' di intervenire direttamente dalla dashboard (terminare query, riavviare cron)

### Perche' un Modulo Odoo e Non uno Strumento Esterno

- **Accesso diretto all'ORM** — puo' leggere stati interni (registry, cache, connection pool) non accessibili dall'esterno
- **Integrazione nativa** — menu, permessi, notifiche usano l'infrastruttura Odoo esistente
- **Zero configurazione infrastrutturale** — nessun agent da installare, nessun servizio separato da mantenere
- **Coerenza con il flusso di lavoro** — gli amministratori restano nell'interfaccia che gia' conoscono

---

## 2. Panoramica Architetturale

### Come Funziona Odoo Internamente (Contesto Necessario)

Per comprendere cosa monitorare, e' essenziale capire l'architettura interna di Odoo 14.

#### Il Server

Odoo puo' funzionare in due modalita':

- **ThreadedServer** (default, `--workers=0`): un singolo processo Python con thread multipli. Ogni richiesta HTTP viene gestita da un thread. I cron job vengono eseguiti da thread dedicati (default: 2 cron thread, configurabile con `--max-cron-threads`).

- **PreforkServer** (`--workers=N`): un processo master che fork-a N worker. Ogni worker gestisce una richiesta alla volta. Ci sono worker dedicati ai cron (`WorkerCron`). Questa e' la modalita' raccomandata per produzione.

In entrambi i casi, il collo di bottiglia principale e' il **connection pool PostgreSQL**. Ogni cursor aperto occupa una connessione. Il pool ha un limite massimo (`--db_maxconn`, default 64). Se si esaurisce, le nuove richieste falliscono con `PoolError: The Connection Pool Is Full`.

#### I Cron Job

I cron sono record del modello `ir.cron` (tabella `ir_cron`). Ogni cron e' collegato a una `ir.actions.server` che contiene il codice Python da eseguire.

Il flusso di esecuzione e':
1. Il cron thread chiama `ir.cron._acquire_job(db_name)` ogni ~60 secondi
2. `_process_jobs()` esegue una query: `SELECT * FROM ir_cron WHERE nextcall <= now() AND active AND numbercall != 0 ORDER BY priority`
3. Per ogni job, tenta di acquisire un lock con `SELECT ... FOR UPDATE NOWAIT`
4. Se il lock riesce, chiama `_process_job()` che a sua volta chiama `_callback()`
5. `_callback()` esegue `self.env['ir.actions.server'].browse(server_action_id).run()`
6. Se c'e' un'eccezione, `_handle_callback_exception()` fa rollback della transazione

**Problema attuale**: il timing dell'esecuzione e' loggato solo in modalita' DEBUG. Non esiste tracciamento persistente di successi, fallimenti o durate.

#### Il Database

Odoo usa PostgreSQL come unico backend. Alcune caratteristiche rilevanti:

- **Isolation level**: `REPEATABLE READ` per cursor serializzati, `READ COMMITTED` altrimenti
- **Locking**: `FOR UPDATE NOWAIT` usato per cron e sequenze per evitare deadlock
- **Autovacuum**: PostgreSQL esegue VACUUM automaticamente, ma tabelle molto trafficate possono accumulare dead tuples
- **Connection pool**: gestito dalla classe `ConnectionPool` in `odoo/sql_db.py`, con lock threading per thread-safety

#### Il Bus (Real-Time)

Il modulo `bus` fornisce un sistema di notifiche push basato su PostgreSQL `NOTIFY/LISTEN`:

1. Il server scrive un messaggio nella tabella `bus_bus` e esegue `NOTIFY imbus`
2. Il dispatcher ascolta su `LISTEN imbus` e risveglia i thread/greenlet in attesa
3. Il client JS mantiene una connessione longpolling su `/longpolling/poll`
4. Quando arriva una notifica, il callback JS viene invocato

Questo meccanismo e' quello che useremo per aggiornare la dashboard in tempo reale.

---

## 3. Aree di Monitoraggio

### 3.1 Monitor Cron Job

| Metrica | Sorgente | Tipo |
|---------|----------|------|
| Cron attivi totali | `ir.cron` (ORM) | Conteggio |
| Cron in ritardo (nextcall < now) | `ir.cron` (ORM) | Conteggio + lista |
| Ultima esecuzione per cron | `system_observability.cron.log` | Datetime |
| Durata ultima esecuzione | `system_observability.cron.log` | Secondi |
| Stato ultima esecuzione | `system_observability.cron.log` | success/error |
| Errori nelle ultime 24h | `system_observability.cron.log` | Conteggio |
| Durata media nelle ultime 24h | `system_observability.cron.log` | Secondi |
| Traceback errore | `system_observability.cron.log` | Testo |
| Cron attualmente in esecuzione | `pg_stat_activity` (query contenente 'ir_cron') | Lista |

### 3.2 Salute Database

| Metrica | Sorgente | Tipo |
|---------|----------|------|
| Connessioni attive | `pg_stat_activity` | Conteggio |
| Connessioni idle | `pg_stat_activity` | Conteggio |
| Connessioni idle in transaction | `pg_stat_activity` | Conteggio |
| Query in esecuzione | `pg_stat_activity` (state='active') | Lista con durata |
| Query bloccate (in attesa di lock) | `pg_stat_activity` (wait_event_type='Lock') | Lista |
| Query lente (durata > soglia) | `pg_stat_activity` | Lista |
| Lock totali | `pg_locks` | Conteggio |
| Lock in attesa (non concessi) | `pg_locks` (granted=false) | Conteggio + dettaglio |
| Dipendenze di lock (potenziali deadlock) | `pg_locks` JOIN self | Grafo |
| Dimensione database | `pg_database_size()` | Bytes |
| Top 30 tabelle per dimensione | `pg_stat_user_tables` + `pg_total_relation_size()` | Lista |
| Dead tuples per tabella | `pg_stat_user_tables` | Conteggio |
| Rapporto dead/live tuples | `pg_stat_user_tables` | Percentuale |
| Ultimo VACUUM per tabella | `pg_stat_user_tables` | Datetime |
| Ultimo ANALYZE per tabella | `pg_stat_user_tables` | Datetime |
| Versione PostgreSQL | `SELECT version()` | Stringa |

### 3.3 Connection Pool Odoo

| Metrica | Sorgente | Tipo |
|---------|----------|------|
| Connessioni usate | `odoo.sql_db._Pool._connections` | Conteggio |
| Connessioni totali nel pool | `odoo.sql_db._Pool._connections` | Conteggio |
| Limite massimo pool | `odoo.sql_db._Pool._maxconn` | Intero |
| Percentuale utilizzo pool | Calcolato | Percentuale |
| Connessioni leaked | `odoo.sql_db._Pool._connections` (attr 'leaked') | Conteggio |

### 3.4 Code di Lavoro

| Metrica | Sorgente | Tipo |
|---------|----------|------|
| Email in coda (outgoing) | `mail.mail` state='outgoing' | Conteggio |
| Email fallite (exception) | `mail.mail` state='exception' | Conteggio |
| Email inviate nelle ultime 24h | `mail.mail` state='sent' + data | Conteggio |
| SMS in coda (se modulo installato) | `sms.sms` state='outgoing' | Conteggio |
| SMS in errore | `sms.sms` state='error' | Conteggio |
| Notifiche fallite | `mail.notification` status='exception' | Conteggio |
| Tipo di errore notifiche | `mail.notification` failure_type | Distribuzione |

### 3.5 Informazioni di Sistema

| Metrica | Sorgente | Tipo |
|---------|----------|------|
| Versione Odoo | `odoo.release.version` | Stringa |
| Versione Python | `platform.python_version()` | Stringa |
| Versione PostgreSQL | `SELECT version()` | Stringa |
| Sistema operativo | `platform.platform()` | Stringa |
| Numero worker | `odoo.tools.config['workers']` | Intero |
| Thread cron massimi | `odoo.tools.config['max_cron_threads']` | Intero |
| Limite connessioni DB | `odoo.tools.config['db_maxconn']` | Intero |
| Limite memoria soft/hard | Config | MB |
| Limite tempo CPU/real | Config | Secondi |
| Utilizzo memoria processo | `resource.getrusage()` | MB |

---

## 4. Struttura del Modulo

```
addons/system_observability/
|
|-- __init__.py
|-- __manifest__.py
|
|-- controllers/
|   |-- __init__.py
|   |-- main.py                          # Endpoint JSON-RPC per dashboard e azioni
|
|-- models/
|   |-- __init__.py
|   |-- cron_execution_log.py            # Modello persistente: log esecuzioni cron
|   |-- ir_cron_inherit.py               # Estensione ir.cron per intercettare esecuzioni
|   |-- observability_dashboard.py       # AbstractModel: aggregazione di tutti i dati
|   |-- observability_alert.py           # Modelli: regole alert + log alert
|   |-- res_config_settings.py           # Estensione impostazioni
|
|-- security/
|   |-- ir.model.access.csv             # Permessi di accesso ai modelli
|
|-- views/
|   |-- templates.xml                    # Registrazione asset JS/CSS in web.assets_backend
|   |-- dashboard_views.xml              # Client action per la dashboard
|   |-- cron_log_views.xml               # Tree/Form/Search per log esecuzioni cron
|   |-- alert_views.xml                  # Tree/Form per regole e log alert
|   |-- res_config_settings_views.xml    # Estensione vista Settings
|   |-- menu.xml                         # Menu e sottomenu
|
|-- static/
|   |-- description/
|   |   |-- icon.png                     # Icona del modulo
|   |-- src/
|       |-- css/
|       |   |-- dashboard.css            # Stili dashboard
|       |-- js/
|       |   |-- dashboard_main.js        # Client action principale (AbstractAction)
|       |   |-- dashboard_widgets.js     # Widget per le singole card KPI
|       |-- xml/
|           |-- dashboard_templates.xml  # QWeb templates per il rendering client-side
|
|-- data/
    |-- cron_data.xml                    # Azione pianificata per check alert
    |-- config_data.xml                  # Parametri di default (ir.config_parameter)
```

### `__manifest__.py`

```python
{
    'name': 'System Observability',
    'version': '14.0.1.0.0',
    'category': 'Technical',
    'summary': 'Dashboard di monitoraggio real-time per sistema e database',
    'description': """
        Monitoraggio completo dell'applicazione Odoo:
        - Tracciamento esecuzioni cron job (durata, stato, errori)
        - Salute database PostgreSQL (connessioni, lock, dimensioni, vacuum)
        - Connection pool Odoo (utilizzo, limiti)
        - Code di lavoro (email, SMS, notifiche)
        - Sistema di alert con soglie configurabili
        - Dashboard real-time con auto-refresh e notifiche push
    """,
    'author': 'Custom',
    'license': 'LGPL-3',
    'depends': ['base', 'bus', 'mail'],
    'data': [
        'security/ir.model.access.csv',
        'data/config_data.xml',
        'data/cron_data.xml',
        'views/templates.xml',
        'views/dashboard_views.xml',
        'views/cron_log_views.xml',
        'views/alert_views.xml',
        'views/res_config_settings_views.xml',
        'views/menu.xml',
    ],
    'qweb': [
        'static/src/xml/dashboard_templates.xml',
    ],
    'installable': True,
    'application': True,
}
```

---

## 5. Dettaglio Tecnico dei Modelli

### 5.1 `system_observability.cron.log` — Log Esecuzioni Cron

Questo e' l'unico modello con tabella database dedicata che il modulo crea (a parte le regole e i log alert). Ogni record rappresenta una singola esecuzione di un cron job.

**Perche' un modello dedicato e non ir_logging?** Il modello `ir_logging` esiste gia' in Odoo ed e' usato per il logging generico delle server action. Tuttavia:
- Non ha campi per durata, stato strutturato, o riferimento al cron
- I record vengono inseriti via SQL diretto per evitare lock, rendendo difficile l'estensione
- Non ha indici ottimizzati per query aggregate (media durate, conteggio errori)

```python
# models/cron_execution_log.py

class CronExecutionLog(models.Model):
    _name = 'system_observability.cron.log'
    _description = 'Cron Execution Log'
    _order = 'start_time desc'

    cron_id = fields.Many2one(
        'ir.cron',
        string='Cron Job',
        index=True,
        ondelete='set null',  # Preserva i log anche se il cron viene eliminato
    )
    cron_name = fields.Char(
        string='Cron Name',
        required=True,
        # Copia del nome al momento dell'esecuzione, per riferimento storico
    )
    start_time = fields.Datetime(string='Start Time', required=True, index=True)
    end_time = fields.Datetime(string='End Time')
    duration = fields.Float(string='Duration (s)', help='Durata in secondi')
    state = fields.Selection(
        [('success', 'Success'), ('error', 'Error')],
        string='State',
        required=True,
        index=True,
    )
    error_message = fields.Text(string='Error Details')
    user_id = fields.Many2one('res.users', string='Execution User')

    @api.autovacuum
    def _gc_old_logs(self):
        """Elimina log piu' vecchi del periodo di retention configurato."""
        days = int(self.env['ir.config_parameter'].sudo().get_param(
            'system_observability.log_retention_days', '30'
        ))
        cutoff = fields.Datetime.subtract(fields.Datetime.now(), days=days)
        old_logs = self.sudo().search([('start_time', '<', cutoff)])
        old_logs.unlink()
```

**Indice SQL** per performance delle query aggregate:

```python
    _sql_constraints = []

    def init(self):
        # Indice composito per query frequenti nella dashboard
        self.env.cr.execute("""
            CREATE INDEX IF NOT EXISTS idx_cron_log_cron_start
            ON system_observability_cron_log (cron_id, start_time DESC)
        """)
        self.env.cr.execute("""
            CREATE INDEX IF NOT EXISTS idx_cron_log_state_start
            ON system_observability_cron_log (state, start_time DESC)
        """)
```

### 5.2 `ir.cron` (inherit) — Intercettazione Esecuzioni

Questo e' il pezzo piu' delicato dell'implementazione. Dobbiamo intercettare il flusso di esecuzione dei cron senza romperlo.

#### Analisi del Codice Originale

Il metodo `_callback()` in `odoo/addons/base/models/ir_cron.py` (linea 96):

```python
# CODICE ORIGINALE (non modificare questo file!)
@api.model
def _callback(self, cron_name, server_action_id, job_id):
    try:
        if self.pool != self.pool.check_signaling():
            self.env.reset()
            self = self.env()[self._name]

        log_depth = (None if _logger.isEnabledFor(logging.DEBUG) else 1)
        odoo.netsvc.log(...)
        start_time = False
        if _logger.isEnabledFor(logging.DEBUG):
            start_time = time.time()  # <-- Solo in DEBUG!
        self.env['ir.actions.server'].browse(server_action_id).run()
        if start_time and _logger.isEnabledFor(logging.DEBUG):
            end_time = time.time()
            _logger.debug('%.3fs ...', end_time - start_time, ...)
        self.pool.signal_changes()
    except Exception as e:
        self.pool.reset_changes()
        _logger.exception("Call from cron %s ...", cron_name, ...)
        self._handle_callback_exception(cron_name, server_action_id, job_id, e)
```

Il metodo `_handle_callback_exception()` (linea 89):

```python
# CODICE ORIGINALE
@api.model
def _handle_callback_exception(self, cron_name, server_action_id, job_id, job_exception):
    self._cr.rollback()  # <-- Rollback della transazione!
```

#### Strategia di Override

**Problema chiave**: quando un cron fallisce, `_handle_callback_exception()` esegue `self._cr.rollback()`. Questo annulla tutto cio' che e' nella transazione corrente, incluso qualsiasi record di log che avremmo creato. Quindi per i log di errore dobbiamo usare un **cursore separato**.

```python
# models/ir_cron_inherit.py

import time
import traceback
import odoo
from odoo import api, fields, models

class IrCronInherit(models.Model):
    _inherit = 'ir.cron'

    @api.model
    def _callback(self, cron_name, server_action_id, job_id):
        """Override per tracciare timing e stato di ogni esecuzione."""
        start = time.time()
        try:
            super()._callback(cron_name, server_action_id, job_id)
        except Exception:
            # L'eccezione e' gia' stata gestita da _handle_callback_exception
            # nel super(). Se arriviamo qui, qualcosa di molto grave e' successo.
            # Non rilanciamo per non rompere il flusso del cron runner.
            return

        # Se arriviamo qui, l'esecuzione e' riuscita
        # (gli errori sono catturati internamente dal try/except del super)
        end = time.time()
        try:
            self.env['system_observability.cron.log'].sudo().create({
                'cron_id': job_id,
                'cron_name': cron_name,
                'start_time': fields.Datetime.now(),  # approssimazione
                'end_time': fields.Datetime.now(),
                'duration': end - start,
                'state': 'success',
                'user_id': self.env.uid,
            })
        except Exception:
            # Non vogliamo MAI che il logging rompa l'esecuzione del cron
            _logger.warning("Failed to log cron execution for %s", cron_name,
                          exc_info=True)

    @api.model
    def _handle_callback_exception(self, cron_name, server_action_id,
                                   job_id, job_exception):
        """Override per loggare l'errore prima del rollback."""
        # Catturiamo il traceback PRIMA del rollback
        error_tb = ''.join(traceback.format_exception(
            type(job_exception), job_exception,
            job_exception.__traceback__
        ))

        # Chiamiamo super() che fa il rollback
        super()._handle_callback_exception(
            cron_name, server_action_id, job_id, job_exception
        )

        # Dopo il rollback, usiamo un NUOVO cursore per inserire il log
        # Questo e' lo stesso pattern usato da ir_logging
        # (odoo/addons/base/models/ir_logging.py)
        try:
            db = odoo.sql_db.db_connect(self._cr.dbname)
            with db.cursor() as log_cr:
                log_cr.execute("""
                    INSERT INTO system_observability_cron_log
                        (cron_id, cron_name, start_time, end_time,
                         duration, state, error_message, user_id,
                         create_date, write_date, create_uid, write_uid)
                    VALUES
                        (%s, %s, %s, %s, %s, 'error', %s, %s,
                         now() at time zone 'UTC',
                         now() at time zone 'UTC', %s, %s)
                """, (
                    job_id, cron_name,
                    fields.Datetime.now(), fields.Datetime.now(),
                    0,  # durata non disponibile nel contesto di errore
                    error_tb,
                    self.env.uid,
                    self.env.uid, self.env.uid,
                ))
        except Exception:
            _logger.warning("Failed to log cron error for %s", cron_name,
                          exc_info=True)
```

**Nota sulla durata in caso di errore**: nel contesto di `_handle_callback_exception`, non abbiamo accesso al timestamp di inizio. Per risolvere questo, possiamo usare una variabile a livello di thread:

```python
import threading

# Variante migliorata con tracking del tempo anche per errori
_cron_start_times = threading.local()

@api.model
def _callback(self, cron_name, server_action_id, job_id):
    _cron_start_times.start = time.time()
    _cron_start_times.cron_name = cron_name
    _cron_start_times.job_id = job_id
    try:
        super()._callback(cron_name, server_action_id, job_id)
    except Exception:
        return
    # ... log successo con durata ...

@api.model
def _handle_callback_exception(self, ...):
    start = getattr(_cron_start_times, 'start', None)
    duration = time.time() - start if start else 0
    # ... log errore con durata corretta ...
```

### 5.3 `system_observability.dashboard` — Provider Dati Dashboard

AbstractModel senza tabella database. Fornisce tutti i dati aggregati per la dashboard attraverso un singolo metodo `get_dashboard_data()`.

```python
# models/observability_dashboard.py

class ObservabilityDashboard(models.AbstractModel):
    _name = 'system_observability.dashboard'
    _description = 'System Observability Dashboard'

    @api.model
    def get_dashboard_data(self):
        """Punto di ingresso principale. Restituisce tutti i dati della dashboard."""
        return {
            'cron': self._get_cron_data(),
            'database': self._get_database_data(),
            'pool': self._get_pool_data(),
            'queues': self._get_queue_data(),
            'system': self._get_system_info(),
            'alerts': self._get_active_alerts(),
            'config': self._get_config(),
        }
```

#### `_get_cron_data()`

```python
    @api.model
    def _get_cron_data(self):
        Cron = self.env['ir.cron'].sudo()
        Log = self.env['system_observability.cron.log'].sudo()
        now = fields.Datetime.now()
        yesterday = fields.Datetime.subtract(now, days=1)

        total = Cron.search_count([])
        active = Cron.search_count([('active', '=', True)])
        overdue = Cron.search_count([
            ('active', '=', True),
            ('nextcall', '<', now),
        ])

        # Statistiche ultime 24h
        logs_24h = Log.search([('start_time', '>=', yesterday)])
        failed_24h = len(logs_24h.filtered(lambda l: l.state == 'error'))
        succeeded_24h = len(logs_24h.filtered(lambda l: l.state == 'success'))
        durations = [l.duration for l in logs_24h if l.duration]
        avg_duration = sum(durations) / len(durations) if durations else 0

        # Lista cron attivi con stato
        crons = Cron.search([('active', '=', True)], order='priority, cron_name')
        cron_list = []
        for cron in crons:
            last_log = Log.search([('cron_id', '=', cron.id)],
                                  order='start_time desc', limit=1)
            cron_list.append({
                'id': cron.id,
                'name': cron.cron_name,
                'nextcall': fields.Datetime.to_string(cron.nextcall),
                'lastcall': fields.Datetime.to_string(cron.lastcall) if cron.lastcall else None,
                'interval': '%d %s' % (cron.interval_number, cron.interval_type),
                'last_duration': last_log.duration if last_log else None,
                'last_state': last_log.state if last_log else None,
                'is_overdue': cron.nextcall < now,
            })

        # Ultimi 20 log
        recent_logs = Log.search([], order='start_time desc', limit=20)
        recent = [{
            'cron_name': log.cron_name,
            'start_time': fields.Datetime.to_string(log.start_time),
            'duration': log.duration,
            'state': log.state,
            'error_message': log.error_message[:200] if log.error_message else None,
        } for log in recent_logs]

        return {
            'total': total,
            'active': active,
            'overdue': overdue,
            'failed_24h': failed_24h,
            'succeeded_24h': succeeded_24h,
            'avg_duration_24h': round(avg_duration, 2),
            'cron_list': cron_list,
            'recent_logs': recent,
        }
```

#### `_get_database_data()`

Questo e' il metodo piu' complesso. Esegue query SQL dirette sui cataloghi di sistema PostgreSQL.

```python
    @api.model
    def _get_database_data(self):
        cr = self.env.cr

        # --- Connessioni attive ---
        cr.execute("""
            SELECT state, count(*) as cnt
            FROM pg_stat_activity
            WHERE datname = current_database()
            GROUP BY state
        """)
        conn_stats = {row[0]: row[1] for row in cr.fetchall()}

        # --- Query attive con dettagli ---
        threshold = int(self.env['ir.config_parameter'].sudo().get_param(
            'system_observability.long_query_threshold', '60'
        ))
        cr.execute("""
            SELECT pid, state, query, usename, backend_type,
                   wait_event_type, wait_event,
                   EXTRACT(EPOCH FROM (now() - query_start)) as duration_sec,
                   query_start
            FROM pg_stat_activity
            WHERE datname = current_database()
              AND pid != pg_backend_pid()
              AND state != 'idle'
            ORDER BY query_start NULLS LAST
        """)
        active_queries = []
        long_running = 0
        blocked = 0
        for row in cr.dictfetchall():
            is_long = row['duration_sec'] and row['duration_sec'] > threshold
            is_blocked = row['wait_event_type'] == 'Lock'
            if is_long:
                long_running += 1
            if is_blocked:
                blocked += 1
            active_queries.append({
                'pid': row['pid'],
                'state': row['state'],
                'query': (row['query'] or '')[:500],  # Tronca per sicurezza
                'user': row['usename'],
                'duration': round(row['duration_sec'] or 0, 1),
                'wait_event': row['wait_event'],
                'is_long': is_long,
                'is_blocked': is_blocked,
            })

        # --- Lock in attesa ---
        cr.execute("""
            SELECT
                blocked_locks.pid AS blocked_pid,
                blocked_activity.usename AS blocked_user,
                blocked_activity.query AS blocked_query,
                blocking_locks.pid AS blocking_pid,
                blocking_activity.usename AS blocking_user,
                blocking_activity.query AS blocking_query
            FROM pg_catalog.pg_locks blocked_locks
            JOIN pg_catalog.pg_stat_activity blocked_activity
                ON blocked_activity.pid = blocked_locks.pid
            JOIN pg_catalog.pg_locks blocking_locks
                ON blocking_locks.locktype = blocked_locks.locktype
                AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
                AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
                AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
                AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
                AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
                AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
                AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
                AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
                AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
                AND blocking_locks.pid != blocked_locks.pid
            JOIN pg_catalog.pg_stat_activity blocking_activity
                ON blocking_activity.pid = blocking_locks.pid
            WHERE NOT blocked_locks.granted
              AND blocked_activity.datname = current_database()
        """)
        lock_waits = [{
            'blocked_pid': r['blocked_pid'],
            'blocked_user': r['blocked_user'],
            'blocked_query': (r['blocked_query'] or '')[:300],
            'blocking_pid': r['blocking_pid'],
            'blocking_user': r['blocking_user'],
            'blocking_query': (r['blocking_query'] or '')[:300],
        } for r in cr.dictfetchall()]

        # --- Dimensione database ---
        cr.execute("SELECT pg_database_size(current_database())")
        db_size = cr.fetchone()[0]

        # --- Top tabelle ---
        cr.execute("""
            SELECT relname,
                   n_live_tup,
                   n_dead_tup,
                   CASE WHEN n_live_tup > 0
                        THEN round(100.0 * n_dead_tup / n_live_tup, 1)
                        ELSE 0 END as dead_pct,
                   last_vacuum,
                   last_autovacuum,
                   last_analyze,
                   last_autoanalyze,
                   pg_total_relation_size(relid) as total_size
            FROM pg_stat_user_tables
            ORDER BY pg_total_relation_size(relid) DESC
            LIMIT 30
        """)
        tables = [{
            'name': r['relname'],
            'live_tuples': r['n_live_tup'],
            'dead_tuples': r['n_dead_tup'],
            'dead_pct': float(r['dead_pct']),
            'last_vacuum': str(r['last_vacuum'] or r['last_autovacuum'] or 'Never'),
            'last_analyze': str(r['last_analyze'] or r['last_autoanalyze'] or 'Never'),
            'size': r['total_size'],
        } for r in cr.dictfetchall()]

        # --- Versione PostgreSQL ---
        cr.execute("SELECT version()")
        pg_version = cr.fetchone()[0]

        return {
            'connections': {
                'active': conn_stats.get('active', 0),
                'idle': conn_stats.get('idle', 0),
                'idle_in_transaction': conn_stats.get('idle in transaction', 0),
                'total': sum(conn_stats.values()),
            },
            'queries': active_queries,
            'long_running_count': long_running,
            'blocked_count': blocked,
            'lock_waits': lock_waits,
            'db_size': db_size,
            'tables': tables,
            'pg_version': pg_version,
        }
```

#### `_get_pool_data()`

```python
    @api.model
    def _get_pool_data(self):
        """Legge lo stato del connection pool di Odoo."""
        import odoo.sql_db
        pool = odoo.sql_db._Pool
        if not pool:
            return {'used': 0, 'total': 0, 'max': 0, 'usage_pct': 0}

        # Copia snapshot della lista (stesso pattern di ConnectionPool.__repr__)
        connections = pool._connections[:]
        used = len([1 for c, u in connections if u])
        total = len(connections)
        leaked = len([1 for c, _ in connections if getattr(c, 'leaked', False)])
        max_conn = pool._maxconn

        return {
            'used': used,
            'total': total,
            'max': max_conn,
            'leaked': leaked,
            'usage_pct': round(100.0 * used / max_conn, 1) if max_conn else 0,
        }
```

#### `_get_queue_data()`

```python
    @api.model
    def _get_queue_data(self):
        result = {}

        # --- Email ---
        Mail = self.env['mail.mail'].sudo()
        for state in ('outgoing', 'sent', 'exception', 'cancel'):
            result['mail_' + state] = Mail.search_count([('state', '=', state)])

        # --- Notifiche fallite ---
        Notif = self.env['mail.notification'].sudo()
        result['notification_failures'] = Notif.search_count([
            ('notification_status', '=', 'exception')
        ])

        # --- SMS (solo se il modulo e' installato) ---
        if self.env.registry.get('sms.sms'):
            Sms = self.env['sms.sms'].sudo()
            for state in ('outgoing', 'sent', 'error', 'canceled'):
                result['sms_' + state] = Sms.search_count([('state', '=', state)])
        else:
            result['sms_installed'] = False

        return result
```

#### `_get_system_info()`

```python
    @api.model
    def _get_system_info(self):
        import resource
        import odoo.release

        cr = self.env.cr
        cr.execute("SELECT version()")
        pg_version = cr.fetchone()[0]

        cr.execute("SELECT pg_postmaster_start_time()")
        pg_start = cr.fetchone()[0]

        config = odoo.tools.config

        # Memoria RSS del processo corrente
        usage = resource.getrusage(resource.RUSAGE_SELF)
        mem_mb = usage.ru_maxrss / 1024  # Linux: ru_maxrss e' in KB

        return {
            'odoo_version': odoo.release.version,
            'python_version': platform.python_version(),
            'pg_version': pg_version,
            'pg_start_time': str(pg_start),
            'os_info': platform.platform(),
            'workers': config.get('workers', 0),
            'max_cron_threads': config.get('max_cron_threads', 2),
            'db_maxconn': config.get('db_maxconn', 64),
            'limit_memory_soft': config.get('limit_memory_soft', 0),
            'limit_memory_hard': config.get('limit_memory_hard', 0),
            'limit_time_cpu': config.get('limit_time_cpu', 0),
            'limit_time_real': config.get('limit_time_real', 0),
            'memory_mb': round(mem_mb, 1),
        }
```

### 5.4 Sistema di Alert

#### Modello `system_observability.alert.rule`

```python
class ObservabilityAlertRule(models.Model):
    _name = 'system_observability.alert.rule'
    _description = 'Observability Alert Rule'

    name = fields.Char(required=True)
    active = fields.Boolean(default=True)
    metric = fields.Selection([
        ('conn_pool_usage', 'Utilizzo Connection Pool (%)'),
        ('active_connections', 'Connessioni Attive'),
        ('waiting_locks', 'Lock in Attesa'),
        ('long_running_queries', 'Query Lunghe (conteggio)'),
        ('failed_crons_24h', 'Cron Falliti (24h)'),
        ('mail_queue_stuck', 'Email in Coda (outgoing)'),
        ('dead_tuples_pct', 'Dead Tuples Max (%)'),
        ('db_size_gb', 'Dimensione Database (GB)'),
        ('idle_in_transaction', 'Connessioni Idle in Transaction'),
    ], required=True)
    threshold_warning = fields.Float(string='Soglia Warning')
    threshold_critical = fields.Float(string='Soglia Critical')
    description = fields.Text(string='Descrizione')
```

#### Modello `system_observability.alert.log`

```python
class ObservabilityAlertLog(models.Model):
    _name = 'system_observability.alert.log'
    _description = 'Observability Alert Log'
    _order = 'create_date desc'

    rule_id = fields.Many2one(
        'system_observability.alert.rule', string='Rule',
        ondelete='set null', index=True,
    )
    level = fields.Selection([
        ('warning', 'Warning'),
        ('critical', 'Critical'),
    ], required=True)
    metric_value = fields.Float(string='Valore Misurato')
    message = fields.Text(string='Messaggio')
    is_resolved = fields.Boolean(string='Risolto', default=False)
    resolved_date = fields.Datetime(string='Data Risoluzione')
```

#### Metodo `_check_alerts()`

Viene chiamato dal cron ogni 5 minuti. Per ogni regola attiva:

1. Legge il valore corrente della metrica dai dati della dashboard
2. Confronta con le soglie warning/critical
3. Se superata, crea un record `alert.log`
4. Invia una notifica push via `bus.bus`

```python
    @api.model
    def _check_alerts(self):
        dashboard = self.env['system_observability.dashboard']
        data = dashboard.get_dashboard_data()
        rules = self.env['system_observability.alert.rule'].search([('active', '=', True)])

        # Mappa metrica -> valore corrente
        metric_values = {
            'conn_pool_usage': data['pool']['usage_pct'],
            'active_connections': data['database']['connections']['active'],
            'waiting_locks': len(data['database']['lock_waits']),
            'long_running_queries': data['database']['long_running_count'],
            'failed_crons_24h': data['cron']['failed_24h'],
            'mail_queue_stuck': data['queues'].get('mail_outgoing', 0),
            'dead_tuples_pct': max([t['dead_pct'] for t in data['database']['tables']] or [0]),
            'db_size_gb': data['database']['db_size'] / (1024**3),
            'idle_in_transaction': data['database']['connections']['idle_in_transaction'],
        }

        for rule in rules:
            value = metric_values.get(rule.metric, 0)
            level = None

            if rule.threshold_critical and value >= rule.threshold_critical:
                level = 'critical'
            elif rule.threshold_warning and value >= rule.threshold_warning:
                level = 'warning'

            if level:
                self.env['system_observability.alert.log'].create({
                    'rule_id': rule.id,
                    'level': level,
                    'metric_value': value,
                    'message': '%s: valore %.2f (soglia %s: %.2f)' % (
                        rule.name, value, level,
                        rule.threshold_critical if level == 'critical'
                        else rule.threshold_warning
                    ),
                })

                # Notifica push via bus
                self.env['bus.bus'].sendone('system_observability', {
                    'type': 'alert',
                    'level': level,
                    'rule': rule.name,
                    'metric': rule.metric,
                    'value': value,
                    'message': '%s: %.2f' % (rule.name, value),
                })
```

### 5.5 Impostazioni

```python
# models/res_config_settings.py

class ResConfigSettings(models.TransientModel):
    _inherit = 'res.config.settings'

    observability_refresh_interval = fields.Integer(
        string='Intervallo Refresh Dashboard (secondi)',
        default=30,
        config_parameter='system_observability.refresh_interval',
    )
    observability_log_retention_days = fields.Integer(
        string='Retention Log Cron (giorni)',
        default=30,
        config_parameter='system_observability.log_retention_days',
    )
    observability_long_query_threshold = fields.Integer(
        string='Soglia Query Lunga (secondi)',
        default=60,
        config_parameter='system_observability.long_query_threshold',
    )
```

---

## 6. Dashboard Frontend

### 6.1 Architettura JS

La dashboard e' implementata come **client action** — un componente JavaScript registrato nell'action registry di Odoo che viene caricato quando l'utente clicca sul menu.

**Perche' client action e non kanban view?**
- Una kanban view richiede record su cui iterare. Il nostro AbstractModel non ha record.
- Abbiamo bisogno di layout personalizzato con sezioni diverse (cron, DB, code, sistema).
- L'account journal dashboard funziona con kanban perche' `account.journal` ha record reali; i nostri dati sono interamente calcolati.

### 6.2 `dashboard_main.js`

```javascript
odoo.define('system_observability.DashboardMain', function (require) {
"use strict";

var AbstractAction = require('web.AbstractAction');
var core = require('web.core');
var QWeb = core.qweb;
var _t = core._t;

var ObservabilityDashboard = AbstractAction.extend({
    template: 'SystemObservabilityDashboard',
    events: {
        'click .o_obs_refresh': '_onClickRefresh',
        'click .o_obs_kill_query': '_onClickKillQuery',
        'click .o_obs_view_cron_logs': '_onClickViewCronLogs',
        'click .o_obs_run_cron': '_onClickRunCron',
        'click .o_obs_view_alerts': '_onClickViewAlerts',
    },

    /**
     * @override
     */
    init: function (parent, action) {
        this._super.apply(this, arguments);
        this.dashboardData = {};
        this._refreshTimer = null;
    },

    /**
     * Carica i dati prima del rendering
     */
    willStart: function () {
        var self = this;
        return this._super.apply(this, arguments).then(function () {
            return self._fetchData();
        });
    },

    /**
     * Rendering iniziale + avvio auto-refresh + sottoscrizione bus
     */
    start: function () {
        var self = this;
        return this._super.apply(this, arguments).then(function () {
            self._renderContent();
            self._startAutoRefresh();
            self._subscribeBus();
        });
    },

    /**
     * Cleanup: ferma timer e rimuove listener bus
     */
    destroy: function () {
        if (this._refreshTimer) {
            clearInterval(this._refreshTimer);
        }
        this._super.apply(this, arguments);
    },

    // ---- Metodi Privati ----

    _fetchData: function () {
        var self = this;
        return this._rpc({
            route: '/system_observability/dashboard_data',
            params: {},
        }).then(function (result) {
            self.dashboardData = result;
        });
    },

    _renderContent: function () {
        var $content = this.$('.o_obs_content');
        $content.html(QWeb.render('ObservabilityDashboardContent', {
            data: this.dashboardData,
            formatSize: this._formatSize,
            formatDuration: this._formatDuration,
        }));
    },

    _startAutoRefresh: function () {
        var self = this;
        var interval = (this.dashboardData.config || {}).refresh_interval || 30;
        this._refreshTimer = setInterval(function () {
            self._fetchData().then(function () {
                self._renderContent();
            });
        }, interval * 1000);
    },

    _subscribeBus: function () {
        this.call('bus_service', 'addChannel', 'system_observability');
        this.call('bus_service', 'onNotification', this,
                  this._onBusNotification);
    },

    _onBusNotification: function (notifications) {
        var self = this;
        _.each(notifications, function (notif) {
            if (notif[0] === 'system_observability' ||
                (notif[1] && notif[1].type === 'alert')) {
                var data = notif[1] || notif;
                // Mostra notifica toast
                self.displayNotification({
                    title: data.level === 'critical' ? 'ALERT CRITICO' : 'Warning',
                    message: data.message || 'Nuovo alert di sistema',
                    type: data.level === 'critical' ? 'danger' : 'warning',
                    sticky: data.level === 'critical',
                });
                // Refresh dashboard
                self._fetchData().then(function () {
                    self._renderContent();
                });
            }
        });
    },

    // ---- Utility ----

    _formatSize: function (bytes) {
        if (bytes >= 1073741824) return (bytes / 1073741824).toFixed(1) + ' GB';
        if (bytes >= 1048576) return (bytes / 1048576).toFixed(1) + ' MB';
        if (bytes >= 1024) return (bytes / 1024).toFixed(1) + ' KB';
        return bytes + ' B';
    },

    _formatDuration: function (seconds) {
        if (!seconds) return '-';
        if (seconds < 1) return (seconds * 1000).toFixed(0) + 'ms';
        if (seconds < 60) return seconds.toFixed(1) + 's';
        return Math.floor(seconds / 60) + 'm ' + Math.floor(seconds % 60) + 's';
    },

    // ---- Event Handlers ----

    _onClickRefresh: function () {
        var self = this;
        this._fetchData().then(function () {
            self._renderContent();
        });
    },

    _onClickKillQuery: function (ev) {
        var pid = $(ev.currentTarget).data('pid');
        var self = this;
        this._rpc({
            route: '/system_observability/kill_query',
            params: {pid: pid},
        }).then(function () {
            self.displayNotification({
                message: _t('Query terminata (PID: ') + pid + ')',
                type: 'success',
            });
            self._onClickRefresh();
        });
    },

    _onClickViewCronLogs: function () {
        this.do_action({
            type: 'ir.actions.act_window',
            name: _t('Cron Execution Logs'),
            res_model: 'system_observability.cron.log',
            view_mode: 'tree,form',
            views: [[false, 'list'], [false, 'form']],
        });
    },

    _onClickRunCron: function (ev) {
        var cronId = $(ev.currentTarget).data('cron-id');
        var self = this;
        this._rpc({
            model: 'ir.cron',
            method: 'method_direct_trigger',
            args: [[cronId]],
        }).then(function () {
            self.displayNotification({
                message: _t('Cron eseguito manualmente'),
                type: 'success',
            });
            self._onClickRefresh();
        });
    },

    _onClickViewAlerts: function () {
        this.do_action({
            type: 'ir.actions.act_window',
            name: _t('Alert Log'),
            res_model: 'system_observability.alert.log',
            view_mode: 'tree,form',
            views: [[false, 'list'], [false, 'form']],
        });
    },
});

core.action_registry.add('system_observability.dashboard', ObservabilityDashboard);
return ObservabilityDashboard;
});
```

### 6.3 QWeb Templates (`dashboard_templates.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">

    <!-- Template principale (usato come 'template' della AbstractAction) -->
    <t t-name="SystemObservabilityDashboard">
        <div class="o_obs_dashboard">
            <div class="o_obs_header d-flex align-items-center justify-content-between p-3">
                <h2 class="m-0">System Observability</h2>
                <div>
                    <button class="btn btn-primary o_obs_refresh">
                        <i class="fa fa-refresh"/> Refresh
                    </button>
                    <button class="btn btn-secondary o_obs_view_alerts ml-2">
                        <i class="fa fa-bell"/> Alert Log
                    </button>
                    <button class="btn btn-secondary o_obs_view_cron_logs ml-2">
                        <i class="fa fa-history"/> Cron Logs
                    </button>
                </div>
            </div>
            <div class="o_obs_content p-3"/>
        </div>
    </t>

    <!-- Contenuto dinamico (ri-renderizzato ad ogni refresh) -->
    <t t-name="ObservabilityDashboardContent">

        <!-- ===== ALERT BANNER ===== -->
        <t t-if="data.alerts &amp;&amp; data.alerts.length">
            <div class="alert alert-danger mb-3" t-foreach="data.alerts" t-as="alert">
                <i class="fa fa-exclamation-triangle"/>
                <strong t-esc="alert.level"/>:
                <span t-esc="alert.message"/>
            </div>
        </t>

        <!-- ===== KPI CARDS (riga superiore) ===== -->
        <div class="row mb-3">

            <!-- Card: Sistema -->
            <div class="col-lg-3 col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header bg-info text-white">
                        <i class="fa fa-server"/> Sistema
                    </div>
                    <div class="card-body">
                        <table class="table table-sm table-borderless mb-0">
                            <tr><td>Odoo</td><td t-esc="data.system.odoo_version"/></tr>
                            <tr><td>Python</td><td t-esc="data.system.python_version"/></tr>
                            <tr><td>Workers</td><td t-esc="data.system.workers"/></tr>
                            <tr><td>Cron Threads</td><td t-esc="data.system.max_cron_threads"/></tr>
                            <tr><td>Memoria</td>
                                <td><t t-esc="data.system.memory_mb"/> MB</td></tr>
                        </table>
                    </div>
                </div>
            </div>

            <!-- Card: Cron Jobs -->
            <div class="col-lg-3 col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header bg-primary text-white">
                        <i class="fa fa-clock-o"/> Cron Jobs
                    </div>
                    <div class="card-body">
                        <div class="d-flex justify-content-around text-center mb-2">
                            <div>
                                <div class="h3 mb-0" t-esc="data.cron.active"/>
                                <small class="text-muted">Attivi</small>
                            </div>
                            <div>
                                <div class="h3 mb-0 text-warning" t-esc="data.cron.overdue"/>
                                <small class="text-muted">In Ritardo</small>
                            </div>
                            <div>
                                <div class="h3 mb-0 text-danger" t-esc="data.cron.failed_24h"/>
                                <small class="text-muted">Errori 24h</small>
                            </div>
                        </div>
                        <small class="text-muted">
                            Durata media: <t t-esc="formatDuration(data.cron.avg_duration_24h)"/>
                        </small>
                    </div>
                </div>
            </div>

            <!-- Card: Database -->
            <div class="col-lg-3 col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header bg-success text-white">
                        <i class="fa fa-database"/> Database
                    </div>
                    <div class="card-body">
                        <div class="d-flex justify-content-around text-center mb-2">
                            <div>
                                <div class="h3 mb-0" t-esc="data.database.connections.active"/>
                                <small class="text-muted">Attive</small>
                            </div>
                            <div>
                                <div class="h3 mb-0" t-esc="data.database.connections.idle"/>
                                <small class="text-muted">Idle</small>
                            </div>
                            <div>
                                <div class="h3 mb-0 text-danger"
                                     t-esc="data.database.lock_waits.length"/>
                                <small class="text-muted">Lock Wait</small>
                            </div>
                        </div>
                        <small class="text-muted">
                            Dimensione: <t t-esc="formatSize(data.database.db_size)"/>
                        </small>
                    </div>
                </div>
            </div>

            <!-- Card: Connection Pool -->
            <div class="col-lg-3 col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header" t-attf-class="card-header text-white
                        #{data.pool.usage_pct > 80 ? 'bg-danger' :
                          data.pool.usage_pct > 50 ? 'bg-warning' : 'bg-secondary'}">
                        <i class="fa fa-plug"/> Connection Pool
                    </div>
                    <div class="card-body">
                        <div class="text-center mb-2">
                            <div class="h2 mb-0">
                                <t t-esc="data.pool.used"/> / <t t-esc="data.pool.max"/>
                            </div>
                            <small class="text-muted">
                                <t t-esc="data.pool.usage_pct"/>% utilizzato
                            </small>
                        </div>
                        <!-- Barra progresso -->
                        <div class="progress">
                            <div class="progress-bar"
                                 t-attf-class="progress-bar
                                     #{data.pool.usage_pct > 80 ? 'bg-danger' :
                                       data.pool.usage_pct > 50 ? 'bg-warning' : 'bg-success'}"
                                 t-attf-style="width: #{data.pool.usage_pct}%"/>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- ===== Code di Lavoro ===== -->
        <div class="row mb-3">
            <div class="col-12">
                <div class="card">
                    <div class="card-header">
                        <i class="fa fa-envelope"/> Code di Lavoro
                    </div>
                    <div class="card-body d-flex justify-content-around text-center">
                        <div>
                            <div class="h4 mb-0" t-esc="data.queues.mail_outgoing || 0"/>
                            <small>Email in Coda</small>
                        </div>
                        <div>
                            <div class="h4 mb-0 text-danger"
                                 t-esc="data.queues.mail_exception || 0"/>
                            <small>Email Fallite</small>
                        </div>
                        <div>
                            <div class="h4 mb-0 text-success"
                                 t-esc="data.queues.mail_sent || 0"/>
                            <small>Email Inviate</small>
                        </div>
                        <div>
                            <div class="h4 mb-0 text-danger"
                                 t-esc="data.queues.notification_failures || 0"/>
                            <small>Notifiche Fallite</small>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- ===== Tabelle Dettaglio ===== -->
        <div class="row mb-3">

            <!-- Query Attive -->
            <div class="col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header">
                        <i class="fa fa-terminal"/> Query Attive
                        (<t t-esc="data.database.queries.length"/>)
                    </div>
                    <div class="card-body p-0">
                        <table class="table table-sm table-hover mb-0">
                            <thead><tr>
                                <th>PID</th><th>Stato</th><th>Durata</th>
                                <th>Query</th><th></th>
                            </tr></thead>
                            <tbody>
                                <t t-foreach="data.database.queries" t-as="q">
                                    <tr t-attf-class="#{q.is_long ? 'table-warning' : ''}
                                                       #{q.is_blocked ? 'table-danger' : ''}">
                                        <td t-esc="q.pid"/>
                                        <td t-esc="q.state"/>
                                        <td t-esc="formatDuration(q.duration)"/>
                                        <td><small t-esc="q.query.substring(0, 100)"/></td>
                                        <td>
                                            <button class="btn btn-sm btn-danger o_obs_kill_query"
                                                    t-att-data-pid="q.pid"
                                                    title="Termina query">
                                                <i class="fa fa-times"/>
                                            </button>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>

            <!-- Esecuzioni Cron Recenti -->
            <div class="col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header">
                        <i class="fa fa-list"/> Ultime Esecuzioni Cron
                    </div>
                    <div class="card-body p-0">
                        <table class="table table-sm table-hover mb-0">
                            <thead><tr>
                                <th>Cron</th><th>Ora</th><th>Durata</th><th>Stato</th>
                            </tr></thead>
                            <tbody>
                                <t t-foreach="data.cron.recent_logs" t-as="log">
                                    <tr t-attf-class="#{log.state === 'error' ? 'table-danger' : ''}">
                                        <td t-esc="log.cron_name"/>
                                        <td><small t-esc="log.start_time"/></td>
                                        <td t-esc="formatDuration(log.duration)"/>
                                        <td>
                                            <span t-attf-class="badge
                                                #{log.state === 'success' ? 'badge-success' : 'badge-danger'}">
                                                <t t-esc="log.state"/>
                                            </span>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>

        <!-- Lock e Tabelle -->
        <div class="row mb-3">

            <!-- Lock in Attesa -->
            <div class="col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header">
                        <i class="fa fa-lock"/>
                        Lock in Attesa (<t t-esc="data.database.lock_waits.length"/>)
                    </div>
                    <div class="card-body p-0">
                        <t t-if="data.database.lock_waits.length === 0">
                            <p class="text-muted text-center p-3 mb-0">
                                Nessun lock in attesa
                            </p>
                        </t>
                        <table t-if="data.database.lock_waits.length > 0"
                               class="table table-sm mb-0">
                            <thead><tr>
                                <th>Bloccato (PID)</th>
                                <th>Bloccante (PID)</th>
                                <th>Query Bloccata</th>
                            </tr></thead>
                            <tbody>
                                <t t-foreach="data.database.lock_waits" t-as="lw">
                                    <tr class="table-danger">
                                        <td t-esc="lw.blocked_pid"/>
                                        <td t-esc="lw.blocking_pid"/>
                                        <td><small t-esc="lw.blocked_query.substring(0, 100)"/></td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>

            <!-- Top Tabelle -->
            <div class="col-md-6 mb-3">
                <div class="card h-100">
                    <div class="card-header">
                        <i class="fa fa-table"/> Top Tabelle per Dimensione
                    </div>
                    <div class="card-body p-0">
                        <table class="table table-sm mb-0">
                            <thead><tr>
                                <th>Tabella</th><th>Dim.</th>
                                <th>Dead %</th><th>Ultimo Vacuum</th>
                            </tr></thead>
                            <tbody>
                                <t t-foreach="data.database.tables.slice(0, 15)" t-as="tbl">
                                    <tr t-attf-class="#{tbl.dead_pct > 20 ? 'table-warning' : ''}">
                                        <td t-esc="tbl.name"/>
                                        <td t-esc="formatSize(tbl.size)"/>
                                        <td t-esc="tbl.dead_pct + '%'"/>
                                        <td><small t-esc="tbl.last_vacuum"/></td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>

    </t>

</templates>
```

### 6.4 Registrazione Asset

```xml
<!-- views/templates.xml -->
<odoo>
    <template id="assets_backend" name="System Observability Assets"
              inherit_id="web.assets_backend">
        <xpath expr="." position="inside">
            <link rel="stylesheet"
                  href="/system_observability/static/src/css/dashboard.css"/>
            <script type="text/javascript"
                    src="/system_observability/static/src/js/dashboard_main.js"/>
            <script type="text/javascript"
                    src="/system_observability/static/src/js/dashboard_widgets.js"/>
        </xpath>
    </template>
</odoo>
```

---

## 7. Controller e API

```python
# controllers/main.py

from odoo import http
from odoo.http import request
from odoo.exceptions import AccessError

class SystemObservabilityController(http.Controller):

    def _check_admin(self):
        if not request.env.user.has_group('base.group_system'):
            raise AccessError("Accesso negato: richiesto gruppo Amministrazione/Impostazioni")

    @http.route('/system_observability/dashboard_data', type='json', auth='user')
    def get_dashboard_data(self):
        """Restituisce tutti i dati della dashboard in un singolo payload JSON."""
        self._check_admin()
        return request.env['system_observability.dashboard'].get_dashboard_data()

    @http.route('/system_observability/kill_query', type='json', auth='user')
    def kill_query(self, pid):
        """Termina un backend PostgreSQL tramite PID.
        Usa pg_terminate_backend() - stesso pattern di odoo/service/db.py:165
        """
        self._check_admin()
        pid = int(pid)  # Sanitizzazione: previene SQL injection
        request.env.cr.execute(
            "SELECT pg_terminate_backend(%s)", (pid,)
        )
        return {'success': True, 'pid': pid}

    @http.route('/system_observability/cancel_backend', type='json', auth='user')
    def cancel_backend(self, pid):
        """Cancella la query corrente di un backend (meno aggressivo di kill).
        Usa pg_cancel_backend() che invia SIGINT al processo.
        """
        self._check_admin()
        pid = int(pid)
        request.env.cr.execute(
            "SELECT pg_cancel_backend(%s)", (pid,)
        )
        return {'success': True, 'pid': pid}
```

---

## 8. Sistema di Alert e Notifiche

### 8.1 Flusso Completo

```
[Cron ogni 5 min] --> _check_alerts()
    |
    +--> Legge dashboard data
    +--> Per ogni regola attiva:
    |      +--> Confronta valore vs soglia
    |      +--> Se superata: crea alert.log + bus.bus.sendone()
    |
    +--> [PostgreSQL NOTIFY imbus]
            |
            +--> [ImDispatch risveglia thread]
                    |
                    +--> [Longpolling risponde al client]
                            |
                            +--> [JS _onBusNotification()]
                                    |
                                    +--> Toast notification + refresh dashboard
```

### 8.2 Dati del Cron Alert

```xml
<!-- data/cron_data.xml -->
<odoo noupdate="1">
    <record id="ir_cron_check_alerts" model="ir.cron">
        <field name="name">System Observability: Verifica Alert</field>
        <field name="model_id" ref="model_system_observability_alert_log"/>
        <field name="state">code</field>
        <field name="code">model._check_alerts()</field>
        <field name="interval_number">5</field>
        <field name="interval_type">minutes</field>
        <field name="numbercall">-1</field>
        <field name="doall" eval="False"/>
        <field name="active" eval="True"/>
    </record>
</odoo>
```

### 8.3 Parametri di Default

```xml
<!-- data/config_data.xml -->
<odoo noupdate="1">
    <record id="config_refresh_interval" model="ir.config_parameter">
        <field name="key">system_observability.refresh_interval</field>
        <field name="value">30</field>
    </record>
    <record id="config_log_retention" model="ir.config_parameter">
        <field name="key">system_observability.log_retention_days</field>
        <field name="value">30</field>
    </record>
    <record id="config_long_query" model="ir.config_parameter">
        <field name="key">system_observability.long_query_threshold</field>
        <field name="value">60</field>
    </record>
</odoo>
```

---

## 9. Sicurezza e Permessi

### 9.1 Accesso ai Modelli

Tutto il modulo e' accessibile **esclusivamente** al gruppo `base.group_system` (Amministrazione / Impostazioni).

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_cron_log,access.cron.log,model_system_observability_cron_log,base.group_system,1,1,1,1
access_alert_rule,access.alert.rule,model_system_observability_alert_rule,base.group_system,1,1,1,1
access_alert_log,access.alert.log,model_system_observability_alert_log,base.group_system,1,1,1,1
access_dashboard,access.dashboard,model_system_observability_dashboard,base.group_system,1,0,0,0
```

### 9.2 Sicurezza delle Query SQL

- Tutte le query ai cataloghi PostgreSQL usano **query parametrizzate** (placeholder `%s`), mai string interpolation
- Il parametro `pid` negli endpoint kill/cancel viene castato a `int()` prima dell'uso
- Le query restituite da `pg_stat_activity` vengono **troncate** a 500 caratteri per evitare di esporre informazioni sensibili eccessivamente lunghe

### 9.3 Dati Sensibili

`pg_stat_activity.query` puo' contenere query con dati sensibili (es. inserimenti con password hashate). La dashboard mostra solo i primi 500 caratteri e le tabelle mostrano solo i primi 100. Il dettaglio completo e' visibile solo espandendo la riga.

---

## 10. Idee Aggiuntive di Monitoraggio

Oltre alle aree core descritte sopra, ecco ulteriori aspetti che il modulo potrebbe monitorare per una copertura veramente completa.

### 10.1 ORM Cache Statistics

Odoo mantiene una cache ORM (`ormcache`) per i risultati di metodi frequenti. Le statistiche sono gia' accessibili internamente:

```python
# Il segnale SIGUSR1 gia' chiama log_ormcache_stats() in server.py
# Possiamo esporre gli stessi dati nella dashboard

from odoo.tools import lru

def _get_ormcache_stats(self):
    """Statistiche della cache ORM: hit rate, dimensione, invalidazioni."""
    stats = []
    for key, cache in self.env.registry._Registry__caches.items():
        if hasattr(cache, 'hit') and hasattr(cache, 'miss'):
            total = cache.hit + cache.miss
            hit_rate = (cache.hit / total * 100) if total else 0
            stats.append({
                'name': str(key),
                'size': len(cache.d) if hasattr(cache, 'd') else 0,
                'hit_rate': round(hit_rate, 1),
                'hits': cache.hit,
                'misses': cache.miss,
            })
    return stats
```

**Utilita'**: un hit rate basso indica che la cache non e' efficace (possibile memory pressure o invalidazioni troppo frequenti). Una cache molto grande puo' indicare memory leak.

### 10.2 HTTP Request Metrics

Monitorare le richieste HTTP in arrivo:

```python
def _get_http_stats(self):
    """Statistiche su thread HTTP attivi e utilizzo semaforo."""
    import odoo.service.server as server_mod
    # Il ThreadedWSGIServerReloadable usa un semaforo per limitare thread
    # Possiamo leggerne lo stato
    srv = getattr(server_mod, 'server', None)
    if srv and hasattr(srv, 'httpd') and hasattr(srv.httpd, 'http_threads_sem'):
        sem = srv.httpd.http_threads_sem
        return {
            'max_threads': sem._initial_value if hasattr(sem, '_initial_value') else 'N/A',
            # Il valore corrente del semaforo non e' direttamente accessibile in modo sicuro
        }
    return {}
```

### 10.3 Monitoraggio Sessioni Utente

```python
def _get_session_stats(self):
    """Conteggio sessioni attive, utenti connessi, ultime attivita'."""
    cr = self.env.cr
    cr.execute("""
        SELECT
            count(DISTINCT s.uid) as active_users,
            count(*) as total_sessions,
            max(s.last_activity) as last_activity
        FROM ir_sessions s
        WHERE s.last_activity > now() - interval '30 minutes'
    """)
    # Nota: ir_sessions potrebbe non esistere in tutte le installazioni
    # Alternativa: contare da res_users.login_date
    return cr.dictfetchone()
```

### 10.4 Monitoraggio Spazio su Disco (Filestore)

Odoo salva allegati nel filestore (`~/.local/share/Odoo/filestore/<dbname>/`):

```python
import shutil

def _get_filestore_stats(self):
    """Dimensione e utilizzo del filestore."""
    filestore_path = odoo.tools.config.filestore(self.env.cr.dbname)
    total, used, free = shutil.disk_usage(filestore_path)

    # Conteggio allegati
    cr = self.env.cr
    cr.execute("SELECT count(*), coalesce(sum(file_size), 0) FROM ir_attachment WHERE store_fname IS NOT NULL")
    att_count, att_size = cr.fetchone()

    return {
        'path': filestore_path,
        'disk_total': total,
        'disk_used': used,
        'disk_free': free,
        'disk_usage_pct': round(100.0 * used / total, 1) if total else 0,
        'attachment_count': att_count,
        'attachment_total_size': att_size,
    }
```

### 10.5 Monitoraggio Indici PostgreSQL

Indici non utilizzati o mancanti sono una delle cause principali di performance scadenti:

```python
def _get_index_stats(self):
    """Indici inutilizzati e tabelle senza indici critici."""
    cr = self.env.cr

    # Indici mai utilizzati (potenziali candidati per la rimozione)
    cr.execute("""
        SELECT s.relname AS table_name,
               s.indexrelname AS index_name,
               pg_relation_size(s.indexrelid) AS index_size,
               s.idx_scan AS scans
        FROM pg_stat_user_indexes s
        JOIN pg_index i ON s.indexrelid = i.indexrelid
        WHERE s.idx_scan = 0
          AND NOT i.indisunique
          AND NOT i.indisprimary
        ORDER BY pg_relation_size(s.indexrelid) DESC
        LIMIT 20
    """)
    unused = cr.dictfetchall()

    # Tabelle con scan sequenziali elevati rispetto agli index scan
    cr.execute("""
        SELECT relname,
               seq_scan, idx_scan,
               CASE WHEN seq_scan + idx_scan > 0
                    THEN round(100.0 * seq_scan / (seq_scan + idx_scan), 1)
                    ELSE 0 END AS seq_pct,
               n_live_tup
        FROM pg_stat_user_tables
        WHERE n_live_tup > 10000
          AND seq_scan > idx_scan
        ORDER BY n_live_tup * seq_scan DESC
        LIMIT 20
    """)
    seq_heavy = cr.dictfetchall()

    return {
        'unused_indexes': unused,
        'sequential_scan_heavy': seq_heavy,
    }
```

### 10.6 Monitoraggio Replication (se applicabile)

Per installazioni con read replica PostgreSQL:

```python
def _get_replication_stats(self):
    """Stato della replicazione (se configurata)."""
    cr = self.env.cr
    cr.execute("""
        SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
               EXTRACT(EPOCH FROM (now() - write_lag)) as write_lag_sec,
               EXTRACT(EPOCH FROM (now() - replay_lag)) as replay_lag_sec
        FROM pg_stat_replication
    """)
    return cr.dictfetchall()
```

### 10.7 Monitoraggio Transazioni Lunghe

```python
def _get_long_transactions(self):
    """Transazioni aperte da troppo tempo (potenziale causa di bloat)."""
    cr = self.env.cr
    cr.execute("""
        SELECT pid, usename, state,
               EXTRACT(EPOCH FROM (now() - xact_start)) as xact_duration_sec,
               query
        FROM pg_stat_activity
        WHERE xact_start IS NOT NULL
          AND state != 'idle'
          AND datname = current_database()
        ORDER BY xact_start
    """)
    return [r for r in cr.dictfetchall() if r['xact_duration_sec'] > 30]
```

### 10.8 Audit Trail Moduli

Monitorare lo stato dei moduli installati e le ultime operazioni di aggiornamento:

```python
def _get_module_stats(self):
    """Stato moduli e ultime operazioni."""
    cr = self.env.cr
    cr.execute("""
        SELECT state, count(*) FROM ir_module_module
        GROUP BY state ORDER BY state
    """)
    states = {r[0]: r[1] for r in cr.fetchall()}

    cr.execute("""
        SELECT name, state, latest_version, write_date
        FROM ir_module_module
        WHERE state NOT IN ('uninstalled', 'uninstallable')
        ORDER BY write_date DESC
        LIMIT 20
    """)
    recent = cr.dictfetchall()

    return {'states': states, 'recently_updated': recent}
```

### 10.9 Monitoraggio Background Jobs (se OCA queue_job e' installato)

Se il modulo OCA `queue_job` e' installato, puo' essere integrato:

```python
def _get_queue_job_stats(self):
    """Statistiche queue_job OCA (se installato)."""
    if not self.env.registry.get('queue.job'):
        return None

    Job = self.env['queue.job'].sudo()
    states = {}
    for state in ('pending', 'enqueued', 'started', 'done', 'failed'):
        states[state] = Job.search_count([('state', '=', state)])

    # Job falliti recenti
    failed = Job.search([('state', '=', 'failed')],
                        order='date_done desc', limit=10)
    return {
        'states': states,
        'recent_failures': [{
            'name': j.name,
            'func': j.func_string,
            'date': str(j.date_done),
            'exc_info': (j.exc_info or '')[:500],
        } for j in failed],
    }
```

### 10.10 Monitoraggio WAL (Write-Ahead Log)

Il WAL e' cruciale per le performance di PostgreSQL e per la recovery:

```python
def _get_wal_stats(self):
    """Statistiche Write-Ahead Log."""
    cr = self.env.cr
    cr.execute("SELECT pg_current_wal_lsn()")
    current_lsn = cr.fetchone()[0]

    cr.execute("""
        SELECT * FROM pg_stat_bgwriter
    """)
    bgwriter = cr.dictfetchone()

    return {
        'current_lsn': str(current_lsn),
        'checkpoints_timed': bgwriter.get('checkpoints_timed'),
        'checkpoints_requested': bgwriter.get('checkpoints_req'),
        'buffers_checkpoint': bgwriter.get('buffers_checkpoint'),
        'buffers_backend': bgwriter.get('buffers_backend'),
    }
```

---

## 11. Tecniche di Analisi Avanzate

### 11.1 Trend Analysis su Serie Temporali

Salvando periodicamente snapshot delle metriche chiave, possiamo calcolare trend:

```python
class ObservabilitySnapshot(models.Model):
    """Snapshot periodico delle metriche (ogni 5 min via cron)."""
    _name = 'system_observability.snapshot'
    _description = 'Metric Snapshot'
    _order = 'timestamp desc'

    timestamp = fields.Datetime(default=fields.Datetime.now, index=True)
    metric_name = fields.Char(index=True)
    metric_value = fields.Float()

    @api.autovacuum
    def _gc_old_snapshots(self):
        """Mantieni solo 7 giorni di snapshot."""
        cutoff = fields.Datetime.subtract(fields.Datetime.now(), days=7)
        self.search([('timestamp', '<', cutoff)]).unlink()
```

Con questi dati si possono calcolare:

- **Moving average**: media mobile a 1h/6h/24h per smussare il rumore
- **Rate of change**: variazione percentuale tra snapshot consecutivi
- **Anomaly detection**: deviazione standard; valori oltre 2 sigma sono anomali
- **Proiezione**: se la dimensione del DB cresce di X MB/giorno, quando raggiungera' il limite disco?

### 11.2 Query Plan Analysis

Per le query lente identificate, possiamo raccogliere il piano di esecuzione:

```python
def _analyze_slow_query(self, query_text):
    """Esegue EXPLAIN ANALYZE sulla query (attenzione: la esegue davvero!)."""
    # ATTENZIONE: usare solo con query SELECT, MAI con INSERT/UPDATE/DELETE
    if not query_text.strip().upper().startswith('SELECT'):
        return {'error': 'Solo query SELECT possono essere analizzate'}

    cr = self.env.cr
    try:
        cr.execute("EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " + query_text)
        plan = cr.fetchone()[0]
        return plan
    except Exception as e:
        return {'error': str(e)}
```

**Nota di sicurezza**: l'EXPLAIN ANALYZE esegue effettivamente la query. Per evitare rischi:
- Eseguire solo su SELECT
- Wrappare in una transazione con ROLLBACK
- Impostare un timeout con `SET LOCAL statement_timeout = '5s'`

### 11.3 Table Bloat Estimation

Il bloat delle tabelle (spazio sprecato da dead tuples e frammentazione) e' una causa comune di degradazione:

```sql
-- Query per stimare il bloat delle tabelle
-- Adattata da check_postgres
SELECT
    current_database() AS db,
    schemaname, tablename,
    pg_total_relation_size(schemaname || '.' || tablename) AS total_bytes,
    CASE WHEN otta=0 OR sml.relpages=0 OR sml.relpages=otta THEN 0.0
         ELSE sml.relpages::float/otta END AS tbloat,
    CASE WHEN relpages < otta THEN 0
         ELSE (bs*(sml.relpages-otta))::bigint END AS wastedbytes
FROM (
    SELECT
        schemaname, tablename, cc.reltuples, cc.relpages, bs,
        CEIL((cc.reltuples*((datahdr+ma-
            (CASE WHEN datahdr%ma=0 THEN ma ELSE datahdr%ma END)
        )+nullhdr2+4))/(bs-20::float)) AS otta
    FROM (
        SELECT
            ma,bs,schemaname,tablename,
            (datawidth+(hdr+ma-(case when hdr%ma=0 THEN ma ELSE hdr%ma END)))::numeric AS datahdr,
            (maxfracsum*(nullhdr+ma-(case when nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2
        FROM (
            SELECT
                s.schemaname, s.tablename, hdr, ma, bs,
                SUM((1-null_frac)*avg_width) AS datawidth,
                MAX(null_frac) AS maxfracsum,
                hdr+(
                    SELECT 1+count(*)/8
                    FROM pg_stats s2
                    WHERE null_frac<>0 AND s2.schemaname = s.schemaname AND s2.tablename = s.tablename
                ) AS nullhdr
            FROM pg_stats s,
                (SELECT
                    (SELECT current_setting('block_size')::numeric) AS bs,
                    CASE WHEN SUBSTRING(SPLIT_PART(v, ' ', 2) FROM '#"[0-9]+#"%' FOR '#')
                        IN ('8','9','10','11','12') THEN 27 ELSE 23 END AS hdr,
                    CASE WHEN v ~ 'mingw32' OR v ~ '64-bit' THEN 8 ELSE 4 END AS ma
                FROM (SELECT version() AS v) AS foo
                ) AS constants
            GROUP BY 1,2,3,4,5
        ) AS foo
    ) AS rs
    JOIN pg_class cc ON cc.relname = rs.tablename
    JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = rs.schemaname AND nn.nspname <> 'information_schema'
) AS sml
ORDER BY wastedbytes DESC
LIMIT 20;
```

### 11.4 Connection Pool Pressure Analysis

Analizzando i pattern di utilizzo del pool nel tempo:

- **Peak usage**: se il pool raggiunge regolarmente l'80%+, e' necessario aumentare `db_maxconn` o ridurre il numero di worker
- **Leaked connections**: se il conteggio "leaked" cresce, c'e' un bug nel codice che non rilascia i cursori
- **Pool exhaustion patterns**: correlando con gli orari, si possono identificare i picchi (es. invio massivo di email, batch import)

### 11.5 Cron Execution Pattern Recognition

Con lo storico delle esecuzioni cron possiamo identificare:

- **Degradazione progressiva**: un cron che impiega sempre piu' tempo ad ogni esecuzione (possibile leak o tabella che cresce senza manutenzione)
- **Fallimenti periodici**: un cron che fallisce sempre allo stesso orario (possibile conflitto con un altro cron o con un batch esterno)
- **Jitter eccessivo**: un cron con durate molto variabili (potrebbe dipendere dal carico del sistema o da query non ottimizzate)

```python
def _get_cron_trends(self, cron_id, days=7):
    """Calcola trend di esecuzione per un singolo cron."""
    cr = self.env.cr
    cr.execute("""
        SELECT
            date_trunc('hour', start_time) as hour,
            avg(duration) as avg_duration,
            max(duration) as max_duration,
            count(*) as executions,
            count(*) FILTER (WHERE state = 'error') as errors
        FROM system_observability_cron_log
        WHERE cron_id = %s
          AND start_time > now() - interval '%s days'
        GROUP BY date_trunc('hour', start_time)
        ORDER BY hour
    """, (cron_id, days))
    return cr.dictfetchall()
```

### 11.6 Lock Dependency Graph

Per casi complessi con molti lock in attesa, costruire un **grafo delle dipendenze** permette di identificare la "radice" del blocco:

```python
def _build_lock_graph(self):
    """Costruisce un grafo diretto delle dipendenze di lock.
    Nodo = PID, Arco = 'A e' bloccato da B'.
    Il nodo con in-degree 0 e out-degree > 0 e' la radice del blocco.
    """
    lock_waits = self._get_lock_waits()  # query vista prima
    graph = {}  # pid -> set of blocking_pids
    for lw in lock_waits:
        blocked = lw['blocked_pid']
        blocking = lw['blocking_pid']
        graph.setdefault(blocked, set()).add(blocking)

    # Trova le radici (PID che bloccano altri ma non sono bloccati)
    all_blocked = set(graph.keys())
    all_blocking = set()
    for pids in graph.values():
        all_blocking |= pids
    roots = all_blocking - all_blocked

    return {
        'graph': {k: list(v) for k, v in graph.items()},
        'roots': list(roots),
        'total_blocked': len(all_blocked),
    }
```

### 11.7 Vacuum e Autovacuum Health

Il vacuum e' fondamentale per le performance PostgreSQL. Monitorare non solo quando e' stato eseguito, ma quanto e' **efficace**:

```sql
-- Tabelle che necessitano VACUUM urgente
-- (dead tuples superano la soglia di autovacuum)
SELECT
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 1) as dead_pct,
    last_autovacuum,
    -- Soglia autovacuum: 50 + 0.2 * n_live_tup (default)
    (50 + 0.2 * n_live_tup)::bigint as autovacuum_threshold,
    n_dead_tup > (50 + 0.2 * n_live_tup) as needs_vacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > (50 + 0.2 * n_live_tup)
ORDER BY n_dead_tup DESC;
```

```sql
-- Progresso VACUUM in corso (utile per tabelle molto grandi)
SELECT
    p.pid,
    a.query,
    p.phase,
    p.heap_blks_total,
    p.heap_blks_scanned,
    p.heap_blks_vacuumed,
    round(100.0 * p.heap_blks_vacuumed / NULLIF(p.heap_blks_total, 0), 1) as pct_done
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON p.pid = a.pid;
```

### 11.8 Transaction ID (XID) Wraparound Monitoring

PostgreSQL usa transaction ID a 32 bit. Se non si esegue VACUUM in tempo, si rischia il **wraparound** che forza lo shutdown del database:

```sql
SELECT
    datname,
    age(datfrozenxid) as xid_age,
    -- Il limite e' 2 miliardi; alert a 500 milioni
    round(100.0 * age(datfrozenxid) / 2000000000, 1) as pct_to_wraparound,
    current_setting('autovacuum_freeze_max_age')::bigint as freeze_max_age
FROM pg_database
WHERE datname = current_database();
```

Questa e' una metrica critica che **deve** avere un alert configurato.

### 11.9 Buffer Cache Hit Ratio

```sql
-- Rapporto hit/miss della cache di PostgreSQL
-- Un valore < 99% su un DB production indica necessita' di piu' shared_buffers
SELECT
    sum(heap_blks_read) as blocks_read,
    sum(heap_blks_hit) as blocks_hit,
    CASE WHEN sum(heap_blks_read) + sum(heap_blks_hit) > 0
         THEN round(100.0 * sum(heap_blks_hit) /
              (sum(heap_blks_hit) + sum(heap_blks_read)), 2)
         ELSE 100 END as hit_ratio
FROM pg_statio_user_tables;
```

### 11.10 Checkpoint Analysis

I checkpoint sono operazioni pesanti che possono causare latenza:

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    -- Se checkpoints_req >> checkpoints_timed, WAL e' troppo piccolo
    CASE WHEN checkpoints_timed + checkpoints_req > 0
         THEN round(100.0 * checkpoints_req /
              (checkpoints_timed + checkpoints_req), 1)
         ELSE 0 END as pct_forced,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_backend,
    -- Se buffers_backend e' alto, i backend devono scrivere da soli
    -- (troppo lento tra checkpoint)
    CASE WHEN buffers_checkpoint + buffers_backend > 0
         THEN round(100.0 * buffers_backend /
              (buffers_checkpoint + buffers_backend), 1)
         ELSE 0 END as pct_backend_writes
FROM pg_stat_bgwriter;
```

---

## 12. Piano di Implementazione

### Fase 1: Scaffolding e Modelli Base
1. Creare la struttura del modulo (directory, `__init__.py`, `__manifest__.py`)
2. Implementare `system_observability.cron.log` con indici
3. Implementare `ir.cron` inherit con override di `_callback` e `_handle_callback_exception`
4. Testare: installare il modulo, eseguire un cron manualmente, verificare che il log venga creato

### Fase 2: Dashboard Backend
5. Implementare `system_observability.dashboard` (AbstractModel) con tutti i metodi `_get_*`
6. Implementare il controller JSON
7. Testare: chiamare l'endpoint via curl/browser e verificare il JSON restituito

### Fase 3: Dashboard Frontend
8. Creare i QWeb templates
9. Implementare `dashboard_main.js` (client action con auto-refresh)
10. Creare CSS
11. Registrare gli asset
12. Testare: navigare alla dashboard nel browser e verificare il rendering

### Fase 4: Sistema Alert
13. Implementare modelli `alert.rule` e `alert.log`
14. Implementare `_check_alerts()` e il cron associato
15. Aggiungere notifiche bus.bus
16. Testare: creare regole con soglie basse, forzare il cron, verificare alert e notifiche

### Fase 5: Viste e Menu
17. Creare tutte le viste XML (tree, form, search)
18. Creare menu e azioni
19. Implementare impostazioni (`res.config.settings`)
20. Testare: navigazione completa del modulo

### Fase 6: Sicurezza e Rifinitura
21. Configurare `ir.model.access.csv`
22. Verificare che utenti non-admin non possano accedere
23. Aggiungere parametri di default
24. Test completo end-to-end

---

## 13. Testing e Verifica

### Test Funzionali

| Test | Come Verificare |
|------|-----------------|
| Installazione modulo | `./odoo-bin -d testdb -i system_observability --stop-after-init` senza errori |
| Dashboard carica | Navigare a menu > System Observability > Dashboard, verificare dati reali |
| Auto-refresh | Lasciare dashboard aperta, osservare aggiornamento dopo N secondi |
| Log cron successo | Eseguire un cron manualmente (pulsante "Run Manually"), verificare log |
| Log cron errore | Creare cron con codice errato (`raise Exception('test')`), eseguire, verificare traceback nel log |
| Kill query | Aprire `psql` e fare `SELECT pg_sleep(120)`, dalla dashboard cliccare Kill |
| Alert warning | Creare regola "Active Connections > 1", forzare cron alert, verificare log alert |
| Alert critico + bus | Stessa cosa con soglia critica, verificare toast notification nella dashboard |
| Sicurezza | Provare ad accedere alla dashboard con un utente non-admin, deve essere negato |
| Retention log | Impostare retention a 0 giorni, forzare autovacuum, verificare pulizia |
| Code email | Creare email outgoing, verificare che appaia nel conteggio dashboard |

### Test di Performance

- La dashboard non deve impiegare piu' di 2 secondi a caricare
- Le query PostgreSQL sui cataloghi di sistema devono completarsi in < 500ms
- L'auto-refresh non deve causare accumulo di connessioni nel pool
- Il cron di check alert non deve durare piu' di 10 secondi

### Test di Robustezza

- Il modulo deve funzionare anche se non ci sono cron log (dashboard vuota ma funzionante)
- Il modulo SMS non e' installato: la sezione SMS deve essere assente (non errore)
- Il connection pool e' vuoto/pieno: deve mostrare i dati senza crash
- Le query pg_stat* devono funzionare anche con PostgreSQL 10+ (verificare compatibilita' colonne)
