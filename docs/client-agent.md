# Optional Paseo coding agent

`client_agent` installs the official OpenCode **1.14.46** Linux x64 baseline
binary, verified against the npm publisher's SHA512 integrity value. This
matches the `@opencode-ai/sdk` version the pinned Paseo source declares in its
server package. Paseo's native OpenCode provider discovers `/usr/local/bin/opencode`
through its existing service PATH. The baseline build avoids an AVX2 requirement.

The state is included by `client_apps.start`. With no `paseo.llm` pillar it is a
no-op. A present but incomplete LLM configuration fails and prevents Paseo from
starting with that configuration. No provider name or model alias is inferred.

Required optional pillar bundle:

```yaml
paseo:
  llm:
    key: <verified per-client LiteLLM virtual key>
    base_url: http://100.64.0.4:4000/v1
    models:
      - gpt-5.6-sol
    default_model: gpt-5.6-sol
    reasoning_effort: medium
    api_mode: responses
```

The URL is the client-accessible API base, including `/v1` when the proxy requires
it. It is passed unchanged. The control plane obtains this bundle from encrypted
instance credentials after verifying the issued key. Its management key is
never part of pillar or a client configuration.
HTTP is allowed only to literal IPv4 addresses in `100.64.0.0/10`, with traffic
encrypted by WireGuard. HTTPS remains available for other endpoints.

`default_model` selects an exact alias from the allowed `models` list for both
coding and auxiliary tasks. The demo selects `gpt-5.6-sol` with `medium` reasoning.
The selected model is marked as reasoning-capable and its OpenCode model options
include `reasoningEffort: medium`. Model options override OpenCode's auxiliary
reasoning defaults as well. Other allowed aliases remain selectable without
inheriting the selected model's reasoning setting. Existing sessions can retain
an explicit model or reasoning variant chosen by their user.

`api_mode: responses` selects the bundled `@ai-sdk/openai` **3.0.53** adapter.
Its `languageModel()` method sends requests to `/responses`, so the configured
base `http://100.64.0.4:4000/v1` becomes `/v1/responses`. OpenCode maps the model
option to the SDK's `openai` namespace; the request body contains
`reasoning: {effort: "medium"}`. Responses requests also set `store: false`.
The alias remains `gpt-5.6-sol`; LiteLLM owns its upstream model mapping.

`api_mode: chat_completions` keeps the bundled `@ai-sdk/openai-compatible`
adapter. Older pillar bundles without the three optional selection fields use
the first allowed model, `medium` reasoning, and Chat Completions. An unknown
transport, unsupported reasoning setting, or default outside the allowlist
fails validation before applying the agent configuration. Accepted reasoning
settings are `low`, `medium`, and `high`; actual model
support remains the responsibility of the configured LiteLLM alias.
The selected aliases must support the coding agent's tool calls; transport
compatibility alone does not establish model capability.

The virtual key is written to the mounted Paseo home at
`.config/opencode/litellm.key`, owned by `paseo`, mode `0600`. The config uses a
`{file:...}` reference, disables automatic upgrades and conversation sharing,
and enables only the configured provider. Config and key changes restart Paseo
on the next controlled Salt apply. Model restrictions and budgets are enforced
by the LiteLLM virtual key, not by a user-editable OpenCode configuration.

Removing pillar does not revoke a previously issued key or remove an existing
installation. Rotation/revocation needs an explicit control-plane workflow.
Issuing a key after initial infrastructure configuration also requires another
controlled application of `client_apps.start`; it does not restart a live
service merely by saving Odoo credentials. Once the previous configuration has
completed, a manager uses **Apply Updated Configuration** on the instance. This
creates a new Salt request generation while preserving its previous receipt;
an unfinished or unknown previous attempt cannot be bypassed with this action.

## Validation

