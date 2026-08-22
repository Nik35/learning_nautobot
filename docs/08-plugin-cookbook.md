# Part 8: The Plugin Cookbook

Working, code-verified samples for every layer of a Nautobot app — with the DiffSync and
metadata patterns spelled out end to end. One running example throughout:
`nautobot_myipam`, syncing IP addresses from a system called **MyIPAM** into
`ipam.IPAddress`, alongside an Infoblox app that owns DNS on the same rows.

Verified 2026-08-22 against nautobot 3.2.4a0, nautobot-app-ssot 4.6.2a0, diffsync 2.2.3a0.

**Read Part 6 for the file layout and Part 7 for shipping.** This doc is the code.

---

## 8.0 Is it easy? An honest answer

It splits into three tiers, and only the first is genuinely easy.

| Tier | What | Difficulty | Why |
|---|---|---|---|
| **1. Scaffolding** | Models, views, tables, forms, filters, nav, REST API, GraphQL | **Easy** | Cookiecutter writes most of it; Nautobot's generic view classes do the rest. Mostly declarative. |
| **2. Sync** | DiffSync models, adapters, the SSoT job | **Moderate** | The contrib layer gives you free CRUD, but the field-name syntax and the load path have sharp edges (8.9, 8.12). |
| **3. Governance** | Field ownership, provenance, conflict avoidance | **Hard, and partly unsupported** | SSoT has no conflict resolution — last writer wins (Part 5). The framework gives primitives; the policy is yours to write (8.7). |

For your app specifically, tier 3 is the whole point. Budget accordingly: the CRUD app
is a week; making it safe to run next to the Infoblox app is the real work.

The single most useful thing to internalise: **a field you do not declare in
`_attributes` can never be written by your sync** (`diffsync/diff.py:266-279`, Part 2).
Ownership is enforced by what you leave out, not by configuration. That makes tier 3
tractable — but only if both apps agree to it.

---

## 8.1 The minimum viable app

Three files and one settings line. Everything else is optional.

### `nautobot_myipam/__init__.py`

The attribute names below are the real ones — `nautobot/extras/plugins/__init__.py:44-104`.

```python
"""App declaration for nautobot_myipam."""

from importlib import metadata

from nautobot.apps import NautobotAppConfig

__version__ = metadata.version(__name__)


class MyIPAMConfig(NautobotAppConfig):
    """App configuration for the nautobot_myipam app."""

    name = "nautobot_myipam"                 # must match the package directory
    verbose_name = "MyIPAM"
    version = __version__
    author = "Nikhil Mohite"
    description = "Custom IPAM data source for Nautobot."
    base_url = "myipam"                      # -> /plugins/myipam/
    required_settings = []                   # raises at startup if missing
    default_settings = {
        "default_status": "Active",
    }
    min_version = "3.2.0"                    # enforced: plugins/__init__.py:236-246
    max_version = "3.99"
    caching_config = {}

    # These are dotted paths resolved at load time. The defaults are shown; you only
    # override them to relocate a module. Full list: plugins/__init__.py:90-104
    # jobs = "jobs.jobs"
    # menu_items = "navigation.menu_items"
    # custom_validators = "custom_validators.custom_validators"
    # template_extensions = "template_content.template_extensions"
    # filter_extensions = "filter_extensions.filter_extensions"
    # table_extensions = "table_extensions.table_extensions"
    # secrets_providers = "secrets.secrets_providers"
    # homepage_layout = "homepage.layout"
    # graphql_types = "graphql.types.graphql_types"
    # banner_function = "banner.banner"


config = MyIPAMConfig  # <- Nautobot looks for the name `config`
```

### `nautobot_myipam/migrations/__init__.py`

Empty file. Create it on day one — without it Django will not treat the app as
migratable and the failure mode is obscure.

### The operator's `nautobot_config.py`

```python
PLUGINS = ["nautobot_myipam"]

PLUGINS_CONFIG = {
    "nautobot_myipam": {
        "default_status": "Active",
    },
    "nautobot_ssot": {
        # Required for provenance metadata. See 8.6 — this is easy to forget.
        "enable_metadata_for": ["MyIPAMDataSource"],
    },
}
```

There is **no entry-point autodiscovery**; `PLUGINS = []` is the default at
`nautobot/core/settings.py:213`. Installing the wheel alone does nothing (Part 7 §7.1).

---

## 8.2 A model

Rule from Part 6 §6.0: anything an operator changes at runtime, or that can exist more
than once, is a **model** — not a `PLUGINS_CONFIG` key. Multiple MyIPAM endpoints means
a model.

