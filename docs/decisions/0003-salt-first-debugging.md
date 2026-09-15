# 0003 — Odoo creates client VMs; Salt configures the client stack

Status: Accepted.

The control-plane Odoo runs in Megaflow with the instance model, queue and external
adapters. Provisioning starts there: it creates a client VM through the provider
and publishes the client's DNS records through Cloudflare. Vultr is the initial
provider and is part of the implemented workflow.

`addons/oduflow` owns lifecycle, bootstrap contracts, reconciliation and the journal;
`addons/oduflow_vultr` supplies provider settings and API operations. Existing
`oduflow.*` ORM names and database tables survive migration. Additional cloud
providers can implement the same bootstrap contract.

A client can begin with a supported clean OS. First boot installs Tailscale and
Salt Minion, enrolls with our Headscale and verifies the pinned Salt master and
pre-enrolled minion identity. Salt then manages storage, dependencies, Paseo and
Oduflow configuration. State changes apply to the existing VM without recreating it.

Client Oduflow launches the Odoo production stack on `company.oduflow.sh`.
Its API contract is verified against pinned Oduflow 1.75.0. This is separate from
Odoo in Megaflow. Headscale and Salt master run in dedicated infrastructure services.

A Packer/Vultr golden-image workflow is now authorized to speed up installation.
The build uses a small temporary builder VM; real client identity, secrets and
configuration are still applied on first boot through Salt. Images must be cleaned
of private keys, enrollment state, customer data and ACME certificates. The clean
OS path remains available; an image is not required to debug or configure a client.

Readiness has distinct meanings: cloud API success confirms a resource, Salt ping
confirms trusted connectivity, and production readiness confirms working apps on
the assigned HTTPS hostnames. Each deployment must complete those public checks.
`roles.client_stack` now installs/configures apps and checks local health;
`roles.client_production` guards ingress, creates/hardens the production, publishes
it and verifies HTTPS plus actual Odoo login/logout and panel authentication.
