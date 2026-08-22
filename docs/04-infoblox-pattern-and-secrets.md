# Part 4: The Infoblox Pattern, Multi-Grid Config, and Secrets

## 4.1 Multiple grids — how it is actually done

Your Nautobot contact said they can "store multiple grid data and other configs in the
DB and set metadata". That is accurate, and here is the mechanism.

`../nautobot-app-ssot/nautobot_ssot/integrations/infoblox/models.py:41`

```python
class SSOTInfobloxConfig(PrimaryModel):
    name = models.CharField(max_length=CHARFIELD_MAX_LENGTH, unique=True)
    default_status    = FK("extras.Status", on_delete=PROTECT)
    infoblox_instance = FK("extras.ExternalIntegration", on_delete=PROTECT)   # :57
    infoblox_wapi_version = CharField(...)
    ...
```

It is a **`PrimaryModel`, not a settings dict**. Each row is one grid. Multiple rows =
multiple grids, each with its own credentials, filters, and enable flags. This is the
right pattern and you should copy it wholesale for your IPAM app.

The per-instance knobs (`models.py:68-150`):

| Field | Purpose |
|---|---|
| `enable_sync_to_infoblox` / `enable_sync_to_nautobot` | Direction control per instance |
| `import_ip_addresses`, `import_subnets`, `import_vlan_views`, `import_vlans` | Object-class scoping |
| `import_ipv4` / `import_ipv6` | Address-family scoping |
| `infoblox_sync_filters` (JSON) | Network-view/prefix scoping |
| `infoblox_network_view_to_namespace_map` (JSON) | Maps grid network views to Nautobot Namespaces |
| `infoblox_dns_view_mapping` (JSON) | DNS view mapping |
| `infoblox_location_ext_attr` | Which Infoblox extattr carries location |
| `cf_fields_ignore` (JSON) | Custom fields to skip |
| `dns_record_type`, `fixed_address_type` | Which record flavours to manage |
| **`infoblox_deletable_models`** (JSON) | **What this instance may delete in Infoblox** |
| **`nautobot_deletable_models`** (JSON) | **What this instance may delete in Nautobot** |
| `job_enabled` | Kill switch |

### The deletable-models pattern is the precedence knob

The two `*_deletable_models` fields are a per-instance, per-model-type delete
authorization list, checked at the actual delete site rather than centrally:

```
diffsync/models/nautobot.py:380  if NautobotDeletableModelChoices.IP_ADDRESS      not in self.adapter.config.nautobot_deletable_models: ...
diffsync/models/nautobot.py:600  if NautobotDeletableModelChoices.DNS_A_RECORD    not in ...
diffsync/models/nautobot.py:702  if NautobotDeletableModelChoices.DNS_HOST_RECORD not in ...
diffsync/models/nautobot.py:787  if NautobotDeletableModelChoices.DNS_PTR_RECORD  not in ...
diffsync/models/infoblox.py:258  if InfobloxDeletableModelChoices.FIXED_ADDRESS   not in self.adapter.config.infoblox_deletable_models: ...
```

Validated in `models.py:351-366`, exposed as a `MultipleChoiceField` in
`forms.py:48-58`. Default is empty list — **delete nothing unless explicitly
authorized**. That is the correct default for a multi-writer environment, and the
opposite of what raw diffsync does. Adopt it.

## 4.2 The IPAM/DNS separation — copy this exactly

This is the answer to your core question, and it is more elegant than you would
expect. From `integrations/infoblox/diffsync/models/base.py`:

```python
class IPAddress(DiffSyncModel):                                          # :67
    _modelname   = "ipaddress"
    _identifiers = ("address", "prefix", "prefix_length", "namespace")
    _attributes  = ("description", "status", "ip_addr_type", "ext_attrs",
                    "has_host_record", "has_a_record", "has_ptr_record",
                    "has_fixed_address", "mac_address", "fixed_address_comment")

class DnsARecord(DiffSyncModel):                                         # :105
    _modelname   = "dnsarecord"
    _identifiers = ("address", "prefix", "prefix_length", "namespace")   # IDENTICAL
    _attributes  = ("dns_name", "ip_addr_type", "description", "status", "ext_attrs")

class DnsHostRecord(DiffSyncModel):   # :132   same identifiers, same attrs shape
class DnsPTRRecord(DiffSyncModel):    # :159   same identifiers, same attrs shape
```

**Same underlying object. Same identifiers. Four separate DiffSync model types with
different `_attributes`.**

The critical detail: **`ipaddress` does not declare `dns_name`.** Only the DNS record
models do. So the IPAM-facing diff physically cannot produce a `dns_name` write — not
by policy, not by a config flag, but because `get_attrs_keys()` (Part 2) will never
include a key that is not in `_attributes`.

Meanwhile the `ipaddress` model carries `has_a_record` / `has_host_record` /
`has_ptr_record` / `has_fixed_address` booleans. It **observes** the DNS state as
read-only signal without **owning** it. That is the pattern: model the overlap as an
observed attribute on one side and an owned attribute on the other.

### Where the overlap is still real

`description`, `status`, `ip_addr_type`, and `ext_attrs` appear in **both**
`_attributes` lists. These are genuine contention points — two model types can both
write them to the same ORM row, and last-writer-wins within a single sync run.
Infoblox mitigates this with sync direction flags and record-type config, not with
field arbitration.

**For your design: enumerate the overlap explicitly and decide per field.** Do not
assume the split is clean just because the identifiers line up. In your IPAM-vs-DNS
case the likely contested set is `description`, `status`, `tenant`, and `role`.

### Suggested split for your app