```python
# nautobot_myipam/models.py
"""Models for the MyIPAM app."""

from django.db import models
from nautobot.apps.models import PrimaryModel, extras_features


@extras_features("custom_links", "custom_validators", "graphql", "webhooks")
class MyIPAMConfigInstance(PrimaryModel):
    """Connection details for one MyIPAM endpoint."""

    name = models.CharField(max_length=255, unique=True)
    url = models.URLField()
    secrets_group = models.ForeignKey(
        to="extras.SecretsGroup",
        on_delete=models.PROTECT,
        related_name="myipam_configs",
        null=True,
        blank=True,
    )
    enabled = models.BooleanField(default=True)
    default_namespace = models.ForeignKey(
        to="ipam.Namespace",
        on_delete=models.PROTECT,
        related_name="+",
    )

    class Meta:
        ordering = ["name"]
        verbose_name = "MyIPAM Config"

    def __str__(self):
        return self.name
```

Two non-negotiables:

- **Inherit `PrimaryModel` or `OrganizationalModel`** (both wrap Nautobot's `BaseModel`).
  A plain `django.db.models.Model` lacks `validated_save()` and
  `associated_object_metadata`, and `NautobotModel` (8.4) will fail against it.
- **Never store the secret here.** `secrets_group` points at Nautobot's secrets machinery;
  values are fetched live and never hit your table (Part 4).

Then `nautobot-server makemigrations nautobot_myipam` and commit the result.

---

## 8.3 The boilerplate tier — nav, views, API

Nothing surprising here; this is the easy tier. Navigation, verified against
`nautobot-app-ssot/nautobot_ssot/navigation.py:1-32`:

```python
# nautobot_myipam/navigation.py
from nautobot.apps.ui import NavMenuAddButton, NavMenuGroup, NavMenuItem, NavMenuTab

menu_items = (
    NavMenuTab(
        name="Apps",
        groups=(
            NavMenuGroup(
                name="MyIPAM",
                weight=100,
                items=(
                    NavMenuItem(
                        link="plugins:nautobot_myipam:myipamconfiginstance_list",
                        name="Configs",
                        permissions=["nautobot_myipam.view_myipamconfiginstance"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:nautobot_myipam:myipamconfiginstance_add",
                                permissions=["nautobot_myipam.add_myipamconfiginstance"],
                            ),
                        ),
                    ),
                ),
            ),
        ),
    ),
)
```

Permission strings are always `<app_label>.<action>_<modelname>`, and `app_label` is your
package name — another reason the name is unchangeable after release (Part 6 §6.0.1).

Views use `NautobotUIViewSet` from `nautobot.apps.views`; the URL name pattern above
(`<modelname>_list`, `_add`, `_edit`) is what the router generates. That part is
genuinely mechanical — copy it from device-onboarding.

---

## 8.4 DiffSync: the mental model, then the code

### The pipeline

Two adapters, each loading a graph of `DiffSyncModel` objects into memory. `diff_to()`
compares them; `sync_to()` walks the resulting diff and calls `create` / `update` /
`delete` on the **target** side models.

```
MyIPAM API  ──load()──>  source adapter  ─┐
                                          ├── diff_to() ──> Diff ──> sync_to()
Nautobot ORM ──load()──>  target adapter ─┘                            │
                                                                       v
                                                          target model .create/.update/.delete
```

### The rule that matters most

Comparison runs over the **intersection** of both sides' `_attributes`
(`diffsync/diff.py:266-279`). Worked example:

```python
# source declares:  _attributes = ("description", "status", "dns_name")
# target declares:  _attributes = ("description", "status")
#
# compared:  description, status
# dns_name:  INVISIBLE. Never compared, never written. No warning, no error.
```

This is the whole basis of safe coexistence (8.5). It is also a silent failure mode when
unintentional — you will spend an afternoon on "why isn't my field syncing" before
noticing a typo in one `_attributes` tuple (8.12).

### The shared contract

Define identifiers and attributes **once**, in a base module both sides import. This is
the single most effective way to avoid accidental asymmetry.

```python
# nautobot_myipam/diffsync/models/base.py
"""Shared DiffSync contracts. Both adapters import from here — never redeclare."""

from typing import Optional
from uuid import UUID

from diffsync import DiffSyncModel


class IPAddressBase(DiffSyncModel):
    """Fields MyIPAM owns on ipam.IPAddress.

    NOTE: `dns_name` is deliberately absent. Infoblox owns it. Adding it here is
    the one change that would let this app silently overwrite their data.
    See docs/04 and 8.5.
    """

    _modelname = "ipaddress"
    _identifiers = ("host", "mask_length", "parent__namespace__name")
    _attributes = ("status__name", "description", "type", "tenant__name")

    host: str
    mask_length: int
    parent__namespace__name: str

    status__name: str
    description: Optional[str] = None
    type: str = "host"
    tenant__name: Optional[str] = None

    pk: Optional[UUID] = None  # not synced; used by contrib for ORM lookups
```

### The Nautobot (target) side — free CRUD

`NautobotModel` implements `create`, `update`, and `delete` generically by reflecting
over the ORM (`nautobot_ssot/contrib/model.py:29-124`). You usually write no methods.

```python
# nautobot_myipam/diffsync/models/nautobot.py
from nautobot.ipam.models import IPAddress
from nautobot_ssot.contrib import NautobotModel

from .base import IPAddressBase


class NautobotIPAddress(IPAddressBase, NautobotModel):
    """Nautobot-side IPAddress. Inherits create/update/delete from NautobotModel."""

    _model = IPAddress
```

### The remote (source) side

Only implement CRUD here if you sync *back* to MyIPAM. For a read-only source, the
model is data-only.

```python
# nautobot_myipam/diffsync/models/myipam.py
from .base import IPAddressBase


class MyIPAMIPAddress(IPAddressBase):
    """Source-side model. Read-only: no create/update/delete needed."""
```

### The adapters

```python
# nautobot_myipam/diffsync/adapters/nautobot.py
from nautobot_ssot.contrib import NautobotAdapter

from ..models.nautobot import NautobotIPAddress


class MyIPAMNautobotAdapter(NautobotAdapter):
    """Target adapter. `load()` is inherited and driven entirely by `top_level`."""

    top_level = ("ipaddress",)

    ipaddress = NautobotIPAddress
```

`NautobotAdapter.load()` walks `top_level`, resolves each name to the class attribute of
the same name, and loads every field in `get_synced_attributes()` from the ORM
(`contrib/adapter.py:197-212`). **Subclassing `NautobotAdapter` — not writing your own
adapter — is what unlocks the metadata features in 8.6** (`jobs/base.py:506-511`).

Scope the queryset when you should only manage a subset:

```python
class NautobotIPAddress(IPAddressBase, NautobotModel):
    _model = IPAddress

    @classmethod
    def get_queryset(cls):
        """Only manage addresses inside namespaces MyIPAM is configured for."""
        from nautobot_myipam.models import MyIPAMConfigInstance

        namespaces = MyIPAMConfigInstance.objects.filter(enabled=True).values_list(
            "default_namespace", flat=True
        )
        return IPAddress.objects.filter(parent__namespace__in=namespaces)
```

`get_queryset()` is the documented override point; `_get_queryset()` wraps it to add
`prefetch_related` for your foreign keys and metadata (`contrib/model.py:40-63`). Override
the public one.

The source adapter is ordinary code — call the API, build models, `self.add()`:

```python
# nautobot_myipam/diffsync/adapters/myipam.py
from diffsync import Adapter

from ..models.myipam import MyIPAMIPAddress


class MyIPAMRemoteAdapter(Adapter):
    top_level = ("ipaddress",)

    ipaddress = MyIPAMIPAddress

    def __init__(self, *args, job, sync=None, client, **kwargs):
        super().__init__(*args, **kwargs)
        self.job = job
        self.sync = sync
        self.client = client

    def load(self):
        for record in self.client.get_addresses():
            self.add(
                self.ipaddress(
                    host=record["address"],
                    mask_length=record["prefix_length"],
                    parent__namespace__name=record["view"],
                    status__name=record["state"].title(),
                    description=record.get("comment") or "",
                    type="host",
                    tenant__name=record.get("owner"),
                )
            )
            self.job.logger.debug("Loaded %s from MyIPAM", record["address"])
```

### The job

```python
# nautobot_myipam/jobs.py
from django.conf import settings
from nautobot.apps.jobs import ObjectVar, register_jobs
from nautobot.extras.jobs import JobException
from nautobot_ssot.jobs.base import DataMapping, DataSource

from nautobot_myipam.diffsync.adapters.myipam import MyIPAMRemoteAdapter
from nautobot_myipam.diffsync.adapters.nautobot import MyIPAMNautobotAdapter
from nautobot_myipam.models import MyIPAMConfigInstance
from nautobot_myipam.utils.client import MyIPAMClient

name = "MyIPAM SSoT"


class MyIPAMDataSource(DataSource):
    """Sync IP addresses from MyIPAM into Nautobot."""

    config = ObjectVar(
        model=MyIPAMConfigInstance,
        description="Which MyIPAM endpoint to sync from.",
    )

    class Meta:
        name = "MyIPAM to Nautobot"
        data_source = "MyIPAM"          # -> MetadataType "Last sync from MyIPAM" (8.6)
        description = "Sync IP address data from MyIPAM."

    @classmethod
    def data_mappings(cls):
        from django.urls import reverse

        return (
            DataMapping("Address", None, "IP Address", reverse("ipam:ipaddress_list")),
        )

    def validate_metadata_configuration(self):
        """Fail loudly if provenance metadata is not enabled. See 8.6."""
        enabled = settings.PLUGINS_CONFIG.get("nautobot_ssot", {}).get("enable_metadata_for", [])
        if self.__class__.__name__ not in enabled:
            self.logger.error(
                'Add `"enable_metadata_for": ["MyIPAMDataSource"]` to the nautobot_ssot '
                "PLUGINS_CONFIG. Without it no ownership metadata is recorded and this "
                "app cannot detect Infoblox-owned fields."
            )
            raise JobException(message="Metadata is not enabled for MyIPAMDataSource.")

    def load_source_adapter(self):
        self.validate_metadata_configuration()
        client = MyIPAMClient.from_config(self.config)
        self.source_adapter = MyIPAMRemoteAdapter(job=self, sync=self.sync, client=client)
        self.source_adapter.load()

    def load_target_adapter(self):
        self.target_adapter = MyIPAMNautobotAdapter(job=self, sync=self.sync)
        self.target_adapter.load()


jobs = [MyIPAMDataSource]
register_jobs(*jobs)
```

You implement only `load_source_adapter` and `load_target_adapter`. `calculate_diff` and
`execute_sync` are provided (`jobs/base.py:189-217`) and rarely need overriding. The
`validate_metadata_configuration` pattern is lifted from
`nautobot_ssot/integrations/cisco_sdwan/jobs.py:174-187` — a real integration that hard-fails
when metadata is off, for exactly your reason.

---

## 8.5 Field ownership by construction

The Infoblox integration already solves the IPAM/DNS overlap. Four DiffSync types,
**identical `_identifiers`**, **disjoint `_attributes`**, all over the same ORM row —
`nautobot_ssot/integrations/infoblox/diffsync/models/base.py`:

| Class | line | `_identifiers` | owns |
|---|---|---|---|
| `IPAddress` | 67 | `("address", "prefix", "prefix_length", "namespace")` | `description`, `status`, `ip_addr_type`, `ext_attrs`, `has_*_record`, `mac_address` |
| `DnsARecord` | 105 | *same* | `dns_name`, … |
| `DnsHostRecord` | 132 | *same* | `dns_name`, … |
| `DnsPTRRecord` | 159 | *same* | `dns_name`, … |

The load-bearing detail: **`IPAddress._attributes` (line 72-83) does not contain
`dns_name`.** The IPAM sync physically cannot touch DNS. Not a policy, not a check at
runtime — it is structurally impossible, because the diff never sees the field.

Write your split down before you write code:

| Field on `ipam.IPAddress` | MyIPAM | Infoblox | In your `_attributes`? |
|---|---|---|---|
| `host`, `mask_length`, `parent` | ✅ owner | — | yes (identifiers) |
| `status` | ✅ owner | — | yes |
| `description` | ✅ owner | — | yes |
| `tenant` | ✅ owner | — | yes |
| `dns_name` | — | ✅ owner | **no — omit deliberately** |
| `role` | ? | ? | **resolve before shipping** |

Put that table in your app's `docs/dev/` and treat additions to `_attributes` as a
review-gated change. A one-word addition to a tuple is enough to start silently
clobbering another team's data.

> This is safe **by construction only if both sides do it.** Infoblox omitting `dns_name`
> from its IPAM model protects DNS from *its own* IPAM sync — it does not stop your app.
> Your omission is what protects them. That is the open item in `CLAUDE.md`: ask whether
> they will declare ownership via `ObjectMetadata` so the guarantee is mutual and
> machine-checkable rather than a convention in two separate repos.

---

## 8.6 Metadata attributes — the provenance layer

This is the "add meta attributes to compare" piece, and it is **two different mechanisms**
that are easy to confuse. Get the distinction right and the rest is straightforward.

| | Mechanism A — sync provenance | Mechanism B — `ObjectMetadataAnnotation` |
|---|---|---|
| What it records | *which fields this job last wrote, and when* | *a value* stored in `ObjectMetadata` instead of a model field |
| Participates in the diff? | **No** — bookkeeping, written after each save | **Yes** — it is a normal entry in `_attributes` |
| You write code? | No — automatic | Yes — annotate the field |
| Backed by | `MetadataType` "Last sync from &lt;source&gt;" | a `MetadataType` you name |
| Use it for | "did Infoblox or MyIPAM write `dns_name`?" | "what is this object's ID in the remote system?" |

### Mechanism A — automatic provenance, and its two gates

When enabled, every `create`/`update` writes an `ObjectMetadata` row whose
`scoped_fields` lists exactly the fields this job syncs, with a timestamp value
(`contrib/model.py:443-457`):

```python
# contrib/model.py:444-457, abridged
@classmethod
def _update_obj_metadata(cls, obj, adapter):
    obj_metadata_scope_fields = adapter.metadata_scope_fields[cls]
    obj_metadata = obj.associated_object_metadata.filter(
        metadata_type=adapter.metadata_type
    ).first() or ObjectMetadata(metadata_type=adapter.metadata_type, assigned_object=obj)

    obj_metadata.scoped_fields = obj_metadata_scope_fields
    obj_metadata.value = datetime.now()
    obj_metadata.validated_save()
```

`scoped_fields` is derived from your `_attributes` automatically — you never maintain it
by hand (`contrib/adapter.py:379-383`):

```python
# contrib/adapter.py:379-383
obj_metadata_scope_fields = list(
    {parameter.split("__", maxsplit=1)[0] for parameter in diffsync_model.get_synced_attributes()}
)
self.metadata_scope_fields[diffsync_model] = obj_metadata_scope_fields
```

`get_synced_attributes()` is just `_identifiers + _attributes`
(`utils/diffsync.py:22-28`), and the `split("__")[0]` reduces `status__name` to `status`
— `scoped_fields` holds direct model field names only, matching the column's help text
at `nautobot/extras/models/metadata.py:256-262`.

So for the model in 8.4 you get, with no extra code:

```
ObjectMetadata(
    metadata_type = MetadataType(name="Last sync from MyIPAM", data_type="datetime"),
    assigned_object = <IPAddress: 10.0.0.5/32>,
    scoped_fields = ["host", "mask_length", "parent", "status", "description", "type", "tenant"],
    value = 2026-08-22T14:03:11,
)
```

Note `dns_name` is absent — the ledger reflects the ownership split for free.

**Both gates must be satisfied or you get nothing, silently** (`jobs/base.py:506-511`):

```python
# jobs/base.py:506-511
if self.__class__.__name__ in settings.PLUGINS_CONFIG.get("nautobot_ssot", {}).get(
    "enable_metadata_for", []
) and isinstance(self.target_adapter, NautobotAdapter):
    self.target_adapter.get_or_create_metadatatype()
```

1. Your **job class name** listed in `PLUGINS_CONFIG["nautobot_ssot"]["enable_metadata_for"]`.
2. Your target adapter must be an `isinstance` of `NautobotAdapter`. A hand-rolled adapter
   gets nothing, even if listed.

The `MetadataType` is created for you from `Meta.data_source`, and content types are
attached per model in `top_level` and its `_children` (`contrib/adapter.py:349-386`).
Hence `Meta.data_source = "MyIPAM"` → `MetadataType(name="Last sync from MyIPAM")`.

> **Trap (Part 3):** contrib creates `"Last sync from X"` while some integrations create
> `"Last Sync from X"`. Different capitalisation, **different rows**. Always match
> case-insensitively when you read them (8.7).

### Mechanism B — a field whose value lives in metadata

Use `ObjectMetadataAnnotation` when the remote system has data with nowhere to live on the
Nautobot model — a remote record ID, a grid name, a last-modified stamp — and you want it
**compared and synced like any other attribute** (`contrib/types.py:90-113`).

```python
# nautobot_myipam/diffsync/models/base.py
from typing import Annotated, Optional

from diffsync import DiffSyncModel
from nautobot_ssot.contrib.types import ObjectMetadataAnnotation


class IPAddressBase(DiffSyncModel):
    _modelname = "ipaddress"
    _identifiers = ("host", "mask_length", "parent__namespace__name")
    _attributes = (
        "status__name",
        "description",
        "type",
        "tenant__name",
        "myipam_record_id",   # <- compared and synced like anything else
    )

    host: str
    mask_length: int
    parent__namespace__name: str
    status__name: str
    description: Optional[str] = None
    type: str = "host"
    tenant__name: Optional[str] = None

    myipam_record_id: Annotated[
        Optional[str], ObjectMetadataAnnotation(metadata_type_name="MyIPAM Record ID")
    ] = None
```

Read path: `contrib/adapter.py:105-108` → `_get_object_metadata_value`, matching on
`metadata.metadata_type.name` (adapter.py:149-158). Write path: stashed during
`_handle_single_field` and written **after** save, since it needs a pk
(`contrib/model.py:144-147, 264-266, 412-441`).

**The annotation does not create the `MetadataType`.** It raises if it is missing
(`model.py:423-427`). Create it in a `post_migrate` signal (8.10).

> **Trap (Part 3, confirmed at `contrib/model.py:435`):** `_set_object_metadata_fields`
> hardcodes `obj_metadata.scoped_fields = []`. Mechanism-B rows are **whole-object**
> scoped, always. Field scoping works for Mechanism A only. If you need a Mechanism-B
> value scoped to specific fields, you must write the `ObjectMetadata` row yourself.

### Which do you want?

For your conflict problem: **Mechanism A**, plus the guard in 8.7. It gives you a
per-field ledger with zero code, and it is the same ledger the Infoblox app would populate
if they enable it — which is precisely why the open question in `CLAUDE.md` is worth
asking before either app ships.

---

## 8.7 Using metadata to decide whether you may write

SSoT has no conflict resolution — last writer wins (Part 5). Below is the policy layer the
framework does not supply. Two independent lines of defence; use both.

### Defence 1 — a readable ownership query

```python
# nautobot_myipam/utils/ownership.py
"""Read the ObjectMetadata provenance ledger. See docs/08 §8.6."""

from nautobot.extras.models.metadata import ObjectMetadata

MY_SOURCE = "MyIPAM"


def field_owners(obj, field_name):
    """Return the set of source names whose sync claims `field_name` on `obj`.

    Matches MetadataType names case-insensitively: contrib writes "Last sync from X"
    but some integrations write "Last Sync from X" — different rows. See docs/03.
    """
    owners = set()
    for metadata in obj.associated_object_metadata.select_related("metadata_type"):
        name = metadata.metadata_type.name
        if not name.lower().startswith("last sync from "):
            continue
        # scoped_fields == [] means whole-object scope, i.e. it claims everything.
        if not metadata.scoped_fields or field_name in metadata.scoped_fields:
            owners.add(name[len("last sync from "):])
    return owners


def owned_by_other(obj, field_name, me=MY_SOURCE):
    """True if some *other* sync source claims this field."""
    return bool(field_owners(obj, field_name) - {me})
```

Note the `scoped_fields == []` branch: an empty list is *whole-object* scope, so it must
be read as "claims everything", not "claims nothing". Getting this backwards turns the
guard into a no-op — and empty lists are exactly what Mechanism B writes (8.6).

### Defence 2 — refuse the write

Override `update()` on the Nautobot-side model. This is the correct seam: it is called for
every changed field on every sync.

```python
# nautobot_myipam/diffsync/models/nautobot.py
from nautobot.ipam.models import IPAddress
from nautobot_ssot.contrib import NautobotModel

from nautobot_myipam.utils.ownership import MY_SOURCE, field_owners

from .base import IPAddressBase


class NautobotIPAddress(IPAddressBase, NautobotModel):
    _model = IPAddress

    def update(self, attrs):
        """Drop any attribute another source owns, and say so loudly."""
        obj = self.get_from_db()
        allowed = {}
        refused = {}  # field name -> set of owning sources

        for key, value in attrs.items():
            field = key.split("__", maxsplit=1)[0]   # status__name -> status
            others = field_owners(obj, field) - {MY_SOURCE}
            if others:
                refused[field] = others
            else:
                allowed[key] = value

        for field, owners in sorted(refused.items()):
            self.adapter.job.logger.warning(
                "Refusing to write `%s` on %s — owned by %s.",
                field,
                obj,
                ", ".join(sorted(owners)),
            )

        if not allowed:
            return self
        return super().update(allowed)
```

### Defence 3 — a hard invariant that survives everything

Custom validators run inside `validated_save()` for *every* writer — jobs, the UI, the
REST API, another app. This is the only check that cannot be bypassed by code you do not
control (`nautobot/extras/plugins/__init__.py:783-800`).

```python
# nautobot_myipam/custom_validators.py
"""Invariants enforced on every save, from any code path."""

from nautobot.apps.models import CustomValidator

from nautobot_myipam.utils.ownership import owned_by_other


class IPAddressOwnershipValidator(CustomValidator):
    """Prevent a MyIPAM-owned description being clobbered by another sync."""

    model = "ipam.ipaddress"

    def clean(self):
        obj = self.context["obj"]
        if obj.present_in_database and owned_by_other(obj, "description", me="MyIPAM"):
            original = self.context["obj"].__class__.objects.get(pk=obj.pk)
            if original.description != obj.description:
                self.validation_error(
                    {"description": "This field is owned by MyIPAM and cannot be modified here."}
                )


custom_validators = [IPAddressOwnershipValidator]
```

Use this sparingly — it fires on every save of every IP address in the system, so keep the
query count low and the condition narrow. A validator that raises on a legitimate operator
edit is worse than no validator. Ship it in *warn* mode (log, don't raise) first, watch the
logs for a release, then tighten.

---

## 8.8 Annotation reference

### Custom fields

`contrib/types.py:47-88`. Maps a DiffSync attribute to a Nautobot custom field **key**:

```python
from typing import Annotated
from nautobot_ssot.contrib.types import CustomFieldAnnotation


class NautobotIPAddress(NautobotModel):
    _model = IPAddress
    _identifiers = ("host",)
    _attributes = ("myipam_zone",)

    host: str
    myipam_zone: Annotated[str, CustomFieldAnnotation(key="myipam_zone")]
```

Read at `contrib/adapter.py:98-103` from `database_object.cf[key]`. Use `key`, not `name`
— `name` is a backwards-compatibility alias slated for removal (types.py:74-76).

### Custom relationships

`contrib/types.py:17-44`:

```python
from nautobot_ssot.contrib.types import CustomRelationshipAnnotation, RelationshipSideEnum

tenant__name: Annotated[
    str,
    CustomRelationshipAnnotation(name="IP to Tenant", side=RelationshipSideEnum.SOURCE),
]
```

### Many-to-many

Use the prebuilt TypedDicts from `nautobot_ssot/contrib/typeddicts.py` (`TagDict`,
`LocationDict`, `PrefixDict`, `VLANDict`, `InterfaceDict`, `ContentTypeDict`, …):

```python
from nautobot_ssot.contrib.typeddicts import TagDict

tags: list[TagDict] = []
```

The `SortKey` annotation inside those dicts (typeddicts.py:8-9) exists so lists can be
sorted into a stable order before comparison — otherwise ordering differences read as
diffs. **But note:** `sort_relationships` is commented out in the main job flow
(`jobs/base.py:513-515`, "Disabled for the time being due to ongoing issues"). Sort your
M2M lists yourself in `load()` until that is fixed.

---

## 8.9 Field-name syntax cheat sheet

The load dispatcher is `contrib/adapter.py:95-146`; branches in order:

| You write | Resolves to | Handled at |
|---|---|---|
| `description` | plain field, `getattr(obj, "description")` | adapter.py:143-146 |
| `status__name` | forward FK traversal | adapter.py:115-122 |
| `parent__namespace__name` | multi-hop FK traversal | adapter.py:115-122 |
| `tags` (list[TagDict]) | many-to-many / one-to-many | adapter.py:132-138 |
| `Annotated[..., CustomFieldAnnotation(key=…)]` | `obj.cf[key]` | adapter.py:98-103 |
| `Annotated[..., CustomRelationshipAnnotation(…)]` | relationship association | adapter.py:116-130 |
| `Annotated[..., ObjectMetadataAnnotation(…)]` | `ObjectMetadata` value | adapter.py:105-108 |

Escape hatch for anything else — define `load_param_<name>` on the **adapter** and it wins
over the default branch (adapter.py:142-145):

```python
class MyIPAMNautobotAdapter(NautobotAdapter):
    top_level = ("ipaddress",)
    ipaddress = NautobotIPAddress

    def load_param_description(self, parameter_name, database_object):
        """Normalise whitespace so cosmetic differences do not read as diffs."""
        return " ".join((database_object.description or "").split())
```

That hook is the cleanest fix for the most common false-diff class: values that differ
only in formatting between the two systems.

---

## 8.10 Signals — creating the MetadataType

Mechanism B requires the `MetadataType` to exist before the first sync (8.6). Create it at
`post_migrate`, alongside any Statuses or Roles your app assumes.

```python
# nautobot_myipam/signals.py
"""Post-migrate hooks: create the objects this app assumes exist."""

from nautobot.core.choices import ColorChoices


def post_migrate_create_objects(sender, apps=None, **kwargs):
    """Create the MetadataType and Status rows the app depends on."""
    from django.contrib.contenttypes.models import ContentType
    from nautobot.extras.choices import MetadataTypeDataTypeChoices
    from nautobot.extras.models import Status
    from nautobot.extras.models.metadata import MetadataType
    from nautobot.ipam.models import IPAddress

    metadata_type, _ = MetadataType.objects.get_or_create(
        name="MyIPAM Record ID",
        defaults={
            "data_type": MetadataTypeDataTypeChoices.TYPE_TEXT,
            "description": "Primary key of this address in MyIPAM.",
        },
    )
    content_type = ContentType.objects.get_for_model(IPAddress)
    if content_type not in metadata_type.content_types.all():
        metadata_type.content_types.add(content_type)

    status, _ = Status.objects.get_or_create(
        name="MyIPAM Reserved",
        defaults={"color": ColorChoices.COLOR_AMBER},
    )
    status.content_types.add(content_type)
```

Wire it up in `MyIPAMConfig.ready()`:

```python
def ready(self):
    """Connect post_migrate signal handlers."""
    from nautobot_myipam.signals import post_migrate_create_objects

    super().ready()
    post_migrate.connect(post_migrate_create_objects, sender=self)
```

Note the shape mirrors what SSoT does for Mechanism A at `contrib/adapter.py:354-378`:
`get_or_create` the type, then attach the content type if absent. Both steps are required
— a `MetadataType` without your model's content type will fail validation on save.

---

## 8.11 Showing provenance in the UI

Requirement from `CLAUDE.md`: provenance must be **visible on the UI**. A
`TemplateExtension` adds a panel to the core IPAddress detail page
(`nautobot/extras/plugins/__init__.py:291-308`).

```python
# nautobot_myipam/template_content.py
"""Inject a field-ownership panel into the core IPAddress detail view."""

from nautobot.apps.ui import TemplateExtension


class IPAddressOwnershipPanel(TemplateExtension):
    """Show which sync source last wrote each field."""

    model = "ipam.ipaddress"

    def right_page(self):
        obj = self.context["object"]
        rows = []
        for metadata in obj.associated_object_metadata.select_related("metadata_type"):
            name = metadata.metadata_type.name
            if not name.lower().startswith("last sync from "):
                continue
            rows.append(
                {
                    "source": name[len("last sync from "):],
                    "fields": metadata.scoped_fields or ["(whole object)"],
                    "when": metadata.value,
                }
            )
        return self.render(
            "nautobot_myipam/inc/ownership_panel.html",
            extra_context={"rows": rows},
        )


template_extensions = [IPAddressOwnershipPanel]
```

The template goes at
`nautobot_myipam/templates/nautobot_myipam/inc/ownership_panel.html` — **namespaced under
your package name**. Django flattens template directories across all installed apps, so a
bare `templates/inc/panel.html` can silently shadow core's (Part 6 §6.1).

---

## 8.12 Debugging: symptom → cause

| Symptom | Cause | Where |
|---|---|---|
| Field never syncs, no error | Missing from `_attributes` on one side; diff uses the intersection | `diffsync/diff.py:266-279` |
| App not loaded at all | Not in `PLUGINS`; there is no autodiscovery | `core/settings.py:213` |
| `PackageNotFoundError` on import | Source on `PYTHONPATH` but not pip-installed | Part 7 §7.2 |
| App refuses to load with a version message | `min_version`/`max_version` mismatch — operator cannot override | `extras/plugins/__init__.py:236-246` |
| No `ObjectMetadata` rows appear | Job name missing from `enable_metadata_for`, **or** adapter is not a `NautobotAdapter` subclass | `jobs/base.py:506-511` |
| Ownership check never triggers | Reading `scoped_fields == []` as "claims nothing"; it means whole-object scope | 8.7 |
| Two "Last sync from" MetadataTypes | Capitalisation mismatch between contrib and integrations | Part 3 |
| Mechanism-B metadata has empty `scoped_fields` | Hardcoded — not a bug you can configure away | `contrib/model.py:435` |
| `No such MetadataType '<name>'` on sync | `ObjectMetadataAnnotation` does not create the type | `contrib/model.py:423-427` |
| M2M list shows a diff every run | Ordering; `sort_relationships` is disabled in the job flow | `jobs/base.py:513-515` |
| Job runs in UI, fails in worker | App not installed in the worker image | Part 7 §7.1 |
| Migrations not detected | Missing `migrations/__init__.py` | Part 6 §6.1 |
| Your template replaced a core page | `templates/` not namespaced under the package name | Part 6 §6.1 |
| `validated_save()` missing on your model | Model does not inherit `PrimaryModel`/`OrganizationalModel` | 8.2 |

---

## 8.13 Build order

1. `__init__.py` + `migrations/__init__.py` + `PLUGINS` → confirm the app loads.
2. Config model + migration → confirm it appears in the UI.
3. **Write the ownership table (8.5) before any DiffSync code.**
4. `diffsync/models/base.py` — the shared contract, both sides importing it.
5. Nautobot-side models + `NautobotAdapter` subclass → run with `dryrun=True`.
6. Source adapter + client.
7. Job, with `validate_metadata_configuration`. Enable `enable_metadata_for`. Confirm
   `ObjectMetadata` rows appear with the `scoped_fields` you expect.
8. The ownership guard (8.7) — **in warn mode first**.
9. The UI panel (8.11).
10. Custom validator last, once the warnings have been quiet for a release.

Steps 1-2 and 4-7 are the easy tier. Step 3 is a conversation with the Nautobot team, and
steps 8-10 are the part no framework will write for you.
