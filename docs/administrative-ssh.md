# Administrative SSH

Oduflow Admin opens a root terminal from the control backend. Observer and portal
users cannot open terminals, enable access, issue grants, or revoke sessions.
Use the control environment HTTPS URL and a separate HTTPS hostname for the
terminal. Oduflow Stack publishes both through its native ingress.

## Authentication and lifetime

1. The gateway creates an HttpOnly browser binding and redirects to Odoo.
2. Odoo checks the administrator's current company access and personal password.
   A reason is mandatory. The form accepts HTTPS only and requires CSRF protection.
3. Odoo issues a browser-bound, one-use grant valid for 30 seconds. Only its hash
   is stored; the browser receives the code in a URL fragment, never a query log.
4. The gateway exchanges the code, creates an ephemeral Ed25519 key in memory,
   and requests a certificate through the signer's protected Unix socket.
5. The signer obtains the principal and pinned server identity from Odoo. It signs
   exactly `instance:<UUID>:admin`, with validity `[now, now + 300 seconds)`.
   PTY is allowed; forwarding, agent forwarding, X11, and user RC are disabled.
6. The client trusts `/etc/ssh/oduflow_ca.pub` through `TrustedUserCAKeys`. Its root
   principals file contains only its own instance principal. `authorized_keys`
   and password SSH authentication are disabled. The gateway verifies the server's
   public host key obtained through the UUID-targeted Salt runner.

Certificate expiry prevents a new SSH authentication; it does not kill an existing
SSH connection. The gateway separately enforces a one-hour session maximum,
15-minute input idle limit, and an Odoo authorization check every 10 seconds.
The API timeout is eight seconds; loss of authorization closes the connection.
Odoo expires unrenewed leases after 45 seconds. Disabling SSH, changing the pinned
identity/CA, removing the user's role or company access, deactivating the user,
starting deletion, or clicking Disconnect invalidates access.

The journal contains the administrator, server, reason, times, and state. It does
not record terminal output, passwords, private keys, or raw Salt returns. It is a
session journal, not a recording of individual shell commands.

Gateway and signer API requests use distinct Ed25519 signing identities. Odoo
stores only their public keys. Each signature covers the operation, exact body,
timestamp and random nonce. Requests expire in 30 seconds and nonces are consumed
once. Gateway signatures cannot authorize signing. There are no shared API tokens.
All three private keys (CA and two API identities) are generated inside the service
volumes and never printed or included in images.

## Services and isolation

`docker/ssh-gateway` builds one image with two modes:

| Service | Command | Mounts | Listener |
| --- | --- | --- | --- |
| `ssh-signer` | `signer` | `ssh-signer-data:/data`, `ssh-signer-socket:/run/oduflow-ssh` | Unix socket only |
| `ssh-gateway` | `gateway` | `ssh-gateway-data:/data`, `ssh-signer-socket:/run/oduflow-ssh` | `0.0.0.0:8088` on the service network, plus an enrolled VPN node |

Signer UID 10002 owns the CA; gateway UID 10001 can use the socket through GID
10001. The socket directory is 0750, socket 0660, and private keys 0600. Gateway
never mounts signer data. Containers use bridge networking without privileged
mode, NET_ADMIN, host networking, or a Docker socket. Tailscale uses userspace
networking and `tailscale nc` supplies the SSH stream.

The current deployment uses separate managed containers. It is process/container
isolation, not a separate physical host or VM from the control environment.
The same image can run on a dedicated gateway VM if host-level isolation is needed.

Register the gateway with Oduflow Stack's normal service route on port 8088. Its
HTTP and WebSocket traffic uses the same HTTPS hostname. The listener is on the
container network; do not publish a Docker host port. The signer has no HTTP
listener and uses only its shared Unix socket. If service registration requires
a route, use unused port 65535 for the signer alone.

Public-only config files (the key files are generated at startup):

```json
{"odoo_url":"https://control.oduflow.sh","api_key":"/data/api.key","ca_key":"/data/ca.key","socket":"/run/oduflow-ssh/signer.sock"}
```

```json
{"odoo_url":"https://control.oduflow.sh","public_url":"https://ssh.control.oduflow.sh","api_key":"/data/api.key","signer_socket":"/run/oduflow-ssh/signer.sock","bind":"0.0.0.0","port":8088,"tailscale_userspace":true}
```

During deployment, register `/data/api.pub` from each service in the Odoo system
parameters `oduflow.ssh_gateway_api_public_key` and
`oduflow.ssh_signer_api_public_key`, respectively. SSH settings display these
public keys and the CA public key as read-only values for inspection.
Only after network rollout succeeds, register the signer's `/data/ca.pub` as
`oduflow.ssh_ca_public_key`. Its presence enables administrative SSH by default on
new instances; existing instances remain explicitly controlled.

## Network setup

Oduflow Stack terminates HTTPS for the control Odoo environment and the terminal
service. Configure HTTP-to-HTTPS redirection at that ingress. Keep
`proxy_mode = True` in `.oduflow/odoo.conf`, and set/freeze `web.base.url` to the
control environment's HTTPS origin. Configure that same origin as `odoo_url` in
both SSH services, and set `oduflow.ssh_gateway_url` to the gateway's HTTPS origin.

