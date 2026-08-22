# Part 3: The Generic ORM Layer, Metadata, and Conflict Precedence

This is the part that answers your actual question. Two sections: how the generic
Nautobot model layer works, then how ownership/provenance is recorded and how you
build precedence logic on top of it.

## 3.1 `NautobotModel` — the generic per-ORM-model class

`../nautobot-app-ssot/nautobot_ssot/contrib/model.py:28`

```python
class NautobotModel(DiffSyncModel, DiffSyncModelUtilityMixin, BaseNautobotModel):
    _model: ClassVar[Model]   # <-- the ONLY thing you must set
```

You declare `_model = ipam.IPAddress` plus `_identifiers`/`_attributes`, and CRUD is
handled generically by reflecting over `_model._meta`. That is the "out of the box for
each database model" behaviour you were describing. It is one generic class, not a
generated class per model.

### Field-name syntax is the whole API

The magic lives in `_handle_single_field` (`model.py:126-222`). A field name in
`_attributes` is parsed by its shape:

| Syntax | Means | Handling |
|---|---|---|
| `name` | plain local field | direct `setattr` |
| `tenant__name` | FK traversal — resolve `Tenant` by `name` | `model.py:171-174` |
| `parent__location__name` | multi-hop FK | recursive lookup |
| `tenant__app_label` + `tenant__model` | generic FK via ContentType | `model.py:379-392` |
| `Annotated[..., CustomFieldAnnotation(key=...)]` | Nautobot custom field | `contrib/types.py` |
| `Annotated[..., CustomRelationshipAnnotation(name=..., side=...)]` | custom relationship | `model.py:328-362` |
| `List[TypedDict]` | to-many / M2M | `model.py:311-327` |
| `Annotated[..., ObjectMetadataAnnotation(...)]` | **backed by ObjectMetadata** | `model.py:412-442` |

Two-underscore separation is a genuine constraint: a Django field whose own name
contains `__` cannot be expressed. Also note `_get_queryset` (`model.py:40-58`) builds
`select_related`/`prefetch_related` from the declared parameters — declare your fields
properly and the N+1 problem is solved for you; declare them lazily and it is not.

### Caching

`NautobotAdapter.get_from_orm_cache(parameters, model_class)` (`adapter.py:72`) wraps
`ORMCache` (`nautobot_ssot/utils/cache.py:25`), keyed by
`frozenset(parameters.items())` under `"{app_label}.{model}"`. Every FK resolution
goes through it. `invalidate_cache()` exists and `cache_hits` is tracked per key —
useful when profiling a slow sync.

**Cache staleness is a real hazard**: the cache is populated during `load()` and lives
across the whole sync. If your `create()` makes an object another model then looks up
by the same parameters, you can get a stale `DoesNotExist`. Call `invalidate_cache()`
between phases when you create objects mid-sync.

### `get_synced_attributes`

`nautobot_ssot/utils/diffsync.py:22`:

```python
@classmethod
def get_synced_attributes(cls) -> list[str]:
    return list(cls._identifiers) + list(cls._attributes)
```

Identifiers **plus** attributes. This is the definition of "the fields this model
touches", and it is what feeds the provenance record below. Remember from Part 2 that
only `_attributes` are *diffed* — but identifiers are still *written* on create, so
counting both here is correct.

---

## 3.2 Metadata: the provenance system

Nautobot 2.2+ added a first-class metadata framework in core. This is **not** custom
fields and **not** tags — it is purpose-built for exactly your problem.

`../nautobot/nautobot/extras/models/metadata.py`

