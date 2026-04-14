# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Odoo 14.0 Community Edition — a modular Python ERP framework. The codebase has two layers: the core framework (`odoo/`) and ~380 installable addon modules (`addons/` and `odoo/addons/`).

## Running the Server

```bash
# Start the server (requires a PostgreSQL database)
./odoo-bin -d <dbname> --addons-path=addons,odoo/addons

# Common flags
./odoo-bin -d <dbname> -u <module>       # Update a module
./odoo-bin -d <dbname> -i <module>       # Install a module
./odoo-bin -d <dbname> --dev=all         # Dev mode (auto-reload on Python/XML changes)
```

The server listens on `http://localhost:8069` by default.

## Running Tests

Odoo uses its own test runner built on Python's `unittest`. Tests run during module installation/update:

```bash
# Run all tests for a module
./odoo-bin -d <testdb> -i <module> --test-enable --stop-after-init

# Run tests matching specific tags/classes/methods
./odoo-bin -d <testdb> -i <module> --test-tags :TestClassName.test_method --stop-after-init

# Run only post_install tests (after all modules loaded)
./odoo-bin -d <testdb> --test-tags post_install --stop-after-init
```

The `--test-tags` filter format is: `[-][tag][/module][:class][.method]`. Multiple specs are comma-separated. By default, tests tagged `standard` and `at_install` run during module install; `post_install` tests run after all modules are loaded.

## Architecture

### ORM Layer (`odoo/models.py`, `odoo/fields.py`, `odoo/api.py`)

The core of Odoo. Models are Python classes that map to PostgreSQL tables:

- **`models.Model`** — persistent database model (most common)
- **`models.TransientModel`** — temporary records, auto-vacuumed (used for wizards)
- **`models.AbstractModel`** — no database table, used as mixins

Key model attributes: `_name` (dot-separated, e.g. `account.move`), `_inherit` (extend existing model or mixin list), `_description`.

API decorators in `odoo/api.py`: `@api.depends()`, `@api.constrains()`, `@api.onchange()`, `@api.model`, `@api.returns()`.

The **Environment** (`self.env`) holds the cursor, user ID, and context. Access other models via `self.env['model.name']`. Records are **recordsets** — iterating, filtering, and mapping operate on sets of records.

### Addon/Module Structure

Each addon in `addons/` follows this layout:

```
addons/my_module/
  __manifest__.py    # Metadata: name, version, depends, data files
  __init__.py        # Python imports
  models/            # ORM model definitions
  views/             # XML: form, tree, kanban, search views and actions
  controllers/       # HTTP route handlers (inherit http.Controller)
  security/          # ir.model.access.csv, record rules
  data/              # XML/CSV seed data
  demo/              # Demo data (loaded with --with-demo)
  static/            # JS, CSS, SCSS, images, QWeb templates
  tests/             # Test classes (TransactionCase, SavepointCase, HttpCase)
  wizard/            # TransientModel definitions and views
  report/            # QWeb report templates
  i18n/              # Translation PO files
```

Module dependencies are declared in `__manifest__.py` under `'depends': [...]`. The framework resolves them via a dependency graph (`odoo/modules/graph.py`).

### Test Base Classes (`odoo/tests/common.py`)

- **`TransactionCase`** — each test method runs in its own rolled-back transaction
- **`SingleTransactionCase`** — all methods share one transaction, rolled back at the end
- **`SavepointCase`** — single transaction, each method gets a savepoint (fast, preferred for most tests)
- **`HttpCase`** — transactional + `url_open()` and Chrome headless browser helpers

Use `@tagged('post_install', '-at_install')` to control when tests execute. Test modules must be imported from the addon's `tests/__init__.py`.

### HTTP/Web Layer (`odoo/http.py`)

Werkzeug-based. Controllers use `@http.route()` decorators:
- `auth='user'` (logged in), `auth='public'` (guest ok), `auth='none'` (no session)
- `type='http'` (standard) or `type='json'` (JSON-RPC)

### Frontend (`addons/web/static/src/`)

The web client uses a custom JS module system (`boot.js`), QWeb XML templates, and SCSS. JS views (form, list, kanban) are in `addons/web/static/src/js/views/`. Field widgets are in `addons/web/static/src/js/fields/`.

### Module Registry (`odoo/modules/registry.py`)

One registry per database. It loads all installed modules, registers models, and caches them. Models are rebuilt on module update. Access via `odoo.registry(dbname)` or `self.env.registry`.

## Key Patterns

- **Inheritance**: `_inherit = 'existing.model'` (extend in-place) vs `_name = 'new.model'` + `_inherit = 'parent.model'` (prototype/delegation inheritance).
- **Computed fields**: `fields.Char(compute='_compute_x', store=True)` — `store=True` persists to DB and enables search/group-by.
- **XML IDs**: External identifiers like `module.xml_id` are used to reference records across modules. Use `self.env.ref('module.xml_id')` in Python.
- **Security**: Access rights in `security/ir.model.access.csv`, record rules in `security/*.xml`. Groups are defined as `<record model="res.groups">`.
- **Data files**: Listed in `__manifest__.py` under `'data'` (always loaded) or `'demo'` (demo mode only). Order matters — files load sequentially.
