# Part 2: How Diff Detection Actually Works

Everything here is in `../diffsync/diffsync/` — 2833 lines total. It is small enough
to read end to end, and you should, because the whole conflict-avoidance strategy in
Part 3 depends on one specific line of it.

## 2.1 The data model

`diffsync/__init__.py:60` — `DiffSyncModel(pydantic.BaseModel)`. Five class-level
declarations control everything:

```python
_modelname: ClassVar[str] = "diffsyncmodel"   # key used to store/look up instances
_identifiers: ClassVar[Tuple[str, ...]] = ()  # globally unique natural key
_shortname:   ClassVar[Tuple[str, ...]] = ()  # locally unique, display only
_attributes:  ClassVar[Tuple[str, ...]] = ()  # THE ONLY FIELDS THAT ARE DIFFED
_children:    ClassVar[Dict[str, str]] = {}   # {_modelname: field_name}
```

The docstring at line 92 states it plainly:

> Only the fields in `_attributes` (as well as any `_children` fields) will be
> considered for the purposes of Diff calculation. A model may define additional
> fields (not included in `_attributes`) for its internal use.

Mutual exclusivity between the three groups is enforced at class-declaration time by
`__pydantic_init_subclass__` (`diffsync/__init__.py:130-158`) — it validates that
every named field exists and that no field appears in two groups. You get an
`AttributeError` at import, not a silent bug at runtime.

Identity is derived, not stored:

```python
def get_identifiers(self):  return self.dict(include=set(self._identifiers))        # :331
def get_attrs(self):        return self.dict(include=set(self._attributes))         # :339
def get_unique_id(self):    return self.create_unique_id(**self.get_identifiers())  # :352
```

`create_unique_id` joins identifier values with `__`. Two models with different
`_modelname` but identical `_identifiers` produce the same unique id **in their own
namespaces** — not a collision, because the store is keyed by
`(_modelname, unique_id)`. **That is the trick Infoblox uses, and the one you want.**

## 2.2 The three-phase pipeline

`sync_from` (`diffsync/__init__.py:563`) is the entry point. `sync_to` at line 608 is
literally `return target.sync_from(self, ...)` — one implementation, two directions.

```
load()          adapter-specific; populates the in-memory store
   |
diff_from()  -> DiffSyncDiffer.calculate_diffs()  -> Diff (tree of DiffElement)
   |
sync_from()  -> DiffSyncSyncer.perform_sync()     -> create()/update()/delete()
   |
sync_complete()   only fired if something actually changed
```

### Phase 1 — diff calculation (`helpers.py:34-282`)

`calculate_diffs()` (`helpers.py:69`) starts at `top_level` and recurses:

```
diff_object_list -> diff_object_pair -> diff_child_objects -> diff_object_list -> ...
```

**Top-level type mismatch is silently skipped.** Line 76:

```python
skipped_types = symmetric_difference(self.dst_diffsync.top_level, self.src_diffsync.top_level)
...
for obj_type in intersection(self.dst_diffsync.top_level, self.src_diffsync.top_level):
```

If your source adapter declares `top_level = ["prefix"]` and the target declares
`top_level = ["prefix", "ipaddress"]`, the `ipaddress` tree is never examined and
never touched. Coarse-grained, but genuinely useful ownership control.

**Pairing** (`helpers.py:110-121`) builds `{unique_id: (src_obj, dst_obj)}` by
outer-joining both sides on `get_unique_id()`. Then `validate_objects_for_diff`
(`helpers.py:141`) raises `TypeError`/`ValueError` if a matched pair disagrees on
type, shortname, or identifiers — a guard against store corruption.

### The action decision (`diff.py:237-253`)

This is the entire "diff detection" logic. Twelve lines:

```python
@property
def action(self):
    if self.source_attrs is not None and self.dest_attrs is None:
        return DiffSyncActions.CREATE
    if self.source_attrs is None and self.dest_attrs is not None:
        return DiffSyncActions.DELETE
    if (self.source_attrs is not None and self.dest_attrs is not None
        and any(self.source_attrs[k] != self.dest_attrs[k] for k in self.get_attrs_keys())):
        return DiffSyncActions.UPDATE
    return None
```

Presence on one side only => create or delete. Present on both with any unequal attr
=> update. Comparison is plain Python `!=` on pydantic-coerced values. There is **no**
semantic comparison, no tolerance, no normalization. Anything you need normalized
(case, list ordering, `None` vs `""`) you must normalize **inside `load()`**, or you
will get perpetual false-positive updates on every run.

### The line that matters most — `get_attrs_keys` (`diff.py:266-279`)

