# Part 7: Packaging & Distribution

How to turn the app into a wheel and get it onto a Nautobot instance that is not your
laptop. Verified 2026-08-22 against nautobot 3.2.4a0 and the release tooling in
`nautobot-app-device-onboarding` / `nautobot-app-ssot`.

## 7.0 The blunt answer about GitHub Packages

**GitHub Packages has no Python registry.** Its supported ecosystems are the Container
registry (`ghcr.io`), npm, NuGet, RubyGems, Apache Maven, and Gradle. There is no
PyPI-compatible endpoint, so `pip install --index-url https://.../pypi/...` against
GitHub Packages does not exist. Anything that implies otherwise is describing either a
container image or a third-party proxy.

So "push to my own GitHub Packages" resolves to one of three real things:

| # | Route | Literally GitHub Packages? | Installs onto *someone else's* Nautobot? | Effort |
|---|---|---|---|---|
| A | `pip install git+https://github.com/...@v0.1.0` | No | Yes | none |
| B | **GitHub Release with the wheel attached** | No (Releases, not Packages) | Yes | one workflow |
| C | Container image on `ghcr.io` | **Yes** | Only if they adopt your image | one workflow + Dockerfile |

**Recommendation: B for the app itself, C in addition only if you also want to hand
people a runnable Nautobot.** C is the only route that uses GitHub Packages proper, but
it ships a whole Nautobot, not a pluggable app — it cannot be installed *onto* an
existing Nautobot, it *replaces* it. If the goal is "install on any Nautobot", B is the
answer and C is a bonus deliverable.

If you want the ergonomics of a real index (`pip install nautobot-app-myipam` with no
URL), see 7.5 — a PEP 503 index on GitHub Pages. If the app will ever be public,
publishing to real PyPI is less work than all of this.

## 7.1 What "installable on any Nautobot" actually requires

A wheel is necessary but not sufficient. Installing an app is five steps, and only the
first is `pip`:

1. `pip install` the wheel **into the same environment as Nautobot** — the same
   virtualenv, or the same container image. An app installed elsewhere is invisible.
2. Add it to `PLUGINS` in `nautobot_config.py`:
   ```python
   PLUGINS = ["nautobot_myipam"]
   ```
   `PLUGINS = []` is the default (`nautobot/core/settings.py:213`). **There is no
   entry-point autodiscovery** — see the correction below. Installing the package alone
   does nothing.
3. Add `PLUGINS_CONFIG["nautobot_myipam"] = {...}` for any deployment-level settings.
4. Run `nautobot-server post_upgrade`
   (`nautobot/core/management/commands/post_upgrade.py`) — runs your migrations,
   collects your static files, refreshes job registration.
5. Restart **both** the `nautobot` web process **and** the `nautobot-worker`. Jobs
   execute in the worker; if the app is not installed there, the sync jobs will not run.

Your `min_version` / `max_version` on `NautobotAppConfig` are enforced at load
(`nautobot/extras/plugins/__init__.py:236-246`) and raise on mismatch, so be honest
about the range — a wrong `max_version` makes your app un-loadable on a host you did
not test.

> **Correction to Part 6 §6.10.** That section shows a
> `[tool.poetry.plugins."nautobot.app"]` entry point. Neither `nautobot-app-ssot`
> 4.6.2a0 nor `nautobot-app-device-onboarding` declares one, and nothing in
> `nautobot/core/` reads such an entry point. Discovery is the `PLUGINS` settings list,
> full stop. Declaring the entry point is harmless but does nothing; do not rely on it.

## 7.2 `pyproject.toml` for distribution

Modelled on `nautobot-app-ssot/pyproject.toml:1-30, 297-299`.

```toml
[tool.poetry]
name = "nautobot-app-myipam"          # the DIST name -> pip install nautobot-app-myipam
version = "0.1.0"                      # the tag will be v0.1.0
description = "Custom IPAM data source for Nautobot."
authors = ["Nikhil Mohite <nikhilmohite1993@gmail.com>"]
license = "Apache-2.0"
readme = "README.md"
homepage = "https://github.com/<you>/nautobot-app-myipam"
repository = "https://github.com/<you>/nautobot-app-myipam"
keywords = ["nautobot", "nautobot-app", "nautobot-plugin"]
classifiers = [
    "Intended Audience :: Developers",
    "Development Status :: 3 - Alpha",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
packages = [
    { include = "nautobot_myipam" },   # the IMPORT name -> PLUGINS = ["nautobot_myipam"]
]
include = [
    # Poetry excludes .gitignore'd files by default; built docs need an explicit include
    { path = "nautobot_myipam/static/nautobot_myipam/docs/**/*", format = ["sdist", "wheel"] }
]

[tool.poetry.dependencies]
python = ">=3.10,<3.14"
nautobot = ">=3.2.0,<4.0.0"            # RANGE, never a pin - see below
nautobot-ssot = ">=4.0.0,<5.0.0"       # only if you build on the SSoT framework

[build-system]
requires = ["poetry-core>=2.0.0,<3.0.0"]
build-backend = "poetry.core.masonry.api"
```