```python
class MetadataType(PrimaryModel):                 # :60
    content_types = M2M(ContentType)              # :61  which models it can apply to
    name          = CharField(unique=True)        # :67
    data_type     = CharField(choices=...)        # :69  text/int/float/bool/date/
                                                  #      datetime/url/json/markdown/
                                                  #      select/multiselect/contact-team
    # data_type is IMMUTABLE after creation (:93)

class MetadataChoice(ChangeLoggedModel, BaseModel):  # :165
    metadata_type, value, weight                     # only for select/multiselect

class ObjectMetadata(ChangeLoggedModel, BaseModel):  # :236
    metadata_type       = FK(MetadataType)           # :237
    contact / team      = FK(...)                    # :242,:249
    scoped_fields       = JSONArrayField()           # :256  <-- THE KEY FIELD
    _value              = JSONField()                # :263
    assigned_object_type / assigned_object_id        # :268  generic FK to any object
```

### Why `scoped_fields` changes everything

`ObjectMetadata` does not just say "this object came from system X". It says
**"this metadata applies to *these specific fields* of this object."** That is a
per-field provenance ledger attached to any Nautobot object, stored in core, with
change logging, REST API, and GraphQL already wired up.

For your problem:

```
IPAddress 10.1.1.5/24
  |- ObjectMetadata(type="Last sync from MyIPAM",  scoped_fields=["address","status","tenant","role"])
  |- ObjectMetadata(type="Last sync from Infoblox", scoped_fields=["dns_name","description"])
```

Now "who owns `dns_name` on this IP" is a database query, not a convention.

### How SSoT populates it

Two mechanisms — and **they are not the same code path**, which is a trap.

**(a) Automatic last-sync stamping** — `contrib/adapter.py:349-386`:

```python
def get_or_create_metadatatype(self):
    metadata_type__name = self.job.__class__.Meta.data_source
    metadata_type, _ = MetadataType.objects.get_or_create(
        name=f"Last sync from {metadata_type__name}",
        defaults={"data_type": "datetime", ...},
    )
    for diffsync_model in diffsync_models:          # walks top_level + all _children
        content_type = ContentType.objects.get_for_model(diffsync_model._model)
        metadata_type.content_types.add(content_type)
        obj_metadata_scope_fields = list(
            {p.split("__", maxsplit=1)[0] for p in diffsync_model.get_synced_attributes()}
        )
        self.metadata_scope_fields[diffsync_model] = obj_metadata_scope_fields
```

Read that carefully. **`scoped_fields` is derived automatically from
`get_synced_attributes()`** — the first segment of every declared identifier and
attribute. Declaring `tenant__name` scopes the field `tenant`. So the ownership
ledger is a direct, automatic consequence of your `_attributes` declaration. You do
not maintain it by hand.

Then on every create and update (`model.py:84` and `model.py:119`),
`_update_obj_metadata` (`model.py:444-457`) writes the row:

```python
obj_metadata.scoped_fields = obj_metadata_scope_fields
obj_metadata.value = datetime.now()
obj_metadata.validated_save()
```

**(b) Explicit per-field metadata** — `ObjectMetadataAnnotation` (`contrib/types.py`).
Lets a DiffSync field read/write an `ObjectMetadata` row instead of a real model
field:

```python
external_id: Annotated[Optional[str], ObjectMetadataAnnotation(
    metadata_type_name="External System ID")] = None
```

Written by `_set_object_metadata_fields` (`model.py:412-442`). Note it sets
`scoped_fields = []` (whole-object scope) and **requires the `MetadataType` to already
exist** — it raises `ObjectCrudException` rather than creating it. Your app must
create the type in a migration or `post_migrate` signal.

### It is opt-in, and you will forget this

`nautobot_ssot/jobs/base.py:507-510`:

```python
if self.__class__.__name__ in settings.PLUGINS_CONFIG.get("nautobot_ssot", {}).get(
    "enable_metadata_for", []
) and isinstance(self.target_adapter, NautobotAdapter):
    self.target_adapter.get_or_create_metadatatype()
```

Two conditions, both required:
1. Your **job class name** listed in `PLUGINS_CONFIG["nautobot_ssot"]["enable_metadata_for"]`
2. Target adapter is a `NautobotAdapter` subclass (i.e. you use the contrib layer)

