# Oduflow Platform Documentation

Public documentation for **https://docs.oduflow.sh**.

This repository contains the documentation site. The platform application source
is maintained separately in the private `oduflow/oduflow-platform` repository.

## Publish

Maintain the English documentation and MkDocs configuration in the platform
checkout, then invoke the **publish-docs** skill (`/publish-docs` or
`$publish-docs` in Codex). It synchronizes the documentation here, validates the
build, commits and pushes to `main`, waits for GitHub Pages, and verifies the
published content. No application repository access is needed by GitHub Actions.

The `.publish-docs.json` manifest records synchronized files. Change those files
in the platform checkout so the next publication includes your changes.

## Local preview

With Python 3.12 and virtual environment support:

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-docs.txt
.venv/bin/python -m mkdocs serve
```

## Build and deployment

```sh
.venv/bin/python -m mkdocs build --strict
```

The **Documentation** GitHub Actions workflow checks pull requests, builds pushes
to `main`, and deploys the resulting `site/` artifact to GitHub Pages. A manual
workflow run can republish `main` without a content change.

See [site maintenance](docs/documentation-site.md) for details.
