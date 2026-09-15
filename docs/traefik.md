# Control-host TLS proxy

`control_proxy` installs official Traefik **3.7.13 linux/amd64** under systemd.
It serves the configured coordination hostname and proxies Headscale at `127.0.0.1:8080`.
It is included in `roles.control_plane`; it can also be applied separately:

```sh
salt-call --local state.apply control_proxy
```

DNS A/AAAA records must point directly at the control server: use Cloudflare
**DNS only**, without orange-cloud proxy. Allow incoming TCP 80/443. Traefik
issues/renews Let's Encrypt certificates through HTTP-01 using
the configured ACME contact. No Cloudflare token, paid ACM or wildcard is needed.
STUN UDP3478 is a separate requirement if embedded DERP is enabled.

The binary comes from [official release v3.7.13](https://github.com/traefik/traefik/releases/tag/v3.7.13)
with archive SHA256
`52cd039a34258dd61c617a95d69252bc6bcae27c520f338186c31c7fef8f6394`,
checked against the published checksums and GitHub asset digest. The service runs
as `traefik`, has only `CAP_NET_BIND_SERVICE` and can write `/var/lib/traefik`.
Its `acme.json` is `0600`, preserved across state reapplication and contains private
certificate keys. Do not clear it during redeployment; protect it as a secret in
backups.

The public proxy permits Headscale control/DERP but returns 404 for paths starting
with `/api`, `/metrics`, `/swagger` or `/debug`. Administration uses separate local,
SSH or VPN access. Headscale metrics/gRPC remain loopback-only. Traefik dashboard
and API are disabled, and no Docker socket is used on this control-host proxy.

HTTP redirects to HTTPS except `/generate_204` (Headscale captive-portal checks)
and Traefik's own HTTP-01 challenge. Incoming `True-Client-IP` and `X-Real-IP` are
removed; Traefik does not trust client-supplied proxy headers when handling
`X-Forwarded-For`. Headscale trusts only local Traefik (`127.0.0.1/32`). POST
Upgrade with `tailscale-control-protocol` works without forcibly replacing
Upgrade/Connection headers. The minimum TLS version is 1.2.

The real 3.7.13 binary was tested locally for version/checksum, HTTPS routing,
blocked and URL-encoded administration paths, unknown Host handling, redirects,
`generate_204`, forged header removal and POST Upgrade returning 101. Reproduce
with Python and PyYAML:

```sh
python3 salt/states/control_proxy/validate.py --binary /path/to/traefik
```

The validator starts temporary loopback listeners and a test backend with ACME
disabled; it never contacts Let's Encrypt. Salt 3006 compilation was also checked
without applying to a host. Live certificate issuance and Tailscale enrollment
have separate evidence in the deployment journal.

Official references:
[Traefik ACME/HTTP-01](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/),
[Headscale reverse proxy](https://headscale.net/stable/ref/integration/reverse-proxy/).
