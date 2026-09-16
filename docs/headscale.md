# Headscale

This page covers a **standalone control host**. For Headscale inside Oduflow
services, use [Headscale in the stack](container-services.md#headscale-in-the-stack).
For daily operations in Odoo, see [node management](headscale-management.md).

`roles.control_plane` includes `headscale`, `control_proxy` and `control_vpn`.
Apply it with local Salt from `/srv/oduflow` before enrolling client minions.
Headscale uses the official **0.29.3 amd64** DEB; the package SHA256 and matching
configuration are pinned in the state.

Set the coordination URL, for example `https://headscale.example.com`. Its Cloudflare record
must remain **DNS only**: Tailscale uses HTTP POST with the
`tailscale-control-protocol` Upgrade, which Cloudflare Proxy/Tunnel does not
support. This restriction concerns the coordination endpoint; application routing
is a separate decision.

## Installation

Requirements: Debian/Ubuntu amd64, installed Salt and a checkout at
`/srv/oduflow`. Bootstrap masks `headscale.service` before first installation.
The state installs configuration/policy, then the package, validates configuration,
unmasks and enables the service. This ordering prevents the package's postinst
from starting an unconfigured service. Its data directory is explicitly owned by
`headscale`.

```sh
salt-call --local --file-root=/srv/oduflow/salt/states \
  state.apply roles.control_plane
headscale configtest
headscale policy check --file /etc/headscale/policy.json
systemctl status headscale
```

`config.yaml` and policy contain no secrets and are `root:root 0644`. Noise keys,
SQLite and runtime files live under `/var/lib/headscale`, owned by
`headscale:headscale` with directory mode `0750`; the packaged service uses umask
`0077`. The Unix socket is available to root and the headscale group. Do not add
the Salt API account to that group.

Headscale binds `127.0.0.1:8080`; local Traefik terminates TLS on 443. Metrics 9090
and administrative gRPC 50443 stay on loopback. The proxy preserves HTTP/1.1
Upgrade, POST and long connections, and replaces client-address headers with the
actual peer information. `trusted_proxies` allows only loopback. Administration
API keys are provisioned explicitly for the integrations that need them.

## VPN access policy

Addresses are assigned sequentially from `100.64.0.0/10`. MagicDNS and resolv.conf
replacement are disabled; clients use `--accept-dns=false`. Embedded DERP is
disabled and the public Tailscale relay map supplies fallback connectivity.
Outbound access to those relays is required; a private DERP is a separate option.

The policy permits only these new TCP connections:

| Source | Destination | Ports | Purpose |
| --- | --- | --- | --- |
| `tag:client` | `tag:master` | 4505, 4506 | Salt minion connections |
| `tag:odoo` | `tag:master` | 8000 | Salt HTTPS API |
| `tag:master` | `tag:odoo` | 443 | Odoo external pillar |
| `tag:client`, `tag:odoo` | `tag:llm` | 4000 | LiteLLM HTTP over WireGuard |
| `tag:master` | `tag:master` | 8081 | Optional private Headscale API proxy |
| `tag:terminal-gateway` | `tag:client` | 22 | Administrative certificate SSH |

Other connections, including client-to-client and client-to-Salt-API, are denied.
LiteLLM inference and management share port 4000. LiteLLM authorization restricts
client keys to inference routes; only Control Odoo receives the management key.
See [LiteLLM VPN setup](litellm-vpn.md) for host configuration and verification.
Tailscale handles return traffic. Salt uses connections initiated by minions,
so a separate master-to-client connection rule is unnecessary. Host firewalls
and VPN-only Salt binding provide additional restrictions.

`control_vpn` pins Tailscale **1.102.4** from the signed Ubuntu resolute repository
and additionally verifies the apt public key by SHA256. It starts `tailscaled`
but does not enroll the node or consume enrollment secrets. UFW allows TCP
22/80/443, UDP 41641 and TCP 4505/4506/8000 only on `tailscale0`. Existing rules
and default policy are retained. Bootstrap must first enable UFW with default
incoming DROP/REJECT and SSH allowed; the helper refuses an inactive/open-default
firewall. Repeated application does not duplicate rules. Tailscale creates its
own netfilter chains, so test actual Headscale ACL enforcement between live nodes.

Each `tagOwners` entry has an empty owner list: users cannot self-assign infrastructure tags
through `--advertise-tags`. An administrator assigns tags using local CLI or
one-time tagged preauth keys. Persistent servers do not use ephemeral keys and
need no reusable enrollment keys. Tagged nodes do not automatically expire their
node keys, so client deletion must explicitly remove them from Headscale.

Create a separate short-lived enrollment key for each node:

```sh
# Run in a root shell; this secret file is outside the checkout.
umask 077
headscale preauthkeys create --tags tag:master --expiration 10m \
  --output json > /run/oduflow-master-enrollment.json
```

Pass its `key` through a protected file to bootstrap, join with
`tailscale up --login-server=https://headscale.example.com
--accept-dns=false --hostname=control-master` and the file-backed auth key,
then remove temporary files. Never put the key in Salt output, Git or shared
runner arguments. Create separate `tag:odoo` and `tag:client` keys similarly.
Check assigned tags in `headscale nodes list`; obtain actual addresses there or
with `tailscale ip -4`, rather than assuming enrollment order.

Headscale enrollment does not accept a Salt minion key. Salt separately requires
a generated keypair and fingerprint verification:
[master](salt-master.md), [minion](salt-minion.md).

## Validation and maintenance

`headscale configtest` and `headscale policy check` were tested with the real
0.29.3 binary and temporary SQLite data. These validate format; deployed VPN/TLS
and ACL behavior require live-node checks. `/health` alone does not prove VPN
connectivity. Current live evidence is in the deployment journal.

The automated master backup and recovery workflow is described in
[master automation](master-automation.md). It creates a SQLite Backup API copy,
verifies database integrity, and preserves the associated control-host identities.

For a manual consistent backup, stop Headscale and preserve `/etc/headscale` together
with all of `/var/lib/headscale`, including keys and SQLite WAL. Restoring only
the primary SQLite file without a consistent WAL is unsafe. Repeated bootstrap
must not overwrite existing state. Version upgrades require migration review
and a newly verified package checksum.

Sources: [release 0.29.3](https://github.com/juanfont/headscale/releases/tag/v0.29.3),
[release configuration](https://github.com/juanfont/headscale/blob/v0.29.3/config-example.yaml),
[reverse proxy](https://headscale.net/stable/ref/integration/reverse-proxy/),
[tags](https://headscale.net/stable/ref/tags/),
[policy](https://headscale.net/stable/ref/policy/).
