# Maintaining the documentation

Use English for source documentation. Write for the person performing the task:
state the purpose and prerequisites, give the procedure, then explain how to
verify its result and recover from the relevant failure.

## Document ownership

| Surface | Owns | Update rule |
| --- | --- | --- |
| Root README | Repository entry point, source map and contributor checks | Link to guides for operational details |
| `docs/` | Public architecture, procedures and technical contracts | Update with the behavior; keep existing page URLs where possible |
| `addons/<module>/README.md` | Module setup, dependencies and module-specific limits | Link to shared platform procedures |
| `addons/<module>/doc/` | Installed Odubook guides and daily changes | Keep configured translations synchronized |
| Standalone Manuals | Uploaded documents, including the integrator guide | Update the existing language file; `noupdate` seeds preserve administrator edits |
| `client/README.md` | Client repository installation, maintenance and checks | Follow the client repository's contribution rules |
| `docs/decisions/` | Architectural rationale and historical design boundaries | State accepted/superseded scope and link to the current contract |
| `reports/deployments/` | Dated evidence for a named target and revision | Preserve historical facts; do not turn a past check into a current guarantee |

## Keep one detailed explanation

An overview may summarize a contract, but it should link to the page that owns
the details. Use these primary references when extending related guides:

| Topic | Primary reference |
| --- | --- |
| Components and vocabulary | [Architecture](spec.md), [terminology](glossary.md) |
| Client revision selection and release delivery | [Client releases](client-releases.md) |
| DNS, namespace and TLS | [DNS and certificates](cloudflare.md) |
| Application state composition | [Client applications](client-apps.md) |
| IDE method, archive verification and hostname migration | [IDE installation](ide-installation.md) |
| Git permissions and credential migration | [Repository access](github-download-access.md) |
| Production creation and publication | [Production bootstrap](client-production.md) |
| Access grants and password changes | [Client access](client-access.md) |
| Backup coverage | [Client backup](client-backup.md) |
| Salt execution status and durable evidence | [Asynchronous execution](salt-async-apply.md), [receipt protocol](salt-result-recovery.md) |
| Gateway runtime and metering | [Managed LiteLLM](litellm-managed.md) |
| Site build and publication | [Documentation site](documentation-site.md) |

Keep package versions and checksums in their executable manifests. Mention an
exact version in prose when it explains compatibility or dated verification;
identify which of those it means. Do not copy a demo IP, source SHA, installed
version or test count into a general requirement.

## Editing and checking

1. Inspect the working tree and record a Git base. Preserve unrelated work.
2. Compare the affected prose with the source contract and related guides. State
   the working directory for commands, including a switch into `client/`.
3. Distinguish current behavior, historical evidence and proposed work. Use **IDE**
   for the product label and preserve technical `paseo` identifiers.
4. Add new public pages to `mkdocs.yml`; use relative `.md` links inside the site.
   Repository-only material needs an explicit repository link and may require
   private repository access.
5. Run `python -m mkdocs build --strict` with the pinned documentation dependencies.
   Check the affected pages at desktop/mobile widths and close the browser session.
6. Review the diff for lost prerequisites, security/ownership guards, broken paths
   and unjustified claims of successful deployment.

For module guides, read the repository's `LANG.md`, `LANG.local.md` and
`odubook-i18n` skill. Synchronize English, Polish and Russian where configured,
add one daily change entry, and run the checker normally and against the
pre-change Git revision. Technical specifications remain source-only. Marker
updates follow translation work, not the other way around.

## Publication and evidence

A local edit or strict build is a **source** result. Publishing the site is a
separate operation through [publish-docs](documentation-site.md#publish-an-update).
Installing an addon and refreshing an uploaded Manual are separate operations
again. State exactly which surface was updated and verified.

Deployment evidence should identify the target, source revision, actual deployed
revision, checks performed and remaining limits. A receipt about a previous
deployment does not certify current health. Source-only audits belong under
`reports/` and must not imply runtime verification.
