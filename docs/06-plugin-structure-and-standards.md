# Part 6: Nautobot App Structure & Development Standards

Copy-pasteable skeleton for a new Nautobot app, matching the conventions actually used
by `nautobot-app-ssot` 4.6.2a0 and `nautobot-app-device-onboarding`. Targets Nautobot
3.2.x. Terminology note: "plugin" is the legacy name; current docs and class names say
**App**. `NautobotAppConfig`, not `PluginConfig`.

## 6.0 Before you write anything — five decisions

1. **App name.** Python package `nautobot_<thing>`, PyPI dist `nautobot-app-<thing>`,
   repo `nautobot-app-<thing>`. The package name becomes your Django `app_label`,
   appears in every permission string (`nautobot_myipam.view_myipamrecord`), and in URL
   namespaces (`plugins:nautobot_myipam:...`). **You cannot change it later without a
   migration and a breaking release.** Pick carefully.
2. **`PrimaryModel` vs `OrganizationalModel`** for each model — see Part 3.4.
3. **Which core models you will write to.** For you: `ipam.IPAddress`, `ipam.Prefix`,
   `ipam.Namespace`. Write down the field-level ownership split now (Part 4.2).
4. **Does it need its own models at all,** or only sync into core models? A pure SSoT
   app may need only a config model. Yours needs both.
5. **Config model vs `PLUGINS_CONFIG`.** Rule of thumb: anything an operator changes at
   runtime, or anything that can exist more than once (multiple grids, multiple
   endpoints), is a **model**. Only deployment-level toggles go in `PLUGINS_CONFIG`.

## 6.1 Repository layout

Verified against `nautobot-app-device-onboarding`, which is cookiecutter-generated
from `nautobot/cookiecutter-nautobot-app` (note its `.cookiecutter.json`).

```
nautobot-app-myipam/
├── .cookiecutter.json                  # if generated from the official template
├── .github/
│   ├── CODEOWNERS
│   ├── ISSUE_TEMPLATE/
│   ├── pull_request_template.md
│   └── workflows/ci.yml
├── changes/                            # towncrier fragments, one file per PR
│   └── .gitignore
├── development/
│   ├── Dockerfile
│   ├── docker-compose.base.yml
│   ├── docker-compose.dev.yml
│   ├── docker-compose.postgres.yml
│   ├── docker-compose.redis.yml
│   ├── creds.example.env
│   ├── development.env
│   ├── nautobot_config.py              # dev-only config; PLUGINS = ["nautobot_myipam"]
│   └── app_config_schema.py
├── docs/
│   ├── index.md
│   ├── admin/                          # install, upgrade, compatibility matrix
│   ├── user/                           # what it does, screenshots
│   └── dev/                            # architecture, contributing
├── nautobot_myipam/                    # THE PACKAGE
│   ├── __init__.py                     # NautobotAppConfig  <- entry point
│   ├── app-config-schema.json          # JSON schema for PLUGINS_CONFIG validation
│   ├── api/
│   │   ├── __init__.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
│   ├── diffsync/                       # only if you do SSoT
│   │   ├── adapters/
│   │   │   ├── myipam.py               # remote system
│   │   │   └── nautobot.py             # NautobotAdapter subclass
│   │   └── models/
│   │       ├── base.py                 # shared _identifiers/_attributes contracts
│   │       ├── myipam.py               # remote-side CRUD
│   │       └── nautobot.py             # Nautobot-side CRUD
│   ├── migrations/
│   │   └── __init__.py
│   ├── templates/nautobot_myipam/      # NAMESPACED - never bare templates/
│   ├── static/nautobot_myipam/         # NAMESPACED
│   ├── tests/
│   │   └── __init__.py
│   ├── utils/
│   │   ├── __init__.py
│   │   └── client.py                   # remote API client, no Django imports
│   ├── choices.py
│   ├── constants.py
│   ├── custom_validators.py            # <- conflict invariants (Part 3.3)
│   ├── exceptions.py
│   ├── filter_extensions.py            # add filters to CORE models
│   ├── filters.py                      # filters for YOUR models
│   ├── forms.py
│   ├── jobs.py
│   ├── models.py
│   ├── navigation.py
│   ├── signals.py                      # post_migrate: create Statuses, MetadataTypes
│   ├── table_extensions.py             # add columns to CORE model tables
│   ├── tables.py
│   ├── template_content.py             # add panels to CORE model detail pages
│   └── urls.py
├── mkdocs.yml
├── pyproject.toml
├── tasks.py                            # invoke tasks
├── invoke.example.yml
├── LICENSE
└── README.md
```