```python
def get_attrs_keys(self):
    if self.source_attrs is not None and self.dest_attrs is not None:
        return intersection(list(self.dest_attrs.keys()), list(self.source_attrs.keys()))
    if self.source_attrs is None and self.dest_attrs is not None:
        return self.dest_attrs.keys()
    if self.source_attrs is not None and self.dest_attrs is None:
        return self.source_attrs.keys()
    return []
```

**Comparison runs over the INTERSECTION of the two sides' attribute keys.**

This is the single most important fact for your project. A field one side does not
load is not compared, and therefore is never written. Field-level ownership is
achieved not through configuration but by **not declaring the field in `_attributes`
on the model that should not own it.** Part 3 builds on this.

`get_attrs_diffs()` then returns `{"-": {old}, "+": {new}}` containing only keys that
actually differ. The syncer consumes only `"+"`.

### Phase 2 — sync (`helpers.py:284-473`)

`sync_diff_element` (`helpers.py:335`) resolves the model class off the **destination**
adapter, then dispatches in `sync_model` (`helpers.py:409`):

```python
if   self.action == CREATE: dst_model = self.model_class.create(adapter=self.dst_diffsync, ids=ids, attrs=attrs)
elif self.action == UPDATE: dst_model = dst_model.update(attrs=attrs)
elif self.action == DELETE: dst_model = dst_model.delete()
```

Note `attrs = diffs.get("+", {})` — **updates carry only changed fields**, so a partial
write is the norm, not an exception. Your `update()` override receives a sparse dict.

Guard rails: create with an existing dst raises `ObjectNotCreated`; update/delete with
a missing dst raises `ObjectNotUpdated`/`ObjectNotDeleted`. `ObjectCrudException` is
caught and, under `CONTINUE_ON_FAILURE`, downgraded to a logged error returning
`(True, None)`; otherwise re-raised (`helpers.py:445-451`).

**Ordering.** Default is parent-then-children. `NATURAL_DELETION_ORDER` flips it for
deletes only (`helpers.py:373-376`) so children die first. `SKIP_CHILDREN_ON_DELETE`
covers the DB-cascade case where recursing would double-delete. Both are read off the
**destination** model instance (`helpers.py:369-371`) — a source-side flag has no
effect here.

Ordering *within* a type is customizable by subclassing `Diff` and defining
`order_children_<group>` (`diff.py:85-108`); `Diff.get_children()` looks up
`order_children_{group}` by name and falls back to `order_children_default`. That is
your hook if IPAM objects must be created before DNS records.

## 2.3 The flag matrix

Two independent enums (`diffsync/enum.py`).

**`DiffSyncModelFlags`** — per model class *or per instance*, set during `load()`:

| Flag | Effect | Enforced at |
|---|---|---|
| `IGNORE` | Never diffed, never changed | `helpers.py:203-210` |
| `SKIP_UNMATCHED_SRC` | Src-only => do not create in dst | `helpers.py:195` |
| `SKIP_UNMATCHED_DST` | Dst-only => do not delete from dst | `helpers.py:199` |
| `SKIP_CHILDREN_ON_DELETE` | Do not recurse into children on delete | `helpers.py:371` |
| `NATURAL_DELETION_ORDER` | Delete children before parent | `helpers.py:370` |

**`DiffSyncFlags`** — passed to the whole `sync_*`/`diff_*` call:
`CONTINUE_ON_FAILURE`, `SKIP_UNMATCHED_SRC`, `SKIP_UNMATCHED_DST`,
`LOG_UNCHANGED_RECORDS`.

**`SKIP_UNMATCHED_DST` is the most important one for multi-app coexistence.** Without
it, any object in Nautobot the remote system does not know about is deleted. With two
apps writing overlapping IP data, that is a mutual-annihilation loop: your IPAM sync
deletes the DNS app's records, its next run deletes yours. Set it per-instance during
`load()` on objects your app did not create — which requires knowing who created them,
which is exactly what Part 3 provides.

Per-instance flag setting is the powerful, under-documented move:

```python
def load(self):
    for obj in ...:
        model = self.ipaddress(...)
        if owned_by_someone_else(obj):
            model.model_flags |= DiffSyncModelFlags.IGNORE
        self.add(model)
```

## 2.4 Storage and scale

`diffsync/store/` — `LocalStore` (plain dict, default) and `RedisStore`. Redis matters
if your sync exceeds worker memory; the diff computation itself is fully in-memory in
both cases.

Both sides are loaded **completely** before comparison begins —
`DiffSyncDiffer.__init__` (`helpers.py:57`) computes
`total_models = len(src_diffsync) + len(dst_diffsync)` up front. For a large IPAM
estate this is the scaling wall you will hit first, well before diff CPU cost matters.
