# Control Odoo settings and secrets

Control-plane configuration is split by sensitivity. Tokens, passwords and the
encryption key are supplied through the process environment. Non-secret
identifiers, URLs, hostnames and feature choices are stored in **Oduflow →
Configuration → Settings**.

## Install or update the control addons

Use the selected Oduflow MCP environment and configured delivery repository and
branch. A fresh control environment uses Odoo 19 with no template; the base
installation needs `oduflow`, the selected provider adapter (currently
`oduflow_vultr`) and enabled integrations such as `oduflow_cloudflare`.
Include this repository's `addons/` in its addon path and the pinned dependencies
from `.oduflow/requirements.txt`.

Commit and push reviewed source, then deliver it with `pull_and_apply`. Schema,
ACL, data and dependency changes require module upgrade; restarting alone is
insufficient. Never edit addon source inside an immutable runtime checkout.
Existing predecessor installations follow [namespace migration](brand-migration.md).
The [stack guide](container-services.md) covers supporting services; verify the
[queue runner](queue-and-notifications.md) before provisioning clients.

## Environment secrets

Set only the credentials required by the enabled providers and integrations:

```ini
ODUFLOW_ENCRYPTION_KEY=<Fernet key>
ODUFLOW_VULTR_API_KEY=<Vultr API token>
ODUFLOW_CLOUDFLARE_TOKEN=<Cloudflare DNS token>
ODUFLOW_CLOUDFLARE_R2_TOKEN=<Cloudflare R2 provisioner token>
ODUFLOW_GITHUB_TOKEN=<repository management token>
ODUFLOW_GITHUB_CLIENT_TOKEN=<legacy explicit HTTPS project token, if required>
ODUFLOW_LITELLM_MANAGEMENT_KEY=<LiteLLM management key>
ODUFLOW_PILLAR_TOKEN=<external pillar bearer token>
ODUFLOW_SALT_API_PASSWORD=<Salt API password>
```

New clients use scoped SSH project keys and public software downloads; see
[repository access](github-download-access.md). `ODUFLOW_GITHUB_CLIENT_TOKEN` is
only a legacy explicit HTTPS option and is unnecessary when the client already
has its verified encrypted Git credential bundle. LiteLLM and provider credentials are
unnecessary when their integration is disabled. Preserve
`ODUFLOW_ENCRYPTION_KEY` separately from database backups: a database restore
cannot decrypt existing client secrets without the same key.

## Oduflow Settings

The settings form stores these non-secret values as Odoo system parameters:

| Section | Setting | Parameter |
| --- | --- | --- |
| Control Plane | Public Access Base URL | `oduflow.access_base_url` |
| Control Plane | GitHub Organization | `oduflow.github_org` |
| Cloudflare DNS | Cloudflare Zone ID | `oduflow.cloudflare_zone_id` |
| Cloudflare R2 Backup | Cloudflare Account ID | `oduflow.cloudflare_account_id` |
| Salt | Salt API URL | `oduflow.salt_api_url` |
| Salt | Salt API User | `oduflow.salt_api_user` |
| Salt | Salt Master VPN IP | `oduflow.salt_master_ip` |
| Salt | Salt SOCKS Proxy | `oduflow.salt_proxy` |
| Salt | Salt TLS Hostname | `oduflow.salt_tls_hostname` |
| Salt | Pillar Gateway Host | `oduflow.pillar_gateway_host` |
| LiteLLM | Management URL | `oduflow.litellm_api_url` |
| LiteLLM | Client URL | `oduflow.litellm_client_url` |
| LiteLLM | SOCKS Proxy | `oduflow.litellm_proxy` |
| LiteLLM | Models | `oduflow.litellm_models` |
| LiteLLM | Default Model | `oduflow.litellm_default_model` |
| LiteLLM | Reasoning Effort | `oduflow.litellm_reasoning_effort` |
| LiteLLM | API Mode | `oduflow.litellm_api_mode` |

The GitHub organization defaults to `oduflow`, the Salt API user to
`oduflow-api`, the Salt master address to `100.64.0.1`, LiteLLM reasoning to
`medium`, and the LiteLLM API mode to `chat_completions`. An empty LiteLLM
management URL disables automatic LiteLLM key provisioning. An empty public
access URL falls back to Odoo's `web.base.url`.

## Upgrade from environment configuration

The `oduflow` and `oduflow_cloudflare` upgrade migrations copy existing
non-secret `ODUFLOW_*` values into empty Oduflow settings. Existing database
settings take precedence and are never overwritten. Upgrade both modules while
the old environment is still present, verify the settings and a real queued
operation, and only then remove the migrated non-secret variables from the
container environment.