**Do not deviate from the default module names** (`jobs.py`, `navigation.py`,
`custom_validators.py`, ...). They are the dotted paths `NautobotAppConfig` resolves at
load time (Part 1). You *can* override the attribute to relocate a module, but every
other Nautobot developer will expect the defaults.

Two hard rules:
- **Namespace `templates/` and `static/`** under a directory matching your package
  name. Django flattens these across all installed apps; a bare `templates/base.html`
  will silently shadow core's.
- **Create `migrations/__init__.py` on day one.** Without it Django will not detect the
  app as migratable, and the failure mode is confusing.

## 6.2 `__init__.py` — the entry point

```python
"""App declaration for nautobot_myipam."""

from importlib import metadata

from nautobot.apps import NautobotAppConfig

__version__ = metadata.version(__name__)


class NautobotMyIpamConfig(NautobotAppConfig):
    """App configuration for the nautobot_myipam app."""

    name = "nautobot_myipam"                  # MUST match the package directory
    verbose_name = "My IPAM"
    version = __version__
    author = "Nikhil Mohite"
    author_email = "nikhilmohite1993@gmail.com"
    description = "IPAM data source integration with field-level ownership."
    base_url = "myipam"                       # mounts at /plugins/myipam/
    min_version = "3.2.0"                     # be honest; it is enforced
    max_version = "3.99"

    required_settings = []                    # fail loudly at startup if absent
    default_settings = {
        "enable_sync": True,
        "default_status_name": "Active",
    }

    caching_config = {}

    def ready(self):
        """Import signal handlers after the app registry is populated."""
        super().ready()
        from . import signals  # noqa: F401  pylint: disable=unused-import


config = NautobotMyIpamConfig
```

The trailing `config = ...` assignment is the convention Nautobot looks for.

**`ready()` is the only safe place to import signals.** Importing models at module level
in `__init__.py` raises `AppRegistryNotReady`.

## 6.3 `models.py`

```python
"""Models for nautobot_myipam."""

from django.core.exceptions import ValidationError
from django.core.serializers.json import DjangoJSONEncoder
from django.db import models
from nautobot.apps.models import OrganizationalModel, PrimaryModel
from nautobot.core.models.fields import CHARFIELD_MAX_LENGTH


class MyIpamConfig(PrimaryModel):
    """One row per remote IPAM instance. See Part 4.1 for why this is a model."""

    name = models.CharField(max_length=CHARFIELD_MAX_LENGTH, unique=True)
    description = models.CharField(max_length=CHARFIELD_MAX_LENGTH, blank=True)

    # Reuse the core integration model - free URL/TLS/CA/timeout/headers + secrets
    remote_instance = models.ForeignKey(
        to="extras.ExternalIntegration",
        on_delete=models.PROTECT,
        help_text="Remote IPAM endpoint and credentials",
    )
    default_status = models.ForeignKey(
        to="extras.Status",
        on_delete=models.PROTECT,
        help_text="Status applied to imported objects",
    )

    enable_sync_to_nautobot = models.BooleanField(default=True)
    enable_sync_to_remote = models.BooleanField(default=False)
    sync_filters = models.JSONField(default=list, encoder=DjangoJSONEncoder, blank=True)

    # Delete authority: opt-in per model type. Default empty = delete nothing.
    nautobot_deletable_models = models.JSONField(
        default=list, encoder=DjangoJSONEncoder, blank=True,
        help_text="Model types this instance is permitted to delete in Nautobot.",
    )

    job_enabled = models.BooleanField(default=False)

    # This is configuration, not network data - opt out of the metadata frameworks.
    is_metadata_associable_model = False
    is_contact_associable_model = False
    is_dynamic_group_associable_model = False
    is_saved_view_model = False
    is_data_compliance_model = False

    class Meta:
        verbose_name = "My IPAM Config"
        verbose_name_plural = "My IPAM Configs"
        ordering = ("name",)

    def __str__(self):
        return self.name

    def clean(self):
        """Validate credentials at config-save time, not at 3am during a sync."""
        super().clean()
        secrets_group = self.remote_instance.secrets_group
        if not secrets_group:
            raise ValidationError(
                {"remote_instance": "The external integration must have a Secrets Group assigned."}
            )
        # Assert the group actually contains the roles you will ask for at runtime.
        # See infoblox/models.py:253-277 for the pattern this mirrors.
```