| Source | Destination | Purpose |
| --- | --- | --- |
| Browser HTTPS | Oduflow Stack Odoo ingress | Control UI and password confirmation |
| Browser HTTPS/WSS | Oduflow Stack gateway ingress, container port 8088 | Terminal |
| Gateway/signer HTTPS | Oduflow Stack Odoo ingress | Signed authorization callbacks |
| Gateway Unix socket | Signer | Certificate signing |
| `tag:terminal-gateway` | `tag:client:22` | Certificate SSH |

The gateway's VPN connection is for outbound SSH to clients. Web ingress does not
use the Salt master or Tailscale Serve. The existing private pillar route on
`tag:odoo:443` is independent and remains in place.

1. Start signer and gateway services with the volumes and configuration above.
2. Enroll the gateway with `tag:terminal-gateway`. Give its OS user Tailscale
   operator access and verify its VPN identity. Keep enrollment keys in files.
3. Verify both public HTTPS origins and WebSocket forwarding through the target Oduflow MCP.
   Odoo must recognize the forwarded HTTPS scheme; keep HTTPS checks, Secure
   cookies, CSRF, browser binding, and signed callbacks enabled.
4. Register the services' public API keys and the CA public key in Odoo. Enable
   SSH on the selected client and apply `roles.client_stack` through its
   UUID-targeted queue operation. Verify provider recovery access first because
   this disables public host SSH.
5. Queue Verify SSH Identity, test the terminal and revocation, then queue
   Restrict External SSH for an existing VM. Check private SSH and public TCP 22.

## Removing the former HTTP workaround

The SSH module no longer intercepts requests to a retired control hostname.
Upgrade to `19.0.1.2.0` removes the obsolete system parameter. Hostname retirement
and HTTP redirection belong to Oduflow Stack's ingress configuration.

For an existing installation, verify native HTTPS ingress before removing the
old routes. Remove `administrative_ssh` from the control master's configuration
and apply `control_config`, `headscale`, and `control_proxy` from this revision.
The renderer no longer emits Odoo, Odoo WebSocket, or terminal ingress routes or
the corresponding master-to-service VPN ACLs. Keep the private pillar ACL.

Remove `ODUFLOW_ADMIN_PROXY_ENABLED` from the old VPN service. Explicitly disable
its persisted Tailscale Serve TCP 444 and 445 routes, and disable the terminal's
persisted Serve 8088 route after switching its ingress. Removing startup code
alone does not clear Tailscale's persistent configuration. Keep Odoo's private
pillar Serve 443 route and the gateway's outbound client SSH access.

See [the commit review](https://github.com/oduflow/oduflow-platform/blob/main/reports/deployments/ssh-http-simplification.md) for the retained SSH changes
and the boundary between source cleanup and deployment migration.

## Vultr firewall lifecycle

New Vultr plan snapshots select `client-private-ssh-v1`. Provisioning creates a
separate firewall group named `oduflow-<UUID>-firewall` in the account authenticated
by that deployment's API credentials. The group allows IPv4/IPv6 UDP 41641, plus
TCP 80/443 for direct TLS ingress. It never adds TCP 22. Vultr's default-deny
firewall handles all other incoming connections; the client additionally binds
sshd to its VPN address and drops non-Tailscale TCP 22 with its own nftables table.

Group/rule creation intents and response IDs use durable dispatch receipts. Lost
responses are reconciled; ambiguous POSTs are not replayed. Unexpected broad rules,
unowned groups and changed credentials stop provisioning.

Existing VM migration requires verified administrative SSH. It verifies the VM's
stored identity, credentials and original configuration, creates its private group,
dispatches one attachment PATCH and confirms the result with GET. The original
snapshot remains immutable; the confirmed group is stored separately. A confirmed
attachment can also be recovered during deletion after an interrupted migration.
The original shared group is not edited or deleted. Instance deletion removes the
owned group only after no VM remains attached, then confirms absence with GET.

## Recovery without Odoo

Keep administrator access to the Vultr account independent of Odoo, Headscale,
the gateway and CA. Use the VM's UUID label and provider console to identify it.
If guest login is unavailable, attach SystemRescue ISO through Vultr and use the
console to repair the guest or reset its local password. This does not require
opening external TCP 22. Follow the provider's [root password recovery procedure](https://docs.vultr.com/how-to-reset-the-root-password-of-a-vultr-compute-instance).

The image cleanup scripts lock the root password. Consequently, opening the console
does not itself guarantee a usable root password: recovery may require the ISO and
a reboot. No permanent emergency SSH key is introduced. Keep an external inventory
of client UUIDs/provider IDs and a backup of the CA volume outside the Odoo database.
Do not format the client block volume during recovery.

## Administrator acceptance check

1. Sign in at the control environment HTTPS URL as an Oduflow Admin.
2. Open the configured client server, select **Administrative SSH**, and click
   **Open Terminal**.
3. Enter a reason and your personal Odoo password on the HTTPS Odoo page. Never
   share that password or enter it on the terminal gateway hostname.
4. Run `id -u` and `hostname`; confirm UID 0 and the selected VM.
5. In Odoo, disconnect the session and confirm the terminal closes. Start another
   session and disable Administrative SSH; confirm authorization revocation closes it.
6. Re-enable access if continued administrator access is desired. This permits new
   grants; revoked sessions cannot be reused.

Opening the terminal hostname directly does not grant shell access. Start from the
server record in the Odoo backend. Portal users receive no root terminal.
