# Salt Master

This guide defines Salt master configuration, authenticated external pillar and
restricted runners for enrollment/execution. Deploy through the
[platform stack](container-services.md) or [standalone master automation](master-automation.md)
as appropriate. The host installation below describes the underlying components.
Dated [deployment evidence](https://github.com/oduflow/oduflow-platform/blob/main/reports/deployments/README.md)
is separate from this source contract; rendering configuration alone does not
establish working VPN, TLS or client provisioning.

## Generate and install

The renderer and local unit tests require Python 3.10+:

```sh
python3 salt/master/render.py --vpn-ip 100.64.0.1 \
  --odoo-url https://odoo.control.example \
  --output /tmp/oduflow-master.conf
python3 -m unittest discover -s tests -p test_salt_master.py -v
```

With Python from Salt 3006, also run `python3 scripts/validate-salt-master.py`.
It checks actual loaders, eauth ACLs, and file authentication using a temporary
random password without starting services. Install `passlib==1.7.4` in that
Python environment; onedir installations must use Salt's bundled Python.

Replace the example addresses. The renderer accepts a Headscale address in
`100.64.0.0/10` and an HTTPS Odoo origin without a path or credentials. Its JSON
output is valid YAML and contains no secrets. Existing output is not overwritten.

On a prepared control VM with `salt-master` and `salt-api` 3006 onedir and
CherryPy installed in Salt's environment:

1. Install the reviewed checkout at `/srv/oduflow`, readable by `salt`, including
   its pinned `client` dependency. The fileserver publishes only that client tree.
   Create `/srv/oduflow/salt/pillar` even if it is empty. The
   `salt/master/extmods` directory contains trusted code and must not be writable
   by the API user or minions. Do not run `saltutil.sync_all` over this directory:
   synchronization can remove manually installed extensions.
2. Install the generated file as `/etc/salt/master.d/oduflow.conf`. Review the
   complete master configuration: other includes must not enable autosign,
   public binds, broad eauth permissions, caches, or reactors. The paths
   `/etc/salt/oduflow-autosign-disabled` and
   `/etc/salt/oduflow-autosign-grains-disabled` must not exist.
3. Store a separate pillar token in `/etc/salt/oduflow-pillar.token`, owned by
   `root:salt`, mode `0640`, with at least 32 characters and no whitespace.
   Configure the same `ODUFLOW_PILLAR_TOKEN` in Odoo. Client VMs never receive it.
4. Install the API certificate and private key at `/etc/salt/pki/api/server.crt`
   and `server.key`; the key must be `root:salt`, mode `0640`. Odoo must verify
   the certificate normally. The default backend is Salt's built-in `file`
   eauth. Install `passlib==1.7.4` in Salt's environment and create
   `/etc/salt/oduflow-api.htpasswd` (`root:salt`, `0640`) with a dedicated
   `oduflow-api` login and a random password hash. Use
   `passlib.hash.sha512_crypt.using(rounds=200000).hash(password)` and the file
   format `oduflow-api:<hash>`. Keep the password in Odoo's secret storage,
   outside Git, process arguments, and logs. Test authentication as `salt`.
5. Start services only after the Headscale address appears on `tailscale0`.
   A systemd dependency on `tailscaled` must also check address readiness;
   a running daemon does not by itself guarantee that the VPN address exists.

Master ports 4505/4506 and HTTPS API port 8000 bind to the VPN address. A bind
address does not replace access controls: clients may reach only 4505/4506,
while only the Odoo node may reach 8000. Only the master may access Odoo's pillar
endpoint over the VPN. Enforce these rules in Headscale ACLs and host firewalls.
DNS/TLS configuration must preserve VPN routing; the renderer cannot verify it.

The file login is not a system account and needs no `/etc/shadow` access.
`salt-master` and `salt-api` run as `salt`. `--auth-backend pam` remains available
for existing, verified PAM installations, but ordinary `pam_unix` cannot let an
unprivileged Salt API authenticate another system user. Do not grant root or
sudo to work around that limitation. See
[Salt file eauth](https://docs.saltproject.io/en/3006/ref/auth/all/salt.auth.file.html).

PBKDF2-SHA256 is not used: Passlib 1.7.4's default `HtpasswdFile` does not include
that scheme, and Salt does not supply a custom CryptContext. Explicit
SHA512-crypt avoids an obsolete default hash. Actual `salt.auth.file.auth` tests
cover valid authentication, wrong passwords, and unknown users.

## Enrollment without trusting grains

Each client has the permanent ID `client-<canonical UUID>`. Before creating its
VM, the trusted orchestrator generates a unique RSA key pair, retains the public
key and SHA256 fingerprint in the operation intent, and supplies the private key
only to that VM's bootstrap. Golden images must contain no shared identity keys.

Before the minion first starts, install **that exact, previously known** public
key at `/etc/salt/pki/master/minions/<minion_id>` (`salt:salt`, `0600`). Treat an
existing file as an idempotent repeat only after exact key comparison. A mismatch
is an identity conflict, never permission to overwrite the key. Odoo uses the
restricted `oduflow.bootstrap` runner for this step. Do not accept pending
keys merely because a name, IP, grain, or minion event token matches.

Check fingerprints with Salt:

```sh
salt-key -F master --hash=sha256
salt-key -f client-683a74a5-9ef6-4512-995a-9d6338c008bc --hash=sha256
```

The first fingerprint becomes bootstrap's `master_finger`; the second must match
the operation's expected fingerprint. The format is 32 colon-separated hex pairs.
Salt hashes the PEM body including line endings; hashing DER or the entire PEM
file is not equivalent. Both sides require `hash_type: sha256`.

There is no autosign or reactor that configures clients in response to an
untrusted `minion/start` event. Odoo explicitly invokes an allowed runner after
enrollment. The API has no general wheel access. Operator key removal and
rotation use local `salt-key`; any future API method must verify exact ownership
rather than grant broad `key.*` access.

## External pillar and results

`salt/master/extmods/pillar/oduflow.py` calls
`GET /oduflow/pillar/client-<UUID>` with a Bearer header. It validates HTTPS,
canonical identity, `schema == 1`, matching `instance_uuid`, HTTP 200, and a
response size of at most 1 MiB. It disables redirects and environment proxies,
uses Python/Salt's normal certificate trust store, and has a ten-second timeout.
`client.expires_at = null` is valid. Raw responses and exceptions are never logged.
Read, access, or schema failures abort pillar retrieval; client states must also
validate required fields before changing the host.

This replaces the initial specification's `http_json` proposal. Salt 3006.16's
implementation logs fields from erroneous HTTP responses and does not accept an
arbitrary timeout through `ext_pillar`. The small custom module makes error
handling explicit. See the
[official 3006.16 implementation](https://github.com/saltstack/salt/blob/v3006.16/salt/pillar/http_json.py).

`pillar_cache`, `minion_data_cache`, and job result caching are disabled. JID
directories and job metadata can still remain; secrets still exist in memory,
the event bus, and on clients. Do not send private secrets in runner arguments,
retain raw highstate returns, or enable debug logs on production clients. Back up
PKI and the Odoo database, with the encryption key stored separately, and restrict
access to all of them as secret material.

## Odoo API

Only the netapi `runner` client is enabled. The explicit allowlist covers
`oduflow.ping`, `oduflow.bootstrap`, legacy `oduflow.apply`, and the
start/status pairs for application configuration, production publication, volume
resize, and validated custom states. There is no arbitrary execution, target
pattern, wheel, or caller-supplied pillar override.

For example, `/run` accepts this JSON structure. Credentials are supplied by the
client from secret storage; the placeholder is not a usable password:

```json
{
  "client": "runner",
  "fun": "oduflow.apply",
  "minion_id": "client-683a74a5-9ef6-4512-995a-9d6338c008bc",
  "eauth": "file",
  "username": "oduflow-api",
  "password": "<secret>"
}
```

`ping` returns `reachable`. The legacy synchronous `apply` runs only
`state.apply roles.client_stack`, waits up to 300 seconds, and returns
`succeeded`, `failed`, or `unknown` with state counts. Configuration now installs
and checks the client applications; its success alone does not prove production
publication or an LLM coding session.

Odoo's queue uses durable asynchronous start/status calls instead. Fixed roles
are `roles.client_stack`, `roles.client_production`, and
`roles.client_storage_resize`; custom execution receives a validated, frozen
highstate payload through its separate restricted method. Per-minion workers
serialize execution. Public results contain only identity, request/JID, status,
and counts, not state names, changes, or comments that could disclose secrets.
See [asynchronous Salt](salt-async-apply.md) for persistence and recovery.

A timeout or `unknown` result proves neither success nor absence of changes.
Reconcile existing receipts and job identity before retrying. General `/jobs`
access is not provided; ordinary Salt result caching remains disabled. Odoo
stores the sanitized summary, not raw Salt returns.

### Prepare identity from Odoo

`oduflow.bootstrap(minion_id, public_key)` accepts only a canonical
`client-<UUID>` and an RSA3072 public key in PEM SubjectPublicKeyInfo format with a
final newline. Odoo creates and encrypts the private key; the master never
receives it. The runner atomically installs the exact accepted public key,
refuses conflicts, and uses the local Headscale API to issue a one-hour,
one-use `tag:client` preauth key with neither ephemeral nor reusable enabled.

The runner requires `/var/cache/salt/oduflow-bootstrap` (`salt:salt`, `0700`)
and `/etc/salt/oduflow-headscale-api.key` (`root:salt`, `0640`). The Headscale API
key never passes through the Odoo request. Private `0600` cache records contain
the public fingerprint, durable intent, and, after success, the VPN auth key.
This protected credential cache is separate from Salt job return caching.

Intent is persisted and fsynced **before** the HTTP POST. A repeat returns the
cached result only when identity matches and the key has not expired. A lost
response, incomplete save, or expired key requires manual reconciliation;
there is no automatic second POST. A successful cache record does not authorize
restoring an accepted identity that an operator removed. A file lock serializes
calls for the same UUID.

The response contains `minion_id`, `master_fingerprint`, `headscale_url`,
`headscale_auth_key`, and `expires_at`. **It contains a secret**: Odoo must encrypt
it and exclude the entire response from logs. The secret also traverses Salt's
trusted event bus, so result caches and event returners must remain disabled.
Bootstrap success means only that Salt/VPN credentials are prepared, not that a
VM, applications, or production stack exists.

The HTTP contract follows the
[official Headscale 0.29.3 schema](https://github.com/juanfont/headscale/blob/v0.29.3/gen/openapiv2/headscale/v1/headscale.swagger.json):
`POST /api/v1/preauthkey` with `aclTags`, `expiration`, `reusable`, and `ephemeral`,
returning `preAuthKey`. Plain HTTP is used only on `127.0.0.1:8080`; redirects and
environment proxies are disabled, timeout is ten seconds, and response size is
bounded.

Configuration was checked against the official Salt 3006 documentation for
[master settings](https://docs.saltproject.io/en/3006/ref/configuration/master.html),
[rest_cherrypy](https://docs.saltproject.io/en/3006/ref/netapi/all/salt.netapi.rest_cherrypy.html),
and [eauth ACLs](https://docs.saltproject.io/en/3006/topics/eauth/access_control.html).
Local tests cover trust boundaries and errors; actual Salt loader, configuration,
and file-authentication checks supplement them. Live onedir validation must also
check file permissions, TLS, VPN ACLs, and real state execution.