Three things that bite:

- **Never pin `nautobot` to an exact version.** The host already has Nautobot installed.
  A pin like `nautobot = "3.2.4a0"` makes pip either fail the resolve or *reinstall
  Nautobot underneath a running deployment*. A range dependency resolves as
  already-satisfied and touches nothing.
- **Dist name vs import name.** `nautobot-app-myipam` is what people `pip install`;
  `nautobot_myipam` is what goes in `PLUGINS`, becomes the Django `app_label`, and
  prefixes every permission string. Say both in your README — this is the single most
  common install failure.
- **`__version__ = metadata.version(__name__)`** (Part 6 §6.2) reads the *installed
  distribution* metadata. It raises `PackageNotFoundError` if the package is on
  `PYTHONPATH` without being pip-installed, which is exactly what a naive `COPY` into a
  Docker image produces. Always `pip install`, never copy the source tree.

Build and inspect locally before you automate anything:

```bash
poetry build                        # -> dist/nautobot_app_myipam-0.1.0-py3-none-any.whl
unzip -l dist/*.whl | head -40      # confirm templates/, static/, migrations/ are in there
```

If `migrations/`, `templates/nautobot_myipam/`, or `static/nautobot_myipam/` are missing
from that listing, they were `.gitignore`'d and Poetry dropped them — fix with `include`
before you ship.

## 7.3 Option A — install straight from git

Zero infrastructure. Good for pilots and for the Nautobot team to try your branch.

```bash
# public repo, pinned to a tag
pip install "git+https://github.com/<you>/nautobot-app-myipam@v0.1.0"

# private repo, using a fine-grained PAT with Contents: read
pip install "git+https://${GH_TOKEN}@github.com/<you>/nautobot-app-myipam@v0.1.0"
```

Why not for production: the target host needs `git` and network access to GitHub at
build time, the build backend gets installed on the host, and `@main` silently drifts.
Always pin a tag. Use this to validate, then move to B.

## 7.4 Option B — GitHub Release with the wheel attached (recommended)

This is what Network to Code do — `nautobot-app-device-onboarding/.github/workflows/release.yml`
has a `publish-github` job that uploads `dist/*` to the release, alongside its PyPI job.
Strip out the PyPI / Slack / post-release-PR parts and you get:

```yaml
# .github/workflows/release.yml
---
name: "Release"
on:
  push:
    tags: ["v*"]

jobs:
  build-and-release:
    runs-on: "ubuntu-latest"
    permissions:
      contents: "write"          # required to create the release and upload assets
    steps:
      - uses: "actions/checkout@v4"

      - uses: "actions/setup-python@v5"
        with:
          python-version: "3.12"

      - name: "Install Poetry"
        run: "pipx install poetry==2.1.3"

      - name: "Build"
        run: "poetry build"

      - name: "Check the tag matches the version in pyproject.toml"
        run: |
          if [ "${{ github.ref_name }}" != "v$(poetry version -s)" ]; then
            echo "Tag ${{ github.ref_name }} != v$(poetry version -s)"; exit 1
          fi

      - name: "Create release and upload the wheel + sdist"
        env:
          GH_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
        run: |
          gh release create "${{ github.ref_name }}" \
            --title "${{ github.ref_name }}" \
            --generate-notes \
            dist/*.whl dist/*.tar.gz
```

The tag-vs-version check is worth keeping; a release whose asset version disagrees with
its tag is a support problem forever.

Cutting a release:

```bash
poetry version 0.1.0
git commit -am "Release 0.1.0" && git tag v0.1.0 && git push --follow-tags
```

Installing on any Nautobot — **public repo**:

```bash
pip install https://github.com/<you>/nautobot-app-myipam/releases/download/v0.1.0/nautobot_app_myipam-0.1.0-py3-none-any.whl
```

**Private repo** — release assets are not anonymously downloadable, so fetch then
install (a plain `curl` of the browser URL returns HTML, not a wheel):

```bash
gh release download v0.1.0 --repo <you>/nautobot-app-myipam --pattern '*.whl' --dir /tmp
pip install /tmp/nautobot_app_myipam-0.1.0-py3-none-any.whl
```

Pin the wheel URL in the deployment's `requirements.txt` so rebuilds are reproducible.

## 7.5 Making plain `pip install nautobot-app-myipam` work

