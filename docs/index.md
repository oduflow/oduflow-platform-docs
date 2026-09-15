---
hide:
  - navigation
  - toc
---

<section class="odu-hero">
  <span class="odu-hero__eyebrow">Managed Odoo Infrastructure</span>
  <h1 class="odu-hero__title">Oduflow Platform Docs</h1>
  <p class="odu-hero__subtitle">
    Provision dedicated <strong>Odoo</strong> environments, onboard customers,
    and operate your infrastructure from one <strong>Odoo 19 control plane</strong>.
  </p>
  <div class="odu-hero__actions">
    <a class="odu-btn odu-btn--primary" href="spec/">Explore the Platform →</a>
    <a class="odu-btn odu-btn--ghost" href="https://github.com/oduflow/oduflow-platform-docs">View on GitHub</a>
  </div>
</section>

<div class="grid cards" markdown>

-   :material-account-group:{ .lg .middle } **Customers and Partners**

    ---

    Configure partner access, onboard customers, and manage their environments.

    [:octicons-arrow-right-24: Customer Configuration](customer-configuration.md)

-   :material-server:{ .lg .middle } **Client Environments**

    ---

    Follow a client from its first deployment to production, backups, and updates.

    [:octicons-arrow-right-24: Client Lifecycle](client-lifecycle.md)

-   :material-cog:{ .lg .middle } **Control Plane**

    ---

    Configure platform services, access roles, credentials, and queued operations.

    [:octicons-arrow-right-24: Control Settings](control-settings.md)

-   :material-cloud-outline:{ .lg .middle } **Infrastructure**

    ---

    Operate Salt, private networking, DNS, and the platform service stack.

    [:octicons-arrow-right-24: Platform Stack](container-services.md)

-   :material-shield-check:{ .lg .middle } **Backup and Recovery**

    ---

    Back up client data, verify restores, and investigate failed deployments.

    [:octicons-arrow-right-24: Backup and Restore](client-backup.md)

-   :material-robot-outline:{ .lg .middle } **Development Tools**

    ---

    Set up the client IDE and optional coding agent for work on Odoo projects.

    [:octicons-arrow-right-24: Coding Agent](client-agent.md)

</div>

## How the platform fits together

The **control plane** manages customers, provider resources, billing, and
operations in Odoo 19. Each **client VM** runs Tailscale, Salt Minion, Paseo,
and Oduflow. Client Oduflow creates the customer's production Odoo stack at
`<slug>.oduflow.sh`.

Salt configures services after enrollment. Optional prebuilt images accelerate
installation; client identity, credentials, and application data arrive after
first boot.

Start with the [platform architecture](spec.md), explore
[client applications](client-apps.md), or follow the
[debugging guide](client-debugging.md) when an operation needs attention.
