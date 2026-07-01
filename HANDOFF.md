# Handoff: prepare `scrapy-feedexporter-azure-storage` for first PyPI release

This file captures everything needed to make this repo publishable on PyPI.
Implement it as a single PR. The PyPI-side steps (trusted publishing) are
handled separately by the maintainer and are listed at the end for reference.

A quick way to use this with Claude Code:
> "Read HANDOFF.md and implement it. Use Plan mode and show me the diffs
> before applying. Don't do anything that needs PyPI access."

---

## Goal

The package has never been published. Prepare it for a clean first release:
modern packaging, Python 3.10+, a changelog, bump-my-version config, and a
tag-triggered PyPI publish workflow using trusted publishing.

## Current state (for reference)

- Flat layout: package `scrapy_azure_exporter/` and `tests/` at the repo root.
- `setup.py` (dist name `scrapy_azure_exporter`, version `0.0.2`,
  `python_requires>=3.8`, deps `azure-storage-blob`, `scrapy`).
- `tox.ini` with envs `py39,py310,mypy,isort,black,flake8`.
- `README.md` (says "Python 3.8+").
- No `pyproject.toml`, no changelog, no bumpversion config,
  no `.github/workflows`, no `LICENSE`, no releases.

## Decisions to confirm before starting

1. **Distribution name** → use `scrapy-feedexporter-azure-storage` (matches the
   repo and sibling plugins). The *import* package stays `scrapy_azure_exporter`,
   so there are **no Python code changes** — only packaging metadata.
2. **License** → use **Apache-2.0** for new projects.
3. **Initial public version** → use **0.1.0**.

---

## Step 1 — Add `pyproject.toml` (replacing `setup.py`)

```toml
[build-system]
requires = ["hatchling>=1.27.0"]
build-backend = "hatchling.build"

[project]
name = "scrapy-feedexporter-azure-storage"
version = "0.1.0"
description = "Scrapy Feed Exporter for Azure Storage"
readme = "README.md"
requires-python = ">=3.10"
license = "Apache-2.0"
license-files = ["LICENSE"]
authors = [{ name = "Zyte", email = "opensource@zyte.com" }]
dependencies = [
    "azure-storage-blob",
    "scrapy",
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Framework :: Scrapy",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Operating System :: OS Independent",
    "Programming Language :: Python",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Programming Language :: Python :: 3.14",
    "Topic :: Internet :: WWW/HTTP",
    "Topic :: Software Development :: Libraries :: Python Modules",
]

[project.urls]
Homepage = "https://github.com/scrapy-plugins/scrapy-feedexporter-azure-storage"
Source = "https://github.com/scrapy-plugins/scrapy-feedexporter-azure-storage"
Issues = "https://github.com/scrapy-plugins/scrapy-feedexporter-azure-storage/issues"

[tool.hatch.build.targets.wheel]
packages = ["scrapy_azure_exporter"]

[tool.hatch.build.targets.sdist]
include = [
    "/scrapy_azure_exporter",
    "/tests",
    "/README.md",
    "/CHANGES.rst",
    "/LICENSE",
]
```

Notes:
- Use hatchling for new projects.
- Add explicit wheel/sdist targets so the wheel contains only
  `scrapy_azure_exporter`, while sdist includes tests and metadata files.

Then **delete `setup.py`**.

## Step 2 — Add a `LICENSE` file

Standard Apache License 2.0 text. Required by the
`license-files` reference in `pyproject.toml`.

## Step 3 — Raise the Python floor to 3.10 everywhere

- `requires-python = ">=3.10"` (done in Step 1).
- `tox.ini`:
  ```ini
  [tox]
  envlist = py310,py311,py312,py313,py314,mypy,isort,black,flake8
  ```
- `README.md`: change "Python 3.8+" to "Python 3.10+". After the first release,
  switch the install line to `pip install scrapy-feedexporter-azure-storage`.

## Step 4 — Add a changelog (`CHANGES.rst`)

```rst
=========
Changelog
=========

0.1.0 (unreleased)
------------------

* Initial public version.
```

(Markdown is fine too, but `.rst` matches the scrapy/zyte ecosystem.)

## Step 5 — Add bump-my-version config (in `pyproject.toml`)

```toml
[tool.bumpversion]
commit = true
tag = true
tag_name = "{new_version}"

[[tool.bumpversion.files]]
filename = "pyproject.toml"
search = 'version = "{current_version}"'
replace = 'version = "{new_version}"'

[[tool.bumpversion.files]]
filename = "CHANGES.rst"
search = '{current_version} (unreleased)'
replace = '{current_version} ({now:%Y-%m-%d})'
```

Notes:
- Keep `tag_name` bare (`{new_version}`, no `v` prefix) to match the workflow trigger.
- `allow_dirty = false` is the default; no need to set it explicitly.
- `current_version` can be omitted when `project.version` is statically set.

## Step 6 — Add `.github/workflows/publish.yml`

Mirrors the scrapinghub reference; the only change is the project name in
`environment.url`.

```yaml
name: Publish
on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+'
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-python@v6
        with:
          python-version: "3.14"
      - run: |
          python -m pip install --upgrade pip
          pip install --upgrade build twine
          python -m build
          twine check dist/*
      - uses: actions/upload-artifact@v6
        with:
          name: packages
          path: dist/
          if-no-files-found: error
          compression-level: 0
  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    environment:
      name: pypi
      url: https://pypi.org/p/scrapy-feedexporter-azure-storage
    steps:
      - uses: actions/download-artifact@v7
        with:
          name: packages
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          print-hash: true
          verbose: true
```

The `id-token: write` permission with no API token is what makes this trusted
publishing (OIDC). The `environment: pypi` ties into the PyPI publisher config
and the GitHub environment.

## Step 7 — Verify locally before opening the PR

```bash
python -m build
twine check dist/*
```

Both should pass. Confirm the built wheel/sdist use the name
`scrapy_feedexporter_azure_storage` (normalized from the dist name) and contain
the `scrapy_azure_exporter` package in wheels, and include `tests` in the sdist.

---

## Maintainer / PyPI side (not part of the code PR)

- Register a **pending publisher** on PyPI before the first release:
  - Project name: `scrapy-feedexporter-azure-storage`
  - Owner: `scrapy-plugins`
  - Repository: `scrapy-feedexporter-azure-storage`
  - Workflow filename: `publish.yml`
  - Environment: `pypi`
- Create the matching **`pypi` environment** under the repo's GitHub settings.

## Release flow (after the PR merges)

1. **First release:** tag the existing version directly:
   ```bash
  git tag 0.1.0 && git push origin 0.1.0
   ```
2. **Subsequent releases:**
   ```bash
   pip install bump-my-version
   bump-my-version bump patch   # or minor / major
   git push --follow-tags
   ```
3. The pushed tag triggers `publish.yml`, which builds, runs `twine check`, and
   publishes via trusted publishing.

## Out of scope (suggested follow-up)

There's no tests/CI workflow, only `tox.ini`. A separate `tests.yml` running the
tox matrix on PRs would be the natural companion to this work.