If you want the clean command without a URL, host a PEP 503 "simple" index on GitHub
Pages from the same repo. The layout is just directories and links:

```
docs-site/simple/index.html                       -> <a href="nautobot-app-myipam/">nautobot-app-myipam</a>
docs-site/simple/nautobot-app-myipam/index.html   -> <a href="https://.../nautobot_app_myipam-0.1.0-py3-none-any.whl">...</a>
```

Consumers then add two lines to `requirements.txt`:

```
--extra-index-url https://<you>.github.io/nautobot-app-myipam/simple/
nautobot-app-myipam==0.1.0
```

Generate it with `dumb-pypi` in the release workflow rather than by hand. Two caveats:
GitHub Pages is **public** even from a private repo, and `--extra-index-url` means pip
also searches PyPI — register the name on PyPI (even as a placeholder) so nobody can
squat it and shadow your package.

## 7.6 Option C — `ghcr.io`, the only literal GitHub Packages route

Publishes a Nautobot image with your app baked in. Genuinely GitHub Packages, and the
right shape for *your own* OpenShift deployment — but understand what it is: it does not
install onto an existing Nautobot, it is a replacement Nautobot.

```dockerfile
# Dockerfile
ARG NAUTOBOT_VER="3.2.4"
ARG PYTHON_VER="3.12"
FROM ghcr.io/nautobot/nautobot:${NAUTOBOT_VER}-py${PYTHON_VER}

USER 0
COPY dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl && rm -f /tmp/*.whl
COPY nautobot_config.py /opt/nautobot/nautobot_config.py
USER 999
```

(The upstream runtime base image is `ghcr.io/nautobot/nautobot`; the `-dev` variant used
by app repos for CI is `ghcr.io/nautobot/nautobot-dev` — see
`nautobot-app-device-onboarding/development/Dockerfile:16`. Ship from the non-dev one.)

```yaml
# add to .github/workflows/release.yml
  publish-ghcr:
    needs: "build-and-release"
    runs-on: "ubuntu-latest"
    permissions:
      contents: "read"
      packages: "write"          # this is what grants the ghcr.io push
    steps:
      - uses: "actions/checkout@v4"
      - uses: "docker/login-action@v3"
        with:
          registry: "ghcr.io"
          username: "${{ github.actor }}"
          password: "${{ secrets.GITHUB_TOKEN }}"
      - uses: "docker/build-push-action@v6"
        with:
          context: "."
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/nautobot-myipam:${{ github.ref_name }}
            ghcr.io/${{ github.repository_owner }}/nautobot-myipam:latest
```

The Docker build needs `dist/` present, so either run `poetry build` in this job too or
pass the wheel between jobs with `actions/upload-artifact` / `download-artifact` (which
is how the NTC workflow moves `distfiles` between its build and publish jobs).

Images are private by default; make the package public in the repo's Packages settings,
or have consumers `docker login ghcr.io` with a PAT carrying `read:packages`.

## 7.7 Versioning and compatibility

- **Tag `v<version>`, keep `pyproject.toml` in sync**, and let CI fail the release if
  they diverge (7.4). NTC bump to a prerelease (`poetry version prerelease` → `0.2.0a0`)
  immediately after each release so the development branch is never mistaken for a
  shipped version.
- **`min_version` / `max_version` are load-time hard failures**
  (`nautobot/extras/plugins/__init__.py:236-246`). Set `max_version = "3.99"` rather
  than `"3.2.99"` unless you actually know you break on 3.3 — an over-tight bound is
  worse than an over-loose one, because the operator cannot override it.
- **Document the compatibility matrix** in `docs/admin/` (app version → Nautobot version
  → Python version). Every NTC app has one and every operator reads it first.
- **Use towncrier** for release notes (`nautobot-app-ssot/pyproject.toml:301-310`) — one
  fragment per PR in `changes/`, assembled at release.

## 7.8 Checklist before the first release

- [ ] `poetry build` succeeds; `unzip -l dist/*.whl` shows `migrations/`,
      `templates/nautobot_myipam/`, `static/nautobot_myipam/`
- [ ] `nautobot` declared as a **range**, not a pin
- [ ] `min_version` / `max_version` reflect versions you have actually run against
- [ ] `migrations/__init__.py` exists and initial migrations are committed
- [ ] README states dist name **and** import name, plus the `PLUGINS` +
      `post_upgrade` + restart-the-worker steps
- [ ] Installed the wheel into a clean Nautobot container end-to-end — not just
      `invoke build` in the dev environment
- [ ] Release workflow verifies tag == `pyproject.toml` version
- [ ] Decided public-PyPI vs private-release-asset **before** the name is published;
      renaming later is a breaking change (Part 6 §6.0.1)
