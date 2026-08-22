# Part 5: Honest Assessment and Design Recommendations

## 5.1 "SSoT is very complete" — is it?

You said you did not believe it. Here is a fair, evidence-based verdict after reading
the code.

### What is genuinely solid

- **The diff engine is small, correct, and readable.** 2833 lines. The recursion is
  clean, the action decision is 12 lines, the flag enforcement is consistent and
  centralized. No hidden magic. This is good engineering.
- **The contrib ORM layer removes enormous boilerplate.** FK traversal, generic FKs,
  M2M, custom fields, custom relationships, ORM caching, `select_related` derivation —
  all handled generically off `_model._meta`. Writing this yourself would be weeks.
- **Config-as-a-model (`SSOTInfobloxConfig`)** is the right pattern for multi-instance,
  and it comes with forms, views, API, permissions for free via `PrimaryModel`.
- **The secrets architecture is genuinely well designed.** No stored values, role-based
  lookup, live fetch, template-injection guard. Better than most commercial tooling.
- **19 shipped integrations**: `aci`, `aristacv`, `bootstrap`, `cisco_sdwan`,
  `citrix_adm`, `device42`, `dna_center`, `infoblox`, `ipfabric`, `itential`,
  `librenms`, `meraki`, `servicenow`, `slurpit`, `solarwinds`, `vsphere` and more.
  That is real coverage.

### Where the "complete" claim overreaches

**1. There is no conflict resolution. At all.**
Search the codebase for arbitration logic between two data sources writing the same
field — there is none. Last writer wins. `SKIP_UNMATCHED_*` and
`*_deletable_models` are *blast-radius limiters*, not arbiters. Every strategy in
Part 3 is something **you build**. The framework gives you the ledger
(`ObjectMetadata.scoped_fields`) and the hooks; the policy is yours.

**2. Metadata is opt-in, recent, and inconsistently implemented.**
It is gated behind `enable_metadata_for` (`jobs/base.py:507`), requires the contrib
adapter, and exists in two parallel implementations whose `MetadataType` names differ
by a capital letter (`"Last sync from"` vs `"Last Sync from"` — Part 3). The
`ObjectMetadataAnnotation` path hardcodes `scoped_fields = []`, throwing away the
field-scoping that makes the feature valuable in the first place. This subsystem is
young. Treat it as a foundation to build on, not a finished product.

**3. Diff is whole-dataset, in-memory, both sides.**
`total_models = len(src) + len(dst)` (`helpers.py:57`). No incremental sync, no
change-feed, no watermarking. For a large IPAM estate this is your first wall.
`RedisStore` moves the store off-heap but does not change the algorithm.

**4. No transactional boundary across the sync.**
Each `create`/`update`/`delete` is its own save. A mid-sync failure under
`CONTINUE_ON_FAILURE` leaves partial state. There is no rollback and no two-phase
commit. Acceptable for eventually-consistent reconciliation; not acceptable if you
need atomicity.

**5. Comparison is naive `!=`.**
No normalization, no type-aware comparison. Every integration re-solves case
folding, `None` vs `""`, and list ordering by hand in `load()`. `contrib/sorting.py`
exists for relationship ordering but is **explicitly disabled** — see
`jobs/base.py:513-514`:
> `# NOTE: Disabled for the time being due to ongoing issues.`
> `# sort_relationships(self.source_adapter, self.target_adapter)`

That is a known-broken feature sitting commented out in the main job flow.

### Verdict

SSoT is a **very good reconciliation engine and ORM abstraction**. It is **not** a
data-governance or conflict-resolution product. Your skepticism was directionally
right, but the conclusion should not be "avoid it" — it should be "adopt it for the
engine, and expect to build the governance layer yourself." The important thing is
that the framework does not fight you: `scoped_fields`, per-instance model flags, and
`CustomValidator` are genuinely sufficient primitives. They are just primitives.

## 5.2 Concrete recommendations for your IPAM app

**1. Build on `NautobotAdapter` / `NautobotModel`, not raw `diffsync`.**
Hand-rolling forfeits ORM caching, `select_related` derivation, FK/M2M/custom-field
handling, and — decisively — **all metadata support**, which is gated on
`isinstance(self.target_adapter, NautobotAdapter)`.

**2. Split your models by ownership, not by ORM table.**
Follow the Infoblox pattern exactly: multiple DiffSync model types, identical
`_identifiers`, disjoint `_attributes`, over the same `ipam.IPAddress`. Never declare
`dns_name` in your IPAM model. This makes the conflict structurally impossible rather
than policy-prevented.

**3. Enumerate the contested fields explicitly.**
Write them down before you write code. Likely set: `description`, `status`, `tenant`,
`role`, `custom_fields`. For each, pick an owner and document it. Infoblox left this
overlap unresolved — do not inherit that.

**4. Turn on metadata from day one.**

```python
PLUGINS_CONFIG = {
    "nautobot_ssot": {
        "enable_metadata_for": ["MyIpamDataSource", "MyIpamDataTarget"],
    },
}
```

Resolve the `MetadataType` case-insensitively. Do not hardcode `"Last sync from"`.

**5. Default `*_deletable_models` to empty.**
Copy the Infoblox pattern (`models.py:136-150`). Deletion authority should be opt-in
per model type per instance. In a two-writer environment, an unrestricted delete pass
is a mutual-annihilation loop.

**6. Set `SKIP_UNMATCHED_DST` until you have proven ownership.**
Both as a job-level `DiffSyncFlags` default and per-instance during `load()` for
objects whose `ObjectMetadata` names another source.

**7. Use `ExternalIntegration` for your remote endpoint.**
Do not invent URL/credential fields. Validate the `SecretsGroup` contents in your
config model's `clean()`, exactly as `_clean_infoblox_instance` does
(`infoblox/models.py:253-277`).

**8. Add a `CustomValidator` for the one invariant you cannot afford to lose.**
It is the only hook that also fires for UI edits and REST writes.

**9. Surface provenance in the UI early.**
`template_extensions` panel + `table_extensions` column. Being able to *see* which
source owns which field is what will make the design defensible in review — and it is
a small amount of code once `ObjectMetadata` is being written.

**10. Normalize aggressively in `load()`.**
Lowercase, sort lists, coerce `None`/`""` consistently on both sides. Core already
lowercases `dns_name` in `clean()` (`nautobot/ipam/models.py:1844`); mirror every such
rule or you will diff-thrash forever.

## 5.3 Open questions worth putting to the Nautobot team

1. Is the `"Last sync from"` / `"Last Sync from"` naming split intentional, or a bug?
   Which is the supported path going forward?
2. Why does `ObjectMetadataAnnotation` hardcode `scoped_fields = []` when the
   automatic path derives real scoping?
3. What is the plan for `sort_relationships`, currently disabled in `jobs/base.py:513`?
4. Is there any intended arbitration story between two apps writing the same field, or
   is that permanently app-author territory?
5. Will their Infoblox app declare `dns_name` ownership via metadata, so a second IPAM
   app can detect it? If not, ask them to — it costs them nothing and is the
   difference between coexistence and collision.

Question 5 is the one to raise first, and raise it before either app ships. It is a
five-line change on their side and it determines whether your app can be safe by
construction or only by convention.
