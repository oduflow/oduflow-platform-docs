# Debugging the complete client lifecycle

The entry point is Odoo 19 in Megaflow. An operator creates a client and starts
provisioning there; no manually supplied client VM is required. Odoo allocates it
through the selected provider and manages Cloudflare DNS. Bootstrap establishes
Tailscale and Salt Minion, then Salt configures Paseo and Oduflow. **Client
Oduflow** launches the production Odoo stack at `company.oduflow.sh`.

Control-plane Odoo and client production Odoo are separate instances.
`headscale.example.com` hosts Headscale and Salt master/API, not client apps.

## Prepare the control plane

`addons/oduflow` contains shared models, queue/journal, DNS contracts and bootstrap;
`addons/oduflow_vultr` implements provider-specific API behavior. Existing
`oduflow.*` ORM names preserve compatibility. The old addon is merged into the
core without a separate wrapper.

Configure the provider plan, region, selected catalogue size, volume and firewall.
The clean-host Vultr path uses a supported Ubuntu 24.04/26.04 amd64 `os_id`.
An optional sanitized golden image can accelerate installation, while Salt still
supplies the actual first-boot configuration. Keep provider credentials in the
control-plane environment, never in a client image or shared pillar.

Before provisioning, establish VPN access to Headscale/Salt API, the trusted
master pin, minion enrollment, DNS and application configuration. Missing required
inputs cause explicit failures; fabricated credentials must not be used to report
readiness.

## Integration run

1. Create the client and run `Prepare Deployment Plan`. Inspect the immutable
   provider/region/VM/disk snapshot, production and panel hostnames, quotas and
   `direct_tls` mode.
2. Start infrastructure from Odoo. The queue allocates/reconciles VM and volume,
   records provider IDs and verifies attachment. It pre-registers the expected
   minion public key before trusted communication starts.
3. Wait for Headscale enrollment and Salt `client-<UUID>`. This UUID comes from
   Odoo and is independent of hostname or provider ID. On the control host:

   ```sh
   salt-run oduflow.ping minion_id=client-<UUID>
   ```

4. Let the Odoo chain apply client storage, dependencies and app configuration.
   It also publishes DNS-only Cloudflare records for the verified VM address.
5. Use the fixed production role, which calls Oduflow 1.75's tested production API
   without a template. It closes public host ingress before creation, sets the
   administrator credential and verifies container identity before reopening.
6. Verify trusted production HTTPS, actual Odoo 19 login/logout and panel
   authentication. Only the corresponding completed lifecycle step establishes
   full readiness and permits client access delivery.

`infra_state=ready` only establishes VM/disk readiness. Salt ping only establishes
transport. Neither independently proves a working production application.

## Iterate on the same VM

Deliver subsequent changes through Salt to the existing minion. In an authorized
operator session, preview and apply the role:

```sh
salt -L client-<UUID> state.apply roles.client_stack test=True
salt-run oduflow.apply minion_id=client-<UUID>
```

Update the master's checkout and reapply without recreating the VM. Suppress
secret-file contents in Salt changes. Check idempotence, reboot/reconnection and
service refusal when the data disk is unavailable. Odoo remains responsible for
cloud resources and DNS.

## Current boundaries

The demo has completed Odoo-driven clean VM creation, VPN/Salt bootstrap, attached
XFS, Paseo/Oduflow installation, DNS and production HTTPS/authentication checks.
`roles.client_stack` includes application configuration and local health checks;
`roles.client_production` owns guarded production publication and public checks.
The [deployment journal](https://github.com/oduflow/oduflow-platform/blob/main/reports/deployments/deployed-control-plane.md) records concrete live evidence.

Those checks do not replace separate validation of full client deletion, backup
restoration or every recovery scenario. Packer images are an optional installation
optimization and must retain Salt's authority over per-client configuration.
