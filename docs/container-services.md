# Native services and the Oduflow platform stack

The platform stack runs the Odoo control plane, Salt Master/API, LiteLLM/exporter
and a private VPN gateway. An optional Headscale service supplies the coordination
server; alternatively, configure an external Headscale installation. Pass `--ssh-image <image@sha256:digest>` to add an administrative SSH gateway and separate certificate signer as described in
[administrative SSH](administrative-ssh.md).

The source of the manifest is `scripts/render-platform-stack.py`. All addresses,
team names and credentials are deployment inputs. Operational identities and live
results belong in `reports/deployments/`, not in this guide.

## Native application release

`litellm.native_packages` installs `litellm-1.100.1-r1` into
`/opt/oduflow-litellm/releases/`. Python 3.12 wheels, including LiteLLM 1.100.1,
psycopg 3.2.10 and Prisma 0.15.0, are pinned with SHA256 hashes in
`salt/states/litellm/files/native-requirements.txt`. The installer generates the
Prisma client from the schema in that pinned wheel. Node and OS libraries are
installed by Salt. Prisma engine generation still downloads the engines selected
by the pinned Prisma release; wheel hashes alone do not attest those binaries.

The installer records the requirements hash and refuses an unknown/incomplete
existing release directory. Inspect a failed installation before retrying; never
remove a directory used by a running gateway. A changed lock needs a new release
identifier and runtime qualification. The Packer LiteLLM template now prepares
these same native packages; previously published snapshots remain unchanged.

New gateway records default to native mode and an external database DSN. Supply
the Oduflow service database URL through the existing encrypted credentials API.
Existing records remain in Docker mode after module upgrade, and old immutable
configuration revisions still replay through their original runtime contract.
Switching an existing installation or moving its database/ledger is a separate,
explicit migration. Image rollback does not roll back PostgreSQL schema changes.

## Images

Choose a reviewed release manifest from `docs/releases/` and verify its source
revision and CI results. Release digests identify immutable artifacts; later
source changes are not included in an earlier image. Record the selected digests
in the deployment report.

Build from the repository root, using the exact reviewed commit:

```sh
docker build -f docker/master/Dockerfile -t oduflow-master:test .
docker build -f docker/litellm/Dockerfile -t oduflow-litellm:test .
docker build -f docker/vpn/Dockerfile -t oduflow-vpn:test .
```

The `Container service qualification` workflow can publish all three images to
the repository's GitHub Container Registry packages after its lifecycle tests:

```sh
gh workflow run container-services.yml \
  --ref "$DELIVERY_BRANCH" -f publish=true
```

Normal push runs only qualify the images. Publication resolves the Ubuntu and
Python base tags to digests, records those digests, and pushes the exact tested
images under `ghcr.io/oduflow/oduflow-platform/{master,litellm,vpn}`. Unique tags
contain the full source SHA, workflow run ID and attempt. No `latest` tag is used.
The workflow pulls the published digests and verifies their source revision
labels; its release artifact contains `images.txt` and `base-images.txt`.
Use the digest references for deployment. Registry access follows package access
permissions; publication does not make the private repository or packages public.

After authenticating the deployment host to GHCR with package read access,
generate a stack pinned to this release (the example uses `jq`):

```sh
release="$RELEASE_MANIFEST"
python3 scripts/render-platform-stack.py \
  --master-image "$(jq -r .images.master "$release")" \
  --litellm-image "$(jq -r .images.litellm "$release")" \
  --vpn-image "$(jq -r .images.vpn "$release")" \
  --with-enrollment \
  --output docker/platform-stack.json
```
`.dockerignore` excludes protected inputs from the build context.
The VPN image reuses the SHA256-verified Tailscale 1.102.4 static installer;
it does not depend on a matching upstream Docker tag being published.

