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
    <a class="odu-btn odu-btn--primary" href="start-here/">Find Your Guide →</a>
  </div>
</section>

<section class="odu-architecture" aria-labelledby="architecture-title">
  <div class="odu-architecture__heading">
    <span class="odu-architecture__eyebrow">The platform at a glance</span>
    <h2 id="architecture-title">One control plane. Every component connected.</h2>
    <p>Odoo coordinates the lifecycle. Salt applies the configuration. Each customer gets an independent environment.</p>
  </div>
  <div class="odu-map">
    <svg class="odu-map__connections" viewBox="0 0 1100 468" preserveAspectRatio="none" aria-hidden="true" focusable="false">
      <defs>
        <marker id="map-arrow-control" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M1 1 L6 4 L1 7" class="odu-map__arrow--control"/></marker>
        <marker id="map-arrow-network" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M1 1 L6 4 L1 7" class="odu-map__arrow--network"/></marker>
        <marker id="map-arrow-ai" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto"><path d="M1 1 L6 4 L1 7" class="odu-map__arrow--ai"/></marker>
      </defs>
      <path class="odu-map__line--control" d="M550 164 V185 Q550 197 538 197 H187 Q175 197 175 209 V228" marker-end="url(#map-arrow-control)"/>
      <path class="odu-map__line--control" d="M550 164 V228" marker-end="url(#map-arrow-control)"/>
      <path class="odu-map__line--control" d="M550 164 V185 Q550 197 562 197 H913 Q925 197 925 209 V228" marker-end="url(#map-arrow-control)"/>
      <path class="odu-map__line--network" d="M175 402 V466" marker-end="url(#map-arrow-network)"/>
      <path class="odu-map__line--control" d="M550 402 V466" marker-end="url(#map-arrow-control)"/>
      <path class="odu-map__line--ai" d="M925 468 V404" marker-end="url(#map-arrow-ai)"/>
    </svg>
    <a class="odu-map__node odu-map__node--control" href="control-settings/">
      <span class="odu-map__category">01 / Orchestration</span>
      <h3>Control Plane <span>Odoo</span></h3>
      <p>Customers, plans, provisioning, billing, and queued operations.</p>
      <span class="odu-map__more">Odoo 19 control plane <span aria-hidden="true">↗</span></span>
    </a>
    <span class="odu-map__edge odu-map__edge--enrollment">Enrollment &amp; policies</span>
    <span class="odu-map__edge odu-map__edge--dispatch">Dispatch jobs</span>
    <span class="odu-map__edge odu-map__edge--budgets">Keys &amp; budgets</span>
    <a class="odu-map__node odu-map__node--headscale" href="headscale/">
      <span class="odu-map__category">02 / Private network</span>
      <h3>Headscale</h3>
      <p>Enrolls Tailscale nodes and defines which services can communicate.</p>
      <span class="odu-map__more">Network coordination <span aria-hidden="true">↗</span></span>
    </a>
    <a class="odu-map__node odu-map__node--salt" href="salt-master/">
      <span class="odu-map__category">03 / Configuration</span>
      <h3>Salt Master</h3>
      <p>Applies software, configuration, updates, and recovery operations.</p>
      <span class="odu-map__more">Desired state → client minions <span aria-hidden="true">↗</span></span>
    </a>
    <a class="odu-map__node odu-map__node--litellm" href="litellm-managed/">
      <span class="odu-map__category">04 / Model access</span>
      <h3>LiteLLM</h3>
      <p>Routes AI requests to model providers, enforces budgets, and meters usage.</p>
      <span class="odu-map__more">Managed AI gateways <span aria-hidden="true">↗</span></span>
    </a>
    <span class="odu-map__edge odu-map__edge--network">VPN coordination <span aria-hidden="true">↓</span></span>
    <span class="odu-map__edge odu-map__edge--configuration">Configuration &amp; updates <span aria-hidden="true">↓</span></span>
    <span class="odu-map__edge odu-map__edge--inference">AI requests <span aria-hidden="true">↑</span></span>
    <div class="odu-map__clients">
      <div class="odu-map__clients-heading">
        <div>
          <span class="odu-map__category">05 / Customer workloads</span>
          <h3><a href="client-apps/">Client Instances <span aria-hidden="true">↗</span></a></h3>
        </div>
        <span class="odu-map__isolation">One dedicated VM per client</span>
      </div>
      <div class="odu-map__instances">
        <div class="odu-map__instance">
          <div class="odu-map__instance-heading"><strong>Client A</strong><span>Independent environment</span></div>
          <div class="odu-map__production">Production Odoo <span>+ PostgreSQL &amp; client data</span></div>
          <div class="odu-map__runtime"><span>Oduflow <small>Application lifecycle</small></span><span>IDE <small>Development workspace</small></span></div>
          <div class="odu-map__foundation"><span>Salt Minion</span><span>Tailscale</span><span>Attached storage</span></div>
        </div>
        <div class="odu-map__instance">
          <div class="odu-map__instance-heading"><strong>Client B</strong><span>Independent environment</span></div>
          <div class="odu-map__production">Production Odoo <span>+ PostgreSQL &amp; client data</span></div>
          <div class="odu-map__runtime"><span>Oduflow <small>Application lifecycle</small></span><span>IDE <small>Development workspace</small></span></div>
          <div class="odu-map__foundation"><span>Salt Minion</span><span>Tailscale</span><span>Attached storage</span></div>
        </div>
      </div>
      <p class="odu-map__clients-caption">Client Oduflow creates and manages the production stack at <code>&lt;slug&gt;.oduflow.sh</code>.</p>
    </div>
  </div>
  <div class="odu-map__legend" aria-label="Connection types">
    <span><i class="odu-map__key odu-map__key--control" aria-hidden="true"></i>Orchestration &amp; configuration</span>
    <span><i class="odu-map__key odu-map__key--network" aria-hidden="true"></i>Private network coordination</span>
    <span><i class="odu-map__key odu-map__key--ai" aria-hidden="true"></i>AI inference</span>
  </div>
  <div class="odu-map__notes">
    <p><strong>A private network connects the services.</strong> Headscale coordinates Tailscale nodes across the control plane, Salt masters, AI gateways, and clients. Tailscale carries their traffic over encrypted WireGuard connections.</p>
    <p><strong>Configuration and usage flow back to Odoo.</strong> Salt returns operation results; LiteLLM exports metered usage for billing. Client minions initiate Salt connections. Arrows show logical operations, and network policies keep client environments isolated.</p>
  </div>
</section>

## Explore the documentation

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
