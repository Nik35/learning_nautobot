# Nautobot Plugin Research — Part 1: Repo Map & Config Override

> Research target: building a custom IPAM app that must coexist with a DNS/Infoblox SSoT app
> without fighting over the same `ipam.IPAddress` rows.
> All line references verified against the code cloned below on 2026-08-22.

## 1.1 What is actually checked out

| Path | Version | Why it matters |
|---|---|---|
| `../nautobot` | 3.2.4a0 | Core. Models, metadata, secrets, app framework. |
| `../nautobot-app-ssot` | 4.6.2a0 | SSoT framework + **all** integrations incl. Infoblox. |
| `../diffsync` | 2.2.3a0 | The actual diff/sync engine. |
| `../nautobot-app-secrets-providers` | — | Vault / AWS / Azure / Delinea / 1Password. |
| `../nautobot-app-device-onboarding` | — | Second worked example of the SSoT contrib layer. |

### The single most important structural fact

**Nautobot core contains zero diff/sync logic.** A full-tree grep for `diffsync` in
`../nautobot` returns nothing. The only `ssot` hits are an unrelated string in
`nautobot/core/settings.py` and `nautobot/dcim/component_creation.py`.

So the layering is:

```
diffsync (PyPI, generic, no Django)
   |  DiffSyncModel, Adapter, DiffSyncDiffer, DiffSyncSyncer
   v
nautobot_ssot.contrib (Django/ORM-aware generic layer)
   |  NautobotModel, NautobotAdapter  <-- "the generic one for each database model"
   v
nautobot_ssot.integrations.<vendor> (Infoblox, ACI, Meraki, ...)
   |  concrete adapters + models + Job + per-instance config model
   v
nautobot core ORM (ipam.IPAddress, extras.ObjectMetadata, ...)
```

The "generic model per database model" you had in mind is
`nautobot_ssot/contrib/model.py::NautobotModel` — one class that reflects over
`_model._meta` and drives create/update/delete for *any* Nautobot ORM model.
It is **not** in core, and it is **not** auto-generated per model; you subclass it
and declare which fields participate.

## 1.2 Config override — how it actually resolves

Three distinct layers, in increasing precedence:

**1. `nautobot/core/settings.py`** — the shipped defaults. Note `PLUGINS_CONFIG = {}`
at `nautobot/core/settings.py:214`. Most settings are already env-var aware via
`is_truthy(os.getenv(...))` (`nautobot/core/settings_funcs.py`), e.g. line 92.

**2. `nautobot_config.py`** — your file, located by `NAUTOBOT_CONFIG` env var.
It is imported *after* core settings, so a bare assignment replaces the core value
outright. The comment at `nautobot/core/settings.py:1364` is worth reading: if you
override `LOGGING` at all, the env-var-driven post-processing step is **skipped
entirely**. The same "replace, don't merge" hazard applies to any dict/list setting
you assign wholesale — `PLUGINS_CONFIG`, `CACHES`, `TEMPLATES`, `CONSTANCE_CONFIG`.
If you are extending rather than replacing, append to the imported value instead of
rebinding the name.

**3. `NautobotAppConfig.default_settings`** — per-app defaults, merged into
`PLUGINS_CONFIG[<app_label>]` at app-load time.
`nautobot/extras/plugins/__init__.py:45-105`:

```python
class NautobotAppConfig(NautobotConfig):
    default = True
    author = ""; author_email = ""; description = ""; version = ""
    base_url = None                 # mounts under /plugins/<base_url>
    min_version = None              # enforced compatibility window
    max_version = None
    default_settings = {}           # merged into PLUGINS_CONFIG
    required_settings = []          # startup fails loudly if absent
    middleware = []
    installed_apps = []             # your app can pull in extra Django apps
    constance_config = {}
    provides_dynamic_jobs = False
```

`required_settings` is the one to lean on: it turns a missing config key into a
startup error instead of an `AttributeError` at 3am.

### The extension points your app declares

Same file, lines 89-105. These are dotted paths *relative to your app package*,
each resolved at load time. Override the attribute to relocate the module.

| Attribute | Default path | Use for your IPAM app |
|---|---|---|
| `custom_validators` | `custom_validators.custom_validators` | **Conflict prevention at write time** — see Part 3 |
| `secrets_providers` | `secrets.secrets_providers` | Register your own secret backends |
| `jobs` | `jobs.jobs` | The SSoT job itself |
| `table_extensions` | `table_extensions.table_extensions` | **Add a provenance column to the IPAddress list view** |
| `template_extensions` | `template_content.template_extensions` | **Inject a "who owns this field" panel on the IPAddress detail page** |
| `filter_extensions` | `filter_extensions.filter_extensions` | Filter IPs by owning data source |
| `override_views` | `views.override_views` | Wholesale replace a core view |
| `graphql_types` | `graphql.types.graphql_types` | |
| `datasource_contents` | `datasources.datasource_contents` | Git-backed config |
| `banner_function` | `banner.banner` | |
| `homepage_layout` | `homepage.layout` | |
| `menu_items` | `navigation.menu_items` | |
| `metrics` | `metrics.metrics` | Prometheus counters for sync drift |
| `jinja_filters` | `jinja_filters` | |

The four bolded rows are how you satisfy your "show details on UI" requirement
without forking core templates.

### The public API surface

Do **not** import from `nautobot.extras.*` / `nautobot.ipam.*` directly in app code.
Nautobot publishes a stable facade in `nautobot/apps/`:

```
admin.py  api.py  change_logging.py  choices.py  config.py  constants.py
datasources.py  dcim.py  events.py  exceptions.py  factory.py  filters.py
forms.py  graphql.py  jobs.py  models.py  querysets.py  secrets.py
tables.py  templatetags.py  testing.py  ui.py  urls.py  utils.py  views.py
```

Anything reachable from `nautobot.apps.*` carries a deprecation policy. Anything
else can move between minor releases. `nautobot.apps.secrets` and
`nautobot.apps.models` are the two you will use most.