If you hand-roll your adapter off raw `diffsync.Adapter`, you get **no metadata at
all**. That alone is a strong argument for building on `contrib`.

### Trap: two implementations, inconsistent naming

There is a second, parallel implementation at
`nautobot_ssot/integrations/metadata_utils.py`. Compare:

| Source | MetadataType name |
|---|---|
| `contrib/adapter.py:355` | `f"Last sync from {...}"` — lowercase **s** |
| `integrations/metadata_utils.py:50` | `f"Last Sync from {...}"` — capital **S** |
| `integrations/metadata_utils.py:28` (`object_has_metadata`) | `f"Last Sync from {...}"` — capital **S** |

These produce **different `MetadataType` rows**. A contrib-created type will not be
found by `object_has_metadata()`. If you query for provenance, do not hardcode either
string — match case-insensitively, or better, resolve the `MetadataType` once at
startup and store the PK.

Also note `metadata_utils.add_or_update_metadata_on_object` takes `scoped_fields` as
an explicit `dict[str, list[str]]` keyed by `"app_label.model"`, whereas contrib
derives it automatically. Two different contracts for the same concept.

---

## 3.3 The precedence toolkit, ranked

Everything Nautobot gives you for deciding "which source wins", best first:

**1. Disjoint `_attributes` across separate DiffSync model types (strongest).**
Costs nothing at runtime, cannot be misconfigured, enforced by `get_attrs_keys()`
intersection. This is what Infoblox does — see Part 4. If your IPAM app never declares
`dns_name` in `_attributes`, it can never write it. Full stop.

**2. `ObjectMetadata.scoped_fields` (the ledger).**
Query it during `load()` to decide flags. This is the mechanism for *dynamic*
precedence — "Infoblox owns `dns_name` unless it hasn't synced in 7 days".

**3. `DiffSyncModelFlags` set per instance during `load()`.**
`IGNORE` / `SKIP_UNMATCHED_DST` on individual objects, driven by (2).

**4. `CustomValidator` (`nautobot/extras/plugins/__init__.py:783`).**
Registered via the `custom_validators` app attribute. Its `clean()` runs on **every**
save of the target model, from any code path — SSoT job, REST API, UI form, ORM
script. This is your last line of defence: a hard invariant that no app can bypass.
Context includes `obj` and `user` (`CustomValidatorContext`, line 719; note `object`
is deprecated in favour of `obj`, removal in 4.0).

**5. Tags / custom fields (weakest — do not lead with these).**
Tags are a flat M2M with no field scoping. Custom fields add a column but carry no
provenance semantics. Both work, neither expresses "which field". Nautobot's own
docs suggest them because they predate the metadata framework. Use metadata instead.

### Where the "custom logic" actually goes

Concretely, three integration points:

```python
# A. Load-time gating - decide what your adapter will even consider changing
class MyIpamNautobotAdapter(NautobotAdapter):
    def load(self):
        super().load()
        infoblox_type = MetadataType.objects.filter(name__iexact="last sync from infoblox").first()
        owned = {
            m.assigned_object_id: m.scoped_fields
            for m in ObjectMetadata.objects.filter(metadata_type=infoblox_type)
        }
        for model in self.get_all("ipaddress"):
            if "dns_name" in owned.get(model.pk, []):
                model.model_flags |= DiffSyncModelFlags.IGNORE   # or drop the field

# B. Write-time guard - your update() override, sparse attrs from Part 2
def update(self, attrs):
    if "dns_name" in attrs and self._foreign_owned("dns_name"):
        self.adapter.job.logger.warning(f"Refusing dns_name write on {self}, owned by Infoblox")
        attrs.pop("dns_name")
    return super().update(attrs)

# C. Invariant - CustomValidator, runs regardless of who is writing
class IPAddressOwnershipValidator(CustomValidator):
    model = "ipam.ipaddress"
    def clean(self):
        ...  # raise self.validation_error(...) on a violation
```