Notes:
- `__str__` and `Meta.ordering` are required in practice — tables and the UI depend on
  both.
- Use `CHARFIELD_MAX_LENGTH` from core rather than a literal `255`.
- `on_delete=models.PROTECT` for config FKs. `CASCADE` on a `Status` FK means deleting
  a status silently destroys your configuration.

## 6.4 `views.py` — modern viewset composition

Nautobot 3.x composes mixins rather than using the monolithic `NautobotUIViewSet`.
Pattern taken verbatim from `integrations/infoblox/views.py`:

```python
"""Views for nautobot_myipam."""

from nautobot.apps.ui import (
    Breadcrumbs, ModelBreadcrumbItem, ObjectDetailContent,
    ObjectFieldsPanel, ObjectTextPanel, SectionChoices,
)
from nautobot.apps.views import (
    ObjectChangeLogViewMixin, ObjectDestroyViewMixin, ObjectDetailViewMixin,
    ObjectEditViewMixin, ObjectListViewMixin, ObjectNotesViewMixin,
)

from .api.serializers import MyIpamConfigSerializer
from .filters import MyIpamConfigFilterSet
from .forms import MyIpamConfigFilterForm, MyIpamConfigForm
from .models import MyIpamConfig
from .tables import MyIpamConfigTable


class MyIpamConfigUIViewSet(
    ObjectDestroyViewMixin,
    ObjectDetailViewMixin,
    ObjectListViewMixin,
    ObjectEditViewMixin,
    ObjectChangeLogViewMixin,
    ObjectNotesViewMixin,
):  # pylint: disable=abstract-method
    """MyIpamConfig UI ViewSet."""

    queryset = MyIpamConfig.objects.all()
    table_class = MyIpamConfigTable
    filterset_class = MyIpamConfigFilterSet
    filterset_form_class = MyIpamConfigFilterForm
    form_class = MyIpamConfigForm
    serializer_class = MyIpamConfigSerializer
    lookup_field = "pk"

    object_detail_content = ObjectDetailContent(
        panels=[
            ObjectFieldsPanel(
                section=SectionChoices.LEFT_HALF,
                weight=100,
                fields="__all__",
            ),
        ],
    )
```

Take only the mixins you need — omitting `ObjectDestroyViewMixin` genuinely removes
delete. `object_detail_content` is declarative; you rarely need a custom template.

## 6.5 `urls.py`

```python
"""URL patterns for nautobot_myipam."""

from nautobot.apps.urls import NautobotUIViewSetRouter

from . import views

router = NautobotUIViewSetRouter()
router.register("config", viewset=views.MyIpamConfigUIViewSet)

urlpatterns = []
urlpatterns += router.urls
```

Reverse names become `plugins:nautobot_myipam:myipamconfig_list` etc. The router
derives them from the model name — do not hand-write path names.

## 6.6 `navigation.py`

