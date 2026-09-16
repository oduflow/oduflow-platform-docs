# Start here

Oduflow Platform provisions and operates dedicated Odoo environments. Choose a
guide for your role and task; administrative procedures require the corresponding
Odoo permissions.

## Choose your path

| You want to… | Start with | Continue with |
| --- | --- | --- |
| Understand the platform | [Architecture](spec.md) | [Components and terminology](glossary.md) |
| Register or work as an integrator | [Visitor and integrator manual](integrator-guide.md) | [Partner access](partner-access.md) |
| Manage your own server | [Customer configuration](customer-configuration.md) | [Passwords and access](client-access.md) |
| Set up the control plane | [Platform stack](container-services.md) | [Settings and secrets](control-settings.md), [access roles](access-control.md) |
| Provision and verify a client | [Client deployment runbook](client-debugging.md) | [Production bootstrap](client-production.md) |
| Update client software | [Client releases](client-releases.md) | [IDE installation](ide-installation.md) |
| Investigate a failed deployment | [Deployment recovery](deployment-recovery-ui.md) | [Salt diagnostics](salt-job-diagnostics.md) |
| Back up or recover data | [Backup boundaries](client-backup.md) | [Master recovery](master-automation.md#backup-and-recovery) |
| Operate AI gateways and billing | [Managed LiteLLM](litellm-managed.md) | [Billing setup](billing-operations.md) |

## Three separate environments

The **control plane** is Odoo 19 with the platform addons. It owns customer
records, deployment plans, cloud resources and operation history.

Each **client VM** runs Salt Minion, Tailscale, Client Oduflow and the client IDE.
Client Oduflow creates and manages that customer's application containers.

**Client production Odoo** is the customer's business application. Its users,
passwords, database and lifecycle are separate from the control-plane Odoo.
The [shared control IDE](control-ide.md) is also separate from each client's IDE.

## Readiness and recovery

| Evidence | What it establishes |
| --- | --- |
| Prepared plan | An immutable deployment specification; no VM readiness |
| Infrastructure ready | Verified VM and attached volume |
| Salt ping | Reachability of the expected enrolled minion |
| Configuration succeeded | The selected configuration completed and application health checks passed |
| Production verified | Public HTTPS, Odoo login/logout and panel authentication passed; opted-in backup checks completed |
| Access delivered | A one-use access grant was consumed |

A running queue job, a reachable server and a healthy production are different
states. Start diagnosis from the instance's **Operations** tab. If an outcome is
unknown, use its reconciliation or recovery action before attempting new work.

## Which document describes the current system?

These guides describe the source contracts. An existing installation may run an
older release or retain an older immutable snapshot. Check its selected and
verified revisions before applying a procedure.

Architecture decisions explain why a design was chosen. Dated deployment reports
record what was checked on a particular installation; they are not a current
installation guide. Module guides in Odubook follow the installed addon version.
The integrator manual is a separately maintained document on the internal Manuals
shelf. See [documentation maintenance](documentation-maintenance.md) for ownership
and publication rules.
