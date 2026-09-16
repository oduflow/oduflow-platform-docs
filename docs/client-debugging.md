# Client deployment and troubleshooting

Use the selected **control Odoo 19** to prepare and provision a client. The
platform allocates its VM; a manually supplied customer server is not required.
For component ownership and readiness stages, see [architecture](spec.md).

## Before provisioning

1. Configure [control settings and secrets](control-settings.md), the provider
   plan, region, server size, volume and firewall policy.
2. Establish Headscale/Salt connectivity, the trusted master pin, authenticated
   pillar and the [queue runner](queue-and-notifications.md).
3. Select a supported OS or compatible [golden image](golden-image.md), the
   [client release](client-releases.md) and [IDE installation](ide-installation.md).
4. Check GitHub, DNS, certificate contact and enabled backup/LLM integrations.
   Missing required inputs must be resolved before provisioning can succeed.

## Provision and verify

1. Create the client and choose **Prepare Deployment Plan**. Inspect the frozen
   resources, revision, quotas, addresses and ingress mode. Preparation may read
   GitHub to resolve `main`; it does not allocate cloud resources.
2. Start infrastructure. Follow **Operations** while the queue reconciles the VM,
   volume, attachment, DNS and expected `client-<UUID>` identity.
3. Confirm trusted connectivity. If diagnosing enrollment on the master, use:

   ```sh
   salt-run oduflow.ping minion_id=client-<UUID>
   ```

4. Wait for configuration to finish. It verifies storage, applies the selected
   client release and checks application health. Check the recorded verified SHA.
5. Follow the separate [production operation](client-production.md). It guards
   public ingress, creates and hardens Odoo, then verifies HTTPS and authentication.
6. Check activation and any opted-in backup result. Use [Access Link](client-access.md)
   for credential handover. Verify a real coding session separately if required.

## Find the failing boundary

| Symptom | Check next |
| --- | --- |
| Preparation fails | Required plan settings, selected revision and access to its GitHub metadata |
| VM or volume remains pending | Provider operation and durable dispatch receipt; resource ownership/attachment |
| Salt ping fails | VPN enrollment, expected minion ID, accepted key, master pin and private-network ACLs |
| Configuration fails | [Operation details](deployment-recovery-ui.md), [safe diagnostics](salt-job-diagnostics.md), selected release and storage receipt |
| Configuration result is unknown | Reconcile the original request/JID; use the explicit recovery workflow when necessary |
| Applications work locally but production does not | [Publication guard, production receipt and HTTPS checks](client-production.md) |
| Agent cannot call a model | [Agent configuration](client-agent.md), gateway key/model policy and inference connectivity |
| Restore check fails | [Backup policy, repository and owned mount](client-backup.md) |

A ping proves transport; `infra_state=ready` proves VM/disk readiness. Neither
proves a usable production application. Do not infer success from an empty error
message, a completed queue wrapper or a process merely being present.

## Apply a correction to the existing VM

Deliver the reviewed client commit, select its full SHA and use
**Apply Updated Configuration**. For customer settings, use **Save & Apply**.
These operations bind the revision and inputs to the durable request. Editing
the master's checkout alone does not update a release-aware client.

Use **Retry** only after a confirmed failure. For an unknown configuration result,
follow [Recover](deployment-recovery-ui.md#recover-an-unknown-configuration-result).
Preserve the earlier receipt and inspect possible partial changes. Do not replace
the normal queue with an unrecorded direct `state.apply` to make the UI look ready.

After a repair, verify the affected application checks and preservation of client
work. A reboot, missing-volume refusal, restore or provider fault needs its own
rehearsal if that behavior is part of the acceptance criteria. Record the target,
source/deployed revisions and actual results in a dated deployment report.