```python
"""Navigation menu for nautobot_myipam."""

from nautobot.apps.ui import NavMenuAddButton, NavMenuGroup, NavMenuItem, NavMenuTab

menu_items = (
    NavMenuTab(
        name="Apps",
        groups=(
            NavMenuGroup(
                name="My IPAM",
                weight=100,
                items=(
                    NavMenuItem(
                        link="plugins:nautobot_myipam:myipamconfig_list",
                        name="Configs",
                        permissions=["nautobot_myipam.view_myipamconfig"],
                        buttons=(
                            NavMenuAddButton(
                                link="plugins:nautobot_myipam:myipamconfig_add",
                                permissions=["nautobot_myipam.add_myipamconfig"],
                            ),
                        ),
                    ),
                ),
            ),
        ),
    ),
)
```

**Always set `permissions`.** Without it the menu entry shows for users who cannot open
the page.

## 6.7 `jobs.py`

```python
"""Jobs for nautobot_myipam."""

from nautobot.apps.jobs import ObjectVar, register_jobs
from nautobot_ssot.jobs.base import DataMapping, DataSource

from .diffsync.adapters import myipam, nautobot
from .models import MyIpamConfig

name = "My IPAM SSoT"   # job group heading in the UI


class MyIpamDataSource(DataSource):
    """Sync from the remote IPAM into Nautobot."""

    config = ObjectVar(model=MyIpamConfig, query_params={"job_enabled": True})

    class Meta:
        name = "My IPAM to Nautobot"
        data_source = "My IPAM"          # <- becomes the MetadataType name. Part 3.2.
        description = "Sync IPAM data into Nautobot with field-level ownership."

    @classmethod
    def data_mappings(cls):
        return (DataMapping("Subnet", None, "Prefix", None),)

    def load_source_adapter(self):
        self.source_adapter = myipam.MyIpamAdapter(job=self, sync=self.sync)
        self.source_adapter.load()

    def load_target_adapter(self):
        self.target_adapter = nautobot.MyIpamNautobotAdapter(job=self, sync=self.sync)
        self.target_adapter.load()


jobs = [MyIpamDataSource]
register_jobs(*jobs)
```

Two things that bite:
- **`register_jobs(*jobs)` at module scope is mandatory.** A job class that is merely
  defined is never discovered.
- **`Meta.data_source` becomes your `MetadataType` name** (`contrib/adapter.py:352`).
  Choose it once and never change it — renaming orphans every existing provenance row.

## 6.8 `signals.py` — post-migrate bootstrapping

This is where you create the objects your app assumes exist. `ObjectMetadataAnnotation`
in particular **requires** the `MetadataType` to pre-exist and raises
`ObjectCrudException` otherwise (`contrib/model.py:424-429`).

```python
"""Signal handlers for nautobot_myipam."""

from django.db.models.signals import post_migrate
from django.dispatch import receiver


@receiver(post_migrate)
def create_custom_objects(sender, apps, **kwargs):
    """Create Statuses, MetadataTypes, and CustomFields this app depends on."""
    if sender.name != "nautobot_myipam":
        return

    from django.contrib.contenttypes.models import ContentType
    from nautobot.extras.models import MetadataType
    from nautobot.extras.models.metadata import MetadataTypeDataTypeChoices
    from nautobot.ipam.models import IPAddress, Prefix

    metadata_type, _ = MetadataType.objects.get_or_create(
        name="Last sync from My IPAM",
        defaults={
            "data_type": MetadataTypeDataTypeChoices.TYPE_DATETIME,
            "description": "Timestamp of the last sync from My IPAM.",
        },
    )
    for model in (IPAddress, Prefix):
        metadata_type.content_types.add(ContentType.objects.get_for_model(model))
```

Guard on `sender.name` — `post_migrate` fires once **per installed app**, so an
unguarded handler runs dozens of times per migrate.

## 6.9 UI extension points for core models

These are how you surface ownership on `ipam.IPAddress` without forking core.

