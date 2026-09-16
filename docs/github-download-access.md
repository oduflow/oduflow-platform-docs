# Client repository access

Software downloads, customer project access and partner platform-source access
have different owners and credentials:

| Repository | Client access |
| --- | --- |
| `oduflow/oduflow-client` | Public HTTPS checkout at the selected immutable SHA; no deploy key |
| `oduflow/paseo` | Public IDE source/releases; no customer deploy key |
| The client's own project repository | A verified, instance-owned SSH deploy key with write access |
| Additional configured private repositories | Separate scoped read-only SSH deploy keys |
| `oduflow/oduflow-platform` | Not distributed to client VMs; [partner source access](partner-access.md#downloading-platform-source-access) is a separate workflow |

## Configure additional private repositories

**Settings → Oduflow → Client Download Repositories** accepts one GitHub
`owner/repository` per line. New plans snapshot the configured list. The platform
repository is rejected; the public client and IDE repositories do not need grants.
Changing the list does not silently rewrite an existing provisioning snapshot.

GitHub requires a distinct deploy key for each repository. Additional read-only
keys belong to the commercial customer within the control company and can be
shared by that customer's instances. The client's project write key remains bound
to its instance. The control GitHub credential needs permission to manage these
keys; it is never sent to SSH-based clients.

## Registration and installation

Key pairs are encrypted before external registration. Queued POSTs use durable
dispatch receipts. Reconciliation checks key material, title, permissions,
repository identity and credential fingerprint. An unresolved POST is not repeated
just because a later lookup does not find it.

Salt installs verified keys on the client data volume for root (Client Oduflow)
and the `paseo` user (IDE), with private permissions and suppressed diffs. Git URL
rewrites select the proper SSH identity for repository URLs ending in `.git`.
SSH uses the pinned GitHub public host keys. The manifest and installed key
configuration must agree before project access is considered ready.

Git fetch/clone/push use these SSH keys. GitHub API features such as creating an
issue or pull request require a separately authorized API credential; a deploy
key does not grant that capability.

## Legacy clients and revocation

Existing provisioning snapshots remain immutable. During queued configuration,
the platform reconciles and revokes recorded client grants for the platform and
public software repositories. A legacy HTTPS project token is replaced only after
a write key for the exact client repository is verified. Salt removes managed
token files and the corresponding GitHub credential-store entries, preserving
other hosts. Removing a stored token does not revoke that token at GitHub.

Deleting an instance removes its project repository/key through the guarded
lifecycle. Additional customer-scoped keys remain until the customer's last
prepared instance is deleted; draft records do not retain access. The deletion
queue verifies revocation before erasing encrypted credentials, while preserving
grant history and prepared instances for audit.

Customer-supplied public SSH keys for their own project use the portal's separate
key-management controls. Downloading a partner's platform key is also separate;
neither workflow changes what the client release installer needs.