```
MyIpamIPAddress   _attributes = ("status", "role", "tenant", "namespace_ref",
                                 "mask_length", "type", "custom_fields")
                  # explicitly NOT dns_name

DnsRecord         _attributes = ("dns_name", ...)
                  # owned by the DNS/Infoblox app
```

`ipam.IPAddress.dns_name` exists in core at `nautobot/ipam/models.py:1740` and is
force-lowercased in `clean()` (`:1844-1846`) — a good example of normalization you
must mirror in `load()` or you will diff-thrash on case.

## 4.3 Secrets — your suspicion was correct

**Confirmed: Nautobot never stores secret values in its database.**

### The model

`../nautobot/nautobot/extras/models/secrets.py:27`

```python
class Secret(PrimaryModel):
    provider   = models.CharField(...)     # :37  a registry key, e.g. "hashicorp-vault"
    parameters = models.JSONField(...)     # :38  HOW to fetch, never WHAT

    def rendered_parameters(self, obj=None):                              # :55
        return {k: render_jinja2(v, {"obj": obj}) for k, v in self.parameters.items()}

    def get_value(self, obj=None):                                        # :63
        provider = registry["secrets_providers"].get(self.provider)       # :71
        if not provider:
            raise SecretProviderError(...)
        return provider.get_value_for_secret(self, obj=obj)               # :76
```

There is no value column and no caching layer. Every read is a live fetch through the
provider. `parameters` are Jinja2-rendered against the requesting object, so one
`Secret` row can resolve differently per device/site.

### The three-class split you noticed

You were right that the separation is deliberate:

```
Secret                    WHERE a credential lives (provider + parameters)
SecretsGroup              a named bundle of credentials                 :101
SecretsGroupAssociation   binds Secret -> Group with a ROLE             :132
                          unique_together (group, access_type, secret_type)
```

`SecretsGroupAssociation` (`:132-155`) is the interesting one. `access_type` (HTTP,
REST, SSH, ...) and `secret_type` (username, password, token, key) form a compound
role, uniquely constrained per group. So consumers ask by **role**, not by name:

```python
SecretsGroup.get_secret_value(access_type, secret_type, obj=None)   # :116
```

Which is exactly how Infoblox consumes it (`infoblox/models.py:253-277`) — validating
at `clean()` time that the group actually contains a REST/username and REST/password
pair, and failing config save if not. Copy that validation pattern; it turns a runtime
auth failure into a form error.

### What core ships vs. what you install

Core has exactly **two** providers (`nautobot/extras/secrets/providers.py`):

| Slug | Class | Line |
|---|---|---|
| `environment-variable` | `EnvironmentVariableSecretsProvider` | `:16` |
| `text-file` | `TextFileSecretsProvider` | `:39` |

(`TextFileSecretsProvider.ParametersForm.clean` blocks relative paths and `..`
sequences — `:48-56`.)

Everything else comes from `nautobot-app-secrets-providers`:

| Slug | Provider |
|---|---|
| `hashicorp-vault` | HashiCorp Vault |
| `aws-secrets-manager` | AWS Secrets Manager |
| `aws-sm-parameter-store` | AWS Systems Manager Parameter Store |
| `azure-key-vault` | Azure Key Vault |
| `one-password` | 1Password |
| (two classes) | Delinea Secret Server, by ID and by Path |

### The OpenShift/pod-secret claim — verified

This is the part you specifically doubted. It is true, and here is the code.
`nautobot_secrets_providers/providers/hashicorp.py`:

```python
K8S_TOKEN_DEFAULT_PATH = "/var/run/secrets/kubernetes.io/serviceaccount/token"   # :23
AUTH_METHOD_CHOICES = ["approle", "aws", "kubernetes", "token"]                  # :24
...
elif auth_method == "kubernetes":                                                # :168
    with open(k8s_token_path, "r", encoding="utf-8") as token_file:
        jwt = token_file.read()
    client.auth.kubernetes.login(role=vault_settings["role_name"], jwt=jwt, **login_kwargs)
```

The full OpenShift chain:

```
Pod service account
  -> projected JWT at /var/run/secrets/kubernetes.io/serviceaccount/token
  -> Vault kubernetes auth login (role from PLUGINS_CONFIG, not the DB)
  -> short-lived Vault token, in memory only
  -> secret value fetched live per request
```

Nothing is persisted. The pod's own identity is the root of trust, and the Nautobot DB
holds only the path/role/mount metadata. Validation is eager
(`hashicorp.py:104-130`): invalid `auth_method`, missing `token`, missing `role_name`
for kubernetes, or missing `role_id`/`secret_id` for approle all raise
`SecretProviderError` before any network call.

The chain your Infoblox config walks, end to end:

```
SSOTInfobloxConfig.infoblox_instance
  -> extras.ExternalIntegration           (nautobot/extras/models/models.py:483)
       .remote_url, .verify_ssl, .timeout, .ca_file_path, .headers, .extra_config
       .secrets_group  ---------------->  extras.SecretsGroup
                                            -> SecretsGroupAssociation (access_type, secret_type)
                                                 -> Secret (provider, parameters)
                                                      -> provider.get_value_for_secret()
                                                           -> live fetch from Vault/env/file
```

**`ExternalIntegration` is a core model** — use it for your IPAM app's remote endpoint
rather than inventing your own URL/credential fields. You get URL, TLS verification,
CA path, timeout, HTTP method, headers, and an `extra_config` JSON escape hatch for
free, plus the secrets binding.

### One caveat worth knowing

`Secret.get_value` and `SecretsGroup.get_secret_value` are both decorated `@unsafe`
(`secrets.py:115`, and `get_secret_value.do_not_call_in_templates = True` at `:125`).
That decorator blocks them from being called inside Django templates and Jinja2
rendering — a deliberate guard against secret exfiltration via a config-context or
computed-field template. Do not try to route around it.
