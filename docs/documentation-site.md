# Documentation site

**https://docs.oduflow.sh** is built with MkDocs Material and published from the
public [oduflow-platform-docs repository](https://github.com/oduflow/oduflow-platform-docs).
The platform application's repository remains private.

For content ownership, review rules and translation checks, see
[documentation maintenance](documentation-maintenance.md).

## Publish an update

In the platform checkout, update the guides in `docs/` and add new pages to
`mkdocs.yml`. Invoke the **publish-docs** skill (`/publish-docs` or `$publish-docs`
in Codex). It performs the complete publication:

1. Synchronizes the current documentation and theme into the public repository.
2. Installs the pinned dependencies and validates the site with
   `python -m mkdocs build --strict` in the public checkout.
3. Commits changed documentation and pushes it to `main`.
4. Waits for the public repository's GitHub Actions build and Pages deployment.
5. Checks HTTPS and the published content digest at `/publish.json`.

An unchanged publication does not create another commit. If publication fails,
the skill reports the failed step and preserves the previous successful site.

## Source ownership

The platform checkout owns `docs/`, `mkdocs.yml`, and `requirements-docs.txt`.
The `publish-docs` skill provides the public README, ignore rules, and
`.github/workflows/docs.yml`. Its synchronizer maintains `.publish-docs.json`
in the public repository to track transferred files and removed pages.

Make synchronized file changes in the platform checkout. The synchronizer stops
if a public file has changed independently, so those changes can be reconciled
before publication.

The logo, favicon, and custom CSS originate from the Oduflow tool's documentation
site. The content and navigation describe the platform. Application code,
environment files, and deployment reports are not synchronized. Links to private
repository files still require repository access.

## Local preview

In either checkout, with Python 3.12 and virtual environment support:

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-docs.txt
.venv/bin/python -m mkdocs serve
```

Open `http://127.0.0.1:8000`. For a strict build:

```sh
.venv/bin/python -m mkdocs build --strict
```

The generated `site/` directory is ignored by Git. A strict build checks navigation
and internal links; it does not publish the site or verify deployed applications.

## Hosting

The public repository uses GitHub Pages with **GitHub Actions** as its publishing
source, custom domain `docs.oduflow.sh`, and branch `main`. Pull requests build
without deploying; pushes affecting the site build and publish automatically.
The **Documentation** workflow also supports manual publication from `main`.

DNS uses a CNAME from `docs.oduflow.sh` to `oduflow.github.io`. The publisher
checks the Pages setup and can complete initial DNS and HTTPS configuration with
the existing authorized platform access. DNS propagation and certificate issuance
must finish before HTTPS verification can pass.
