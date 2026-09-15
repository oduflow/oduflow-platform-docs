# Vultr infrastructure images

The repository has three independent Packer entry points. Build each file
explicitly, not the whole `packer/` directory: they intentionally declare their
own provider, variables and manifest.

| Server | Template | Snapshot contract | Manifest |
| --- | --- | --- | --- |
| Client | `client/packer/client.pkr.hcl` | `oduflow-image-v1` | `packer/output/manifest.json` |
| Salt Master | `packer/master.pkr.hcl` | `oduflow-master-image-v1` | `packer/output/master-manifest.json` |
| LiteLLM | `packer/litellm.pkr.hcl` | `oduflow-litellm-image-v1` | `packer/output/litellm-manifest.json` |

All templates pin Packer 1.16.0, Vultr plugin 2.7.0 and Ubuntu 24.04 x64
(OS 2284). Infrastructure builders use `vc2-1c-1gb` with a 25 GB boot disk;
this is a build size, not production capacity guidance. The default region is
`ams`. These are dependency images, with fresh enrollment and configuration
required on every cloned server.

## Contents and Salt reuse

Both new images reuse `salt/minion/install.sh` to install the pinned Salt
3006.27 and Tailscale 1.102.4 prerequisites from verified repositories.

The Master image adds Salt Master/API 3006.27 and applies
`control_master.packages` for API Python dependencies. The normal
`control_master` state includes that same package state. Headscale, Traefik,
backup configuration, certificates, master PKI and API credentials are supplied
by the existing master bootstrap workflow after cloning; they are not baked in.

The LiteLLM image installs Docker/containerd and applies `litellm.native_packages` to
install the hash-locked native gateway, generated Prisma client, metering virtualenv
and runtime helper files. Native VM installation and the OCI image reuse this state.
The ordinary guarded `litellm.install` chooses packages for the requested runtime. It still requires a
valid identity and verified block volume before writing runtime settings.
No Client Oduflow/Paseo application build runs in either infrastructure image.

Docker/containerd stay masked. On LiteLLM images, their systemd drop-ins also
require `/srv/oduflow/data` to be a mount point. There are no OCI image caches,
databases, gateway settings or application data on the boot disk. The selected
LiteLLM container is pulled onto the verified data volume during normal Salt
configuration. Python helper files and dependencies under `/opt` are software,
not gateway data.

## Build and publish

Run from an isolated checkout of a reviewed commit. Set
`ODUFLOW_VULTR_API_KEY` securely in the builder process environment. Packer
does not read this credential from Odoo settings, and does not upload it to the
VM. Never place it in shell arguments or committed variable files.

```sh
mkdir -p packer/output
packer init packer/master.pkr.hcl
packer validate packer/master.pkr.hcl
packer build -var "source_revision=$(git rev-parse HEAD)" packer/master.pkr.hcl

packer init packer/litellm.pkr.hcl
packer validate packer/litellm.pkr.hcl
packer build -var "source_revision=$(git rev-parse HEAD)" packer/litellm.pkr.hcl
```

Packer creates a disposable VM, uploads only `salt/`, installs dependencies and
runs cleanup. It then creates a private snapshot on the account owning the API
key and removes the builder VM and temporary SSH-key record on successful
completion. Each manifest records the snapshot ID and source revision.
Inspect failed builds before another attempt; never blindly repeat ambiguous
cloud resource creation or delete a machine based only on its name.

Cleanup refuses existing Master/Client identity, control settings, gateway
settings, data mounts, master/API keys, Docker data and symlinked protected
paths. It reuses the minion cleanup for VPN/minion identity, cloud-init data,
SSH host keys and machine ID, removes the uploaded sources and Salt caches,
deletes builder authorized keys and locks its temporary root password.
The plugin's shutdown behavior and fresh-clone verification requirements are
the same as for the [Client image](golden-image.md).

## Use and verification boundary

There is no Packer job runner or publication button in `oduflow_vultr`.
These templates follow the existing external Client build workflow.

The current Odoo plan provisioning accepts the Client contract
`oduflow-image-v1` only. Do not paste a Master or LiteLLM snapshot into the
Client Golden Image field or relabel it as a Client image. Infrastructure
snapshots can be used to create a fresh VM in Vultr separately. Register a
Master clone with its new SSH host key through the existing `oduflow_master`
installation flow. A LiteLLM clone still needs VPN/minion enrollment and its
own verified block volume before the managed LiteLLM Salt configuration runs.
Automatic selection of these role-specific snapshots in Odoo is not introduced
by these build templates.

Static validation and offline Salt tests establish template/configuration
contracts only. Publication and end-to-end provisioning require a real build
and a fresh clone with enrollment. No Master or LiteLLM published snapshot ID
is claimed by this change.