(A) is cheapest and catches the most. (B) is defence in depth. (C) is the only one
that also catches a human editing the object in the UI.

### For the UI requirement

`ObjectMetadata` is a normal Nautobot model with REST + GraphQL. To surface ownership:

- `template_extensions` — add a panel to the IPAddress detail page listing each
  `ObjectMetadata` row and its `scoped_fields`.
- `table_extensions` — add an "Owned by" column to the IPAddress list view.
- `filter_extensions` — filter the list by owning data source.

All three are declared on your `NautobotAppConfig` (see Part 1). No core changes.

---

## 3.4 Using DiffSync with your own custom models

Short answer: yes — with one requirement that is easy to miss.

### The DiffSync layer works with anything

`diffsync.DiffSyncModel` is a pure pydantic class with no Django dependency. It can
model your own tables, a vendor REST API, a YAML file, a CSV. The diff engine does not
know Nautobot exists.

### The `NautobotModel` layer requires Nautobot's `BaseModel`

`NautobotModel` calls two things that plain `django.db.models.Model` does not have:

| Call site | Requires | Provided by |
|---|---|---|
| `obj.validated_save()` — `contrib/model.py:246` | `validated_save()` | `nautobot/core/models/__init__.py:147` |
| `obj.associated_object_metadata.filter(...)` — `contrib/model.py:432, 449` | `GenericRelation` to `ObjectMetadata` | `nautobot/core/models/__init__.py:66` |

Both live on the abstract `BaseModel`. A plain Django model raises `AttributeError` at
the first save. Everything else `NautobotModel` touches — `_meta.get_fields()`,
`objects.all()`, `_meta.get_field()`, `cls._model()` — is standard Django and works
with any model.

So:

```python
from nautobot.apps.models import PrimaryModel   # or OrganizationalModel

class MyIpamRecord(PrimaryModel):
    ...
```

`PrimaryModel` and `OrganizationalModel` (`nautobot/core/models/generics.py:19-67`) are
identical except that `PrimaryModel` adds `tags = TagsField()`. Both bring
`ChangeLoggedModel`, `CustomFieldModel`, `RelationshipModel`, `ContactMixin`,
`NotesMixin`, `DynamicGroupsModelMixin`, `SavedViewMixin`, `DataComplianceModelMixin`.
Use `PrimaryModel` for actual network resources, `OrganizationalModel` for
categorization and grouping models.

### The bonus: your models join the provenance ledger for free

`BaseModel` sets `is_metadata_associable_model = True` by default
(`nautobot/core/models/__init__.py:61`). Your custom models therefore participate in
the same `ObjectMetadata` + `scoped_fields` system as core models, and Nautobot's own
detail-view metadata panel renders automatically — it is gated on exactly that flag at
`nautobot/core/ui/object_detail.py:2776` and `nautobot/core/views/utils.py:434`.

Note the deliberate opt-out pattern in Infoblox (`infoblox/models.py:145-150`):

```python
is_saved_view_model             = False
is_dynamic_group_associable_model = False
is_metadata_associable_model    = False
is_contact_associable_model     = False
is_data_compliance_model        = False
```

Config models should not carry provenance metadata — they *are* the configuration.
Do the same on your own config model; leave the defaults alone on your data models.

### Mixed adapters are normal

If a model in your sync has no Nautobot ORM row — the remote side of any integration —
use plain `DiffSyncModel`. Infoblox does exactly this: its remote-side models in
`integrations/infoblox/diffsync/models/base.py` are raw `DiffSyncModel`, while the
Nautobot side subclasses `NautobotModel`.

Only the **Nautobot-side adapter** needs to be a `NautobotAdapter` — and remember from
3.2 that this is also what gates automatic metadata stamping
(`isinstance(self.target_adapter, NautobotAdapter)`, `jobs/base.py:509`).