Master and LiteLLM run systemd as PID 1. No Docker socket or nested Docker engine
is required. `SIGRTMIN+3` starts the ordered halt sequence; the container-only
halt unit overrides `SuccessAction=exit-force` so PID 1 exits normally after
services stop, including when the qualification profile grants `CAP_SYS_BOOT`.
See the upstream [halt unit](https://github.com/systemd/systemd/blob/v255/units/systemd-halt.service)
and [shutdown implementation](https://github.com/systemd/systemd/blob/v255/src/shutdown/shutdown.c).
LiteLLM boot applies the same Salt role as the VM using a container
storage guard. The volume must be mounted and either empty or match the exact
metering source/incarnation; unknown contents are refused. Master PKI and reduced
dispatch receipts each have persistent volumes; Salt job/pillar caches stay off.

## Stack and protected inputs

Use Oduflow with merged [PR #229](https://github.com/oduist/oduflow/pull/229), which
adds `runtime` settings to services and stacks. Generate a manifest with published image
digests (JSON is valid YAML), then validate and review the plan:

```sh
python3 scripts/render-platform-stack.py \
  --master-image "$MASTER_IMAGE" --litellm-image "$LITELLM_IMAGE" \
  --vpn-image "$VPN_IMAGE" --with-enrollment --output docker/platform-stack.json
oduflow stack validate docker/platform-stack.json
oduflow stack plan docker/platform-stack.json --team 1
oduflow stack apply docker/platform-stack.json --team 1
```

Use an isolated team/test host. The generated qualification profile explicitly
sets `privileged: true` for the two systemd containers, a private cgroup namespace,
tmpfs runtime mounts and a 240-second stop timeout. It is not a proven production
security profile. Oduflow never implicitly enables privileged mode from `runtime`.
The VPN service needs no capabilities, host networking or privileged mode.

The manifest creates one Odoo environment and one **service database** for LiteLLM
inside Oduflow's existing PostgreSQL. It never creates another PostgreSQL server.
Services use unused port 65535 for the current mandatory exposure field; no process
listens there. Inference, Salt API and Salt transport are reachable through the
private gateway, not public Traefik routes.

Before applying, prepare the local `docker/protected/` input files referenced by
the manifest. Protect their directory and files with modes 0700/0600. They are
gitignored and must never be committed or printed. Master inputs include:

- `oduflow-container-settings.json`: `master_uuid`, `odoo_url` (the private TLS
  pillar gateway, e.g. `https://oduflow-1-svc-vpn:8443`), `headscale_url` and
  `headscale_api_url` (the existing Headscale server's management endpoint).
  When those endpoints are on the VPN, set `outbound_proxy` to the VPN service's
  internal HTTP proxy, e.g. `http://oduflow-1-svc-vpn:1056`. Both external pillar
  and Headscale calls use this explicit proxy; environment proxies stay ignored.
- `oduflow-api.htpasswd`, `oduflow-pillar.token`, `oduflow-headscale-api.key`.
- `pki/api/server.crt` and `server.key` for Salt API. The certificate must match
  the hostname Odoo verifies when connecting through the VPN gateway.
- `oduflow-pillar-ca.crt`, the CA for the private pillar proxy. Its leaf certificate
  must match the configured pillar hostname or VPN IP in its SAN.

VPN inputs include `gateway.json`, the one-use `enrollment.key`, and pillar proxy
`server.crt`/`server.key`. A non-secret gateway configuration looks like:

```json
{
  "headscale_url": "https://headscale.example.org",
  "hostname": "platform-services",
  "pillar": true,
  "services": {
    "4000": "oduflow-1-svc-litellm",
    "4505": "oduflow-1-svc-master",
    "4506": "oduflow-1-svc-master",
    "8000": "oduflow-1-svc-master"
  }
}
```

For the first VPN boot, add `--with-enrollment` to the generator command. Once
the gateway reports a running, persisted VPN identity, regenerate the manifest
without that option before subsequent applies. The durable manifest excludes
`enrollment.key` so reconciliation cannot restore a consumed enrollment secret.

When using an external Headscale, configure its coordination and management
endpoints before starting the VPN service. Configure its ACLs so client nodes can reach inference and Salt transport,
while the control plane alone reaches Salt API. No Docker subnet is advertised.
The optional `headscale.private_api` Salt state exposes Headscale's existing
loopback API on the `tailscale0` interface at port 8081 using a systemd socket
proxy. Set pillar `oduflow:headscale_api_bind` to the coordination node's VPN IP
(default `100.64.0.1`) and apply the reviewed Headscale policy. Only `tag:master`
peers may reach this port, and the Headscale API key remains required. Use
`http://100.64.0.1:8081` through the VPN HTTP proxy; public API routes remain closed.
Odoo uses the gateway's internal SOCKS listener on port 1055 for outbound VPN
access. Master reaches the restricted pillar proxy directly over private TLS.
The proxy remains the direct peer checked by Odoo's trusted-pillar-host setting.

The process applying the stack supplies `ODUFLOW_ENCRYPTION_KEY`,
`ODUFLOW_PILLAR_TOKEN`, `ODUFLOW_SALT_API_PASSWORD` and `ODUFLOW_LITELLM_SETTINGS`
through the named team secrets `platform-encryption-key`, `platform-pillar-token`,
`platform-salt-api-password` and `platform-litellm-settings`. The Salt API password
must match the protected Master htpasswd file.
Only `ODUFLOW_CONTROL_PUBLIC_HOST` is read from the applying process environment.
Secrets are entered by an operator in Dashboard → Credentials → Secrets;
the manifest contains only `secret:<name>` references. The LiteLLM settings
secret is a protected JSON configuration using
the existing [managed gateway contract](litellm-managed.md): config/digest,
software version, master/backend keys, source/incarnation, ingest authentication
and rating contexts. Bootstrap supplies the native release, internal listener and
the generated service-database URL; do not place database credentials in YAML.
The generated database URL remains an ordinary container environment variable,
resolved from the service database at apply time. It is not a named vault secret.

For manual deployment alongside the existing control Odoo, add
`--existing-control <control-environment>` to the generator command. The output is a
`ManualServicePlan`, not an applyable Stack: it contains services, databases,
volumes and protected file requirements, and deliberately omits environment
creation/update. The VPN upstream is the existing `oduflow-<team>-<control-environment>-odoo`.
Use the plan with Oduflow service tools. Do not feed this plan to `stack apply`
or replace the running control environment's credentials with test values.

Metering source/key/rating records and control-plane URLs must be initialized
through the normal Odoo APIs before real usage. This first manifest does not
automatically initialize customer billing records or migrate existing clients.
The container bootstrap owns its configuration apply; the existing Odoo
"Apply with Salt" host action still targets enrolled VM nodes, not Oduflow
service containers. Replace the service configuration through stack apply.

## Headscale in the stack

Pass `--headscale-image <image@sha256:digest>` to include the official Headscale
container and separate configuration/data volumes. Render its protected inputs
from the authoritative Salt defaults:

```sh
python3 scripts/render-headscale-container.py \
  --public-url "https://$HEADSCALE_HOSTNAME" \
  --output-dir docker/protected/headscale
```

The public route permits only coordination endpoints; the administrative API,
metrics and gRPC are not published. Master may reach the authenticated API over
the private service network. The SQLite database and Noise key persist on the
data volume. Keep both volumes during replacement and back them up consistently.

Headscale requires a direct HTTPS endpoint with support for POST protocol Upgrade.
Cloudflare DNS-only records are supported; Cloudflare Proxy/Tunnel cannot carry
this protocol. A host without a public IP requires port forwarding or an independent
public ingress tunnel. See the [Headscale proxy requirements](https://headscale.net/stable/ref/integration/reverse-proxy/).

For a fresh network, first apply a bootstrap manifest containing the control Odoo
and Headscale. Validate Headscale, issue a short-lived tagged enrollment key into
a protected file, then apply the full manifest. After enrollment, omit the key
from durable desired state and verify a repeat plan has no changes.
