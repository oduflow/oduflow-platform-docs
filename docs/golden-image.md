# Optional Vultr golden image

For the separate Salt Master and LiteLLM dependency image templates, see
[Vultr infrastructure images](infrastructure-images.md).

`client/packer/client.pkr.hcl` builds a new disposable Ubuntu 24.04 LTS x64 instance
using Vultr OS **2284** and **vc2-4c-8gb**. It never snapshots an enrolled
client. Packer **1.16.0** and the official Vultr plugin **2.7.0** are pinned.
The region defaults to `ams`. The builder plan only sizes the build: the
snapshot keeps the same 25 GB client disk contract.

The builder installs Salt **3006.27**, Tailscale **1.102.4**, the pinned
Node/uv/Oduflow artifacts, Ubuntu's Docker/containerd packages, and builds Paseo
from its pinned commit. A temporary 4 GB swap file supports that build and is
removed before snapshot. Building the Paseo monorepo needs several GB of scratch
disk and memory; the build tree is deleted after installation, so only the
installed release remains in the image. Compiling Paseo on every client instead
of reusing this image is supported but slow. The package versions are recorded in
`/etc/oduflow/image.json`; Ubuntu dependency packages still follow the signed
Ubuntu repositories, so the resulting snapshot, rather than rebuilding later,
is the exact reusable artifact.

Docker and containerd remain masked during the build. Their image-level units
require `/srv/oduflow/data` to be a mount point. No databases, client settings,
client data volume, application services or OCI image caches are initialized.
The builder size is chosen for compiling Paseo. It is neither a client
allocation nor a guarantee that the full production application workload fits on
a server of any particular size.

## Build

Set `ODUFLOW_VULTR_API_KEY` in the build process environment. The Packer
variable is sensitive and the credential is not sent to shell provisioners or
copied into the image. Do not put credentials in command arguments or committed
Packer variable files.

```sh
mkdir -p packer/output
packer init client/packer/client.pkr.hcl
packer validate client/packer/client.pkr.hcl
packer build -var source_revision=REVIEWED_CLIENT_COMMIT client/packer/client.pkr.hcl
```

Use an isolated checkout of a reviewed commit for the build. The builder uploads
only the repository's `salt` tree and removes that temporary copy afterward.
Final cleanup invokes the existing `salt/minion/clean-image.py` refusal checks,
removes Salt and Tailscale identities, cloud-init instance data, SSH host keys,
temporary authorized keys, machine ID and swap; it locks the temporary root
password. The plugin requests shutdown through its existing SSH connection, takes the
snapshot and removes the temporary instance and Packer SSH-key record. Its
shutdown helper does not verify the final provider power state; cold snapshotting
is not claimed. The cleaned image must therefore be verified by a fresh clone.
The snapshot is retained. Failed builds may need inspection of provider cleanup;
never mistake an existing production instance for a disposable builder.

`packer/output/manifest.json` records the resulting snapshot ID and source
revision. Snapshot descriptions begin with
`oduflow-image-v1 ubuntu24.04 disk25 `. This is a compatibility marker on the
operator's Vultr account, not a cryptographic image attestation.

## Select and launch

On an Oduflow plan, select **Installation Source → Golden Image**, enter the
completed Packer snapshot ID and select a server plan with at least 25 GB local
boot storage. Preparing a client freezes the snapshot ID, image contract and
minimum disk size. Later edits to the plan do not alter prepared clients.

Before requesting paid infrastructure, the control plane verifies that the
snapshot exists on the configured account, is complete and has the expected
marker. It retains the actual snapshot metadata and credential fingerprint in
the encrypted instance bundle. A changed snapshot record is rejected.

Vultr creation sends `snapshot_id` instead of `os_id`, with the same fresh
cloud-init enrollment, firewall group and optional operator SSH key used for a
clean installation. Instance reconciliation requires the exact returned
`snapshot_id`, label, region, plan and firewall. A generic `os_id=164` alone is
insufficient proof of which image was used. Existing historical snapshot fields
remain audit-only unless a newly prepared plan explicitly selects Golden Image.

On first boot, cloud-init creates the unique VPN/minion identity. Salt mounts
the newly verified client data volume and writes actual client configuration.
Only then does it unmask Docker/containerd and their socket. Application package
receipts let the existing installer reuse the already installed versions.
Production and LLM keys are supplied separately through encrypted pillar.

## Validation and official references

Packer init, formatting, full configuration validation and shell syntax checks
pass. Tests cover the small clean builder, service masking, first-boot ordering,
snapshot metadata, creation payload and exact-image reconciliation. Runtime
snapshot creation and clone/enrollment checks are separate deployment evidence.

- [Official Vultr Packer builder 2.7.0](https://github.com/vultr/packer-plugin-vultr/blob/v2.7.0/docs/builders/vultr.mdx)
- [Builder shutdown implementation](https://github.com/vultr/packer-plugin-vultr/blob/v2.7.0/builder/vultr/step_shutdown.go)
- [Vultr instance source and returned snapshot ID](https://docs.vultr.com/reference/terraform/resources/instance)
- [Official Go instance schema](https://github.com/vultr/govultr/blob/master/instance.go)
- [Snapshot metadata and size in bytes](https://docs.vultr.com/reference/terraform/resources/snapshot)
- [Vultr OS catalogue](https://api.vultr.com/v2/os?per_page=500)