```python
# template_content.py  -> panel on the IPAddress detail page
from nautobot.apps.ui import TemplateExtension

class IPAddressOwnership(TemplateExtension):
    model = "ipam.ipaddress"

    def right_page(self):
        return self.render(
            "nautobot_myipam/inc/ownership_panel.html",
            extra_context={"metadata": self.context["object"].associated_object_metadata.all()},
        )

template_extensions = [IPAddressOwnership]
```

```python
# custom_validators.py  -> the invariant nobody can bypass (Part 3.3)
from nautobot.apps.models import CustomValidator

class IPAddressOwnershipValidator(CustomValidator):
    model = "ipam.ipaddress"

    def clean(self):
        obj = self.context["obj"]
        ...  # self.validation_error({"dns_name": "..."}) on a violation

custom_validators = [IPAddressOwnershipValidator]
```

Also available: `table_extensions.py` (columns on core list views) and
`filter_extensions.py` (filters on core models). All four are resolved by the dotted
paths on `NautobotAppConfig` (Part 1).

## 6.10 `pyproject.toml` essentials

```toml
[tool.poetry]
name = "nautobot-app-myipam"
version = "0.1.0"
description = "IPAM data source integration for Nautobot"
packages = [{ include = "nautobot_myipam" }]

[tool.poetry.dependencies]
python = ">=3.9,<3.14"
nautobot = "^3.2.0"
nautobot-ssot = "^4.6.0"

[tool.poetry.plugins."nautobot.app"]        # <- discovery entry point
myipam = "nautobot_myipam"
```

Pin `nautobot` with a caret, not `*`. `min_version`/`max_version` on the AppConfig
enforce it a second time at runtime — keep the two consistent or you get a confusing
startup error that contradicts your install metadata.

## 6.11 Development checklist

**Setup**
- [ ] `migrations/__init__.py` exists
- [ ] `templates/` and `static/` namespaced under the package name
- [ ] `development/nautobot_config.py` lists your app in `PLUGINS`
- [ ] `config = YourAppConfig` at the bottom of `__init__.py`
- [ ] Entry point declared in `pyproject.toml`

**Models**
- [ ] Inherits `PrimaryModel` / `OrganizationalModel`, not `django.db.models.Model`
- [ ] `__str__` and `Meta.ordering` defined
- [ ] `on_delete=PROTECT` on config FKs
- [ ] Config models opt out of the metadata/contact/saved-view frameworks
- [ ] `clean()` validates the `SecretsGroup` contains the roles you will request

**SSoT**
- [ ] Nautobot-side adapter subclasses `NautobotAdapter` (else: no metadata at all)
- [ ] `_attributes` excludes every field another app owns — especially `dns_name`
- [ ] `Meta.data_source` chosen once and frozen
- [ ] Job class name added to `PLUGINS_CONFIG["nautobot_ssot"]["enable_metadata_for"]`
- [ ] `*_deletable_models` defaults to empty
- [ ] `SKIP_UNMATCHED_DST` set until ownership is proven
- [ ] Values normalized in `load()` — case, list order, `None` vs `""`
- [ ] Dry-run tested against production-shaped data before any write

**Safety**
- [ ] `CustomValidator` registered for the one invariant you cannot lose
- [ ] `register_jobs(*jobs)` called at module scope
- [ ] `post_migrate` handler guarded on `sender.name`
- [ ] Nav items carry `permissions`

## 6.12 Bootstrapping

Use the official template rather than hand-creating the tree:

```bash
pip install cookiecutter
cookiecutter https://github.com/nautobot/cookiecutter-nautobot-app.git
```

It generates the layout above plus CI, towncrier, mkdocs, and the invoke tasks. Then
delete what you do not need. `nautobot-app-device-onboarding/.cookiecutter.json` shows
the answers a real app used, which is worth reading before you answer the prompts.

Standard dev loop once generated:

```bash
invoke build
invoke migrate
invoke createsuperuser
invoke debug          # foreground, logs visible
invoke nbshell        # Django shell with Nautobot loaded
invoke unittest
```
