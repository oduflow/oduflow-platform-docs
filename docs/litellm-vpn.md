# LiteLLM over the private Tailscale network

The operator manages the LiteLLM host. It is registered with Headscale as
`litellm`, tagged `tag:llm`, with VPN address **100.64.0.4**. The selected API
endpoint is **http://100.64.0.4:4000**. WireGuard encrypts the traffic between VPN
peers; a separate HTTPS listener and DNS record are not required for this setup.

## Host configuration

Run LiteLLM on port 4000 and make that port reachable on the host's Tailscale
interface. When publishing a Docker port, bind it to the VPN address, for example
`100.64.0.4:4000:4000`. Arrange startup after Tailscale has restored the address.
Retain LiteLLM authentication. Do not publish the management key in client
configuration or disable authentication because the transport is private.

The deployed Headscale policy permits `tag:client` and `tag:odoo` to connect to
`tag:llm` on TCP 4000. Client-to-client access remains denied. Inference and
management use the same listener, so HTTP endpoint authorization is enforced by
LiteLLM keys rather than separate network ports:

| Credential | Purpose | Destination |
| --- | --- | --- |
| Headscale enrollment key | Register one host in the VPN | Tailscale on that host |
| LiteLLM management key | Create and inspect client keys | Control Odoo only |
| LiteLLM client key | Call approved models within its budget | That client's VM |

Client keys are issued as `llm_api` keys. The control plane verifies their
`allowed_routes`, explicit model allowlist, budget, and ownership metadata before
publishing them through Salt. Check actual denial of management endpoints using
a client key when validating a new LiteLLM deployment.

## Register a new LiteLLM host

Install and start Tailscale first. Then create an enrollment in **Oduflow →
Private Network → Server Enrollments**, issue a key, and open its connection
instructions. The default key is single-use, non-ephemeral, and valid for an
initial registration within 30 minutes. Save it to `/run/headscale.authkey`, owned
by root with mode 0600, and run as root on the target host:

```sh
tailscale up \
  --login-server=https://headscale.example.com \
  --auth-key=file:/run/headscale.authkey \
  --hostname=litellm \
  --accept-dns=false
tailscale ip -4
rm /run/headscale.authkey
```

Remove the key file after successful registration. Preserve Tailscale's state
across restarts. The Headscale coordination endpoint still uses HTTPS; the
application API inside WireGuard uses HTTP. An alternative root command on
the coordination server, issued only when the target host is ready, is:

```sh
umask 077
headscale preauthkeys create --tags tag:llm --expiration 30m \
  --output json > /run/litellm-enrollment.json
```

Transfer only the `key` value through a private file and remove the temporary
JSON afterward. Use either Odoo or the CLI for one enrollment, not both.

## Control Odoo configuration

Configure the non-secret values in **Oduflow → Configuration → Settings →
LiteLLM**:

```ini
Management URL: http://100.64.0.4:4000
Client URL: http://100.64.0.4:4000/v1
SOCKS Proxy: socks5h://oduflow-1-svc-oduflow-vpn:1055
Models: ["gpt-5.6-sol"]
Default Model: gpt-5.6-sol
Reasoning Effort: medium
API Mode: responses
```

Use actual configured model aliases. The management credential remains in the
`ODUFLOW_LITELLM_MANAGEMENT_KEY` environment secret. The management URL has no `/v1` suffix because
its adapter calls `/key/generate` and `/key/info`. The client base URL includes
`/v1` and is passed unchanged to OpenCode. The selected `gpt-5.6-sol` alias uses
the Responses API and medium reasoning. Other deployments may explicitly select
`chat_completions` for a compatible model. Model choice and reasoning settings
are frozen with the encrypted provisioning configuration and included in Salt
pillar only after the client key has been verified.

The management virtual key must belong to a service user with the `proxy_admin`
role. Keep its allowed routes restricted to management operations. The
`management_routes` allowlist alone does not grant that role: a key without an
owning user can read some metadata while `/key/generate` still returns 401.
Changing a management key's owner in LiteLLM preserves the key fingerprint; do
not replace an in-flight candidate or clear its durable dispatch receipt merely
to retry an ambiguous creation.

Control Odoo needs the explicit SOCKS proxy because its VPN gateway uses userspace
networking. Client VMs and their Paseo/OpenCode processes use their host's ordinary
Tailscale routes. They need no additional VPN identity per agent.

Client URL validation permits HTTP only for literal IPv4 addresses inside
`100.64.0.0/10`. Public addresses, arbitrary HTTP hostnames and credentials in URLs
remain invalid. HTTPS remains supported for other deployments.

Apply environment changes through Megaflow `update_environment`. Issue a client
key, apply updated Salt configuration, then verify inference and a coding-agent
session before marking the complete client workflow successful.

References: [Headscale registration](https://headscale.net/stable/ref/registration/),
[Tailscale connections](https://tailscale.com/docs/reference/connection-types),
[LiteLLM virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys).

## Recover an unresolved candidate after management configuration changes

An Oduflow Admin can enqueue `action_recover_llm_key()` on a prepared client.
This explicit recovery applies only to an unpublished candidate with an existing
unresolved dispatch receipt. It retains the original encrypted configuration and
receipt, keeps exactly the same candidate key, and creates a separate recovery
receipt. Changing the API endpoint is refused. No provider request runs inside
the HTTP action.

The queued job first looks up the candidate by its SHA256 hash. If absent, it
checks `/openapi.json` for the reviewed LiteLLM **1.96.2** implementation before
allowing one create request. In that version, the caller's token hash is the
primary key and creation upserts with `update={}`. This prevents a second virtual
key row for the same candidate; it does not promise that ancillary event hooks
run only once. An alias conflict or lost POST response is reconciled with an
exact-key GET. Re-running recovery after a claimed request performs reconciliation
only and never issues a second POST.

The returned key must match the complete ownership metadata, alias, budget,
model allowlist, inference-only routes and expiration policy before the new
configuration becomes canonical or its key enters Salt pillar. An existing key
with an old model policy fails closed; recovery does not silently edit its policy
or reset spend. The original dispatch remains unchanged, including when the new
receipt succeeds. A different provider version requires a fresh implementation
review rather than weakening this gate.

Implementation evidence: [LiteLLM 1.96.2 token upsert](https://github.com/BerriAI/litellm/blob/v1.96.2/litellm/proxy/utils.py),
[caller key and alias handling](https://github.com/BerriAI/litellm/blob/v1.96.2/litellm/proxy/management_endpoints/key_management_endpoints.py),
[token primary key](https://github.com/BerriAI/litellm/blob/v1.96.2/schema.prisma).