`python3 -m unittest discover -s tests -p test_client_agent.py` checks absence,
malformed/partial secrets, configuration injection, allowed models, private
permissions, mount prerequisites and Paseo service ordering. Existing client-app
contracts are checked independently. Agent tests also cover explicit default
selection, reasoning, Responses transport, and invalid selection rejection.
A fixture using the exact bundled OpenAI SDK version verified that a custom
provider sends `/v1/responses` with the configured alias, medium reasoning, and
`store: false`. The fixture replaces HTTP transport and makes no inference call;
live LiteLLM and coding-session validation remain separate checks.

## Official references

- `packages/server/package.json` at the pinned `oduflow/paseo` commit
  identifies its OpenCode SDK dependency; the published package's
  `agent/providers/opencode/server-manager.js` resolves the `opencode` executable.
- [OpenCode baseline package 1.14.46](https://registry.npmjs.org/opencode-linux-x64-baseline/1.14.46)
  supplies the exact artifact and integrity value.
- [OpenCode custom providers](https://opencode.ai/docs/providers/#custom-provider)
  documents custom adapters and base URL/model options.
- [OpenCode configuration](https://opencode.ai/docs/config/)
  documents global config and file-backed credentials.
- [Pinned OpenCode config source](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/src/config/config.ts)
  and [provider source](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/src/provider/provider.ts)
  confirm that these interfaces exist in the installed version.

- [Pinned OpenCode provider configuration schema](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/src/config/provider.ts)
  defines the model `reasoning` flag and `options` map.
- [Pinned OpenCode session option merging](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/src/session/llm.ts)
  applies model options after main and auxiliary defaults.
- [Pinned OpenCode provider option mapping](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/src/provider/transform.ts)
  maps `@ai-sdk/openai` options to the SDK's `openai` namespace.
- [Bundled SDK versions](https://github.com/anomalyco/opencode/blob/v1.14.46/packages/opencode/package.json)
  and [OpenAI SDK 3.0.53 package](https://registry.npmjs.org/@ai-sdk%2fopenai/3.0.53)
  identify the exact adapter used by the installed OpenCode release.

## Paseo provider defaults

Client Paseo configuration enables only OpenCode. Claude, Codex, Copilot, Pi,
and Oh My Pi are disabled through `agents.providers.<id>.enabled`; installing
another executable does not enable its provider. The model picker is separate:
OpenCode receives the client's verified LiteLLM model allowlist and default
model, so enabling OpenCode does not expose every model known to Paseo.

## Client Oduflow MCP

OpenCode's global configuration connects to the client's HTTPS `/mcp` endpoint.
Its Authorization header references `{env:ODUFLOW_MCP_TOKEN}`; the JSON contains
no token. Salt supplies the client's existing team token through Paseo's
root-owned, mode-0600 `/etc/paseo/credentials.env`. This grants agents management
access to all environments within that client's Oduflow team. It does not grant
access to the platform control plane.

Credential rotation updates this environment variable together with Oduflow's
authentication token and the Paseo password. Paseo and newly launched agent
processes must load the new environment; editing the file alone does not update
an already-running process. The MCP config lives outside client repositories.

### Optional Codex and Claude Code clients

`client_agent.cli`, included by `client_apps.install`, installs the pinned official
Codex 0.154.0 and Claude Code 2.1.270 Linux amd64 native binaries with SHA-512
verification. This credential-free stage also runs during the next client image
build. Updating Salt alone does not rebuild an existing published image.

`client_agent.mcp` configures the Paseo user's `~/.codex/config.toml` and
`~/.claude.json` during provisioning, preserving unrelated settings and MCP
servers. Both connect to `https://<client-oduflow-host>/mcp` and resolve
`ODUFLOW_MCP_TOKEN` from the Paseo service environment. Tokens are not written to
these files. Codex uses `bearer_token_env_var`; Claude uses `${ODUFLOW_MCP_TOKEN}`
in its HTTP authorization header. Existing running agent sessions must be
recreated to pick up configuration changes. Direct SSH sessions do not inherit
the Paseo service environment automatically.

These CLIs remain disabled in Paseo by default; only OpenCode is enabled.
Authentication to OpenAI or Anthropic for model inference is separate from
client Oduflow MCP authentication and is not provisioned by these states.
