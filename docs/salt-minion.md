# Salt minion and client storage foundation

Client bootstrap installs Tailscale and Salt Minion, enrolls the expected
identity and establishes trusted configuration transport. Storage states then
verify the owned XFS volume and install systemd mount guards. Application
installation and production verification are later stages of the
[client deployment workflow](client-debugging.md).

Source paths in this page are relative to the **platform repository root**.
Optional [golden images](golden-image.md) reuse the same installation and cleanup
helpers; they do not contain enrolled client identities.

## Enrollment contract

The selected release runs `client/salt/minion/install.sh` on the clean host to install signed, pinned
Salt 3006, Tailscale, Python 3 and OpenSSL, then join Headscale. The master listens on
its explicit Headscale IPv4 address. Provision the minion key pair in the
trusted orchestrator **before** boot, associate its public key with the durable
instance UUID, and preload that exact public key on the master. See
[salt-master.md](salt-master.md). Never accept a pending key just because its
reported ID or grain matches an instance.

The platform normally invokes `bootstrap.sh` from the selected release checkout.
For low-level enrollment diagnosis, the equivalent bootstrap command runs as root
on the client after VPN connectivity, with that checkout as its working directory:

```sh
cd /opt/oduflow/client/releases/REVIEWED_CLIENT_SHA
python3 salt/minion/bootstrap.py \
  --instance-uuid 12345678-1234-1234-1234-123456789abc \
  --master 100.64.0.10 \
  --master-fingerprint '<SHA256 fingerprint supplied by trusted control plane>' \
  --private-key /run/oduflow/minion.pem \
  --public-key /run/oduflow/minion.pub
```

The script verifies the key pair and Salt major version, installs root-only
keys/config, starts `salt-minion`, and prints only its ID/public fingerprint.
Use a clean, trusted OS with no other minion configuration overrides. Repeated
identical enrollment is accepted; changing an existing key or enrollment is
refused. Source key files in `/run` are caller-owned; the provisioning process must
remove them after successful enrollment and must not embed credentials in a
reusable image. Passing paths keeps key contents out of process arguments.

`minion_id` is exactly `client-<canonical lowercase UUID>`. The pinned master
fingerprint must be the colon-separated SHA256 `master.pub` value from
`salt-key -F master --hash=sha256`, communicated through the trusted orchestrator.
Salt fingerprints hash the PEM body including its line endings, not DER.
There are no autosign tokens or trust-bearing grains. Pillar/job caches are
disabled and log levels limited to warning. These settings do not imply that
secrets cannot exist in process memory or other system logs.

## Storage handoff and states

Use the [client release workflow](client-releases.md) to apply `roles.client_stack`
from the selected client checkout only to the exact UUID minion. The master
publishes only its pinned client dependency, never platform states. Schema 1 must contain `instance_uuid` and:

```yaml
storage:
  device: /dev/disk/by-id/<verified-provider-volume-identifier>
  mount: /srv/oduflow/data
  allow_format: false
  # filesystem_uuid: <expected UUID of an existing owned XFS filesystem>
```

`client.expires_at: null` is allowed: these states do not start the demo clock.
Device identity comes from the provider-to-guest handoff; a by-id path alone does
not prove customer ownership. The orchestrator must verify volume ownership and
attachment before supplying it. `/dev/vdb` and other unstable paths are rejected.

An existing filesystem must match `storage.filesystem_uuid` or the root-only
receipt `/etc/oduflow/storage.json`. A blank volume fails safely by default.
Only after independently confirming that a volume is newly created, disposable,
and belongs to this instance may the operator set boolean `allow_format: true`.
The control plane maps encrypted `storage_allow_format` and
`storage_filesystem_uuid` into these pillar fields. The Vultr adapter authorizes
formatting only after verifying the recorded creation receipt, exact owned volume
and attachment; an explicitly supplied filesystem UUID prevents that default.
The client still checks the actual device and ownership receipt before formatting.
Do not use a pillar override to bypass a failed ownership check.

Preparation waits at most 120 seconds for the by-id device (configurable helper
limit: 600), rejects partitions, child devices, readonly/removable disks, mounted
system disks, swap, other filesystem/partition signatures, nested mounts and
nonempty mount directories. It uses `mkfs.xfs` **without** `-f` only for an explicitly
authorized blank volume. It then records instance/device/filesystem UUID ownership.
An interrupted format with no receipt is recovered by independently inspecting
the filesystem UUID and supplying it explicitly; it is never reformatted blindly.
A missing/replaced volume with an existing receipt cannot be reformatted.

The filesystem mounts persistently with `prjquota`; verification checks the actual
mounted device, XFS type, project quota and writable options. Expected service
units `oduflow.service` and `paseo.service` receive `RequiresMountsFor`, `BindsTo`
and `ExecStartPre` checks. Application installation states must use these exact unit
names and depend on the guard files/systemd reload before `service.running`.
The guards stop the services when systemd deactivates the mount and reject
startup on an incorrect mount. They do not assert daemon health or configure
XFS project IDs/quotas. These storage guards do not install or start the application services themselves.

Preparation is serialized locally by an advisory lock. This prevents competing
runs of this helper; disk hotplug and unrelated root processes remain an external
coordination responsibility. Keep provider attachment stable throughout highstate.

## Online storage growth

An administrator supplies an explicit larger target size. The control plane
reconciles the provider resize before Salt grows the existing owned XFS mount.
The original plan snapshot remains unchanged; verified current capacity is stored
separately.

This path does not shrink, format, detach the disk or stop client services. It
retains the same volume and filesystem ownership checks. An unknown resize result
requires reconciliation before another request; no paid increase is inferred
without an explicit target.

## Image preparation and cleanup

The [client Packer recipe](golden-image.md) reuses these helpers on a disposable
builder. They are not a cleanup procedure for an existing customer VM.

`roles.client_image` installs only the storage/bootstrap prerequisites and the
exact `versions.salt` package pin from an already configured signed repository;
it disables/stops `salt-minion`. This state is one stage of the image build.
The complete recipe also supplies VPN and application packages before final cleanup.

As the last operation on a **disposable, never-enrolled image builder**:

```sh
python3 client/salt/minion/clean-image.py --confirm-disposable-image
```

This refuses instances with enrollment/storage receipts, stops Salt/Tailscale,
removes their identity/cache, SSH host keys, machine identity and cloud-init
instance data. Immediately shut down and snapshot without rebooting. This script
is not a sanitizer for arbitrary previously used machines; never turn a client
VM containing application data or credentials into a golden image.

## Validation

No disk, mount, service or cloud changes occur during these checks:

```sh
cd client
# Install requirements-test.txt before running these checks.
python3 -m unittest discover -s tests -p test_salt_minion.py -v
python3 scripts/validate-salt-minion.py
```

The unit tests use fake command responses and temporary files to exercise storage
refusals, ownership recovery, repeat formatting, mount guards and schema rendering.
The second check uses the release-pinned Salt Jinja/YAML rendering and low-state compilation
and compares the fingerprint algorithm with Salt. A disposable Ubuntu VM with a
new block volume is still needed to validate udev identifiers, actual XFS project
quotas, systemd behavior, network enrollment, reboots and volume attachment races.

Reference contracts: [Salt minion configuration](https://docs.saltproject.io/en/3006/ref/configuration/minion.html#master-finger),
[Salt mount state](https://docs.saltproject.io/en/3006/ref/states/all/salt.states.mount.html),
[systemd mount dependencies](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#RequiresMountsFor=).
