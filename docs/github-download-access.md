# Client repository access

Settings → Oduflow → Client Download Repositories contains one GitHub
`owner/repository` per line. `data/github_repositories.xml` initializes it to
`oduflow/oduflow-platform`. The public `oduist/oduflow` repository needs no
deploy key and is not included in this list. The data uses `noupdate` so
module upgrades preserve administrator edits. New plans snapshot this list.

New client deployments use an SSH deploy key with write access for their own
repository and separate read-only deploy keys for the configured repositories.
GitHub requires a distinct deploy key for each repository. Shared repository
keys belong to the commercial customer within the control company and are
reused across that customer's instances. The control GitHub token needs
permission to manage deploy keys in every configured repository. It is never
sent to new SSH-based clients. GitHub API operations such as creating issues
or pull requests still need a separately configured GitHub API credential;
Git fetch/clone/push use the managed SSH keys.

Key pairs are encrypted before the provisioning queue starts. External POSTs
use durable dispatch receipts and reconcile matching key material, title,
permissions, repository ID and credential fingerprint before reuse. An unknown
POST that cannot be found is not retried automatically. Resolve its outcome
before resuming provisioning or deletion.

Salt installs the keys on the verified client volume for root (Oduflow) and
Paseo, with private permissions and suppressed content diffs. System Git URL
rewrites select the correct SSH identity for HTTPS and SSH repository URLs
ending in `.git`. SSH verifies GitHub against the pinned public host keys from
https://api.github.com/meta; it does not accept arbitrary host keys.

Deleting an instance deletes its own repository and key. Shared keys remain
until deletion of the customer's last prepared instance, including instances
whose older plans have no download grants. Draft records do not hold access.
Revocation runs in the deletion queue before encrypted credentials are erased;
it verifies key absence on a later poll. Changing the configuration list does
not lose previously issued keys: they are still revoked on final deletion.
Prepared instances and grant history remain for audit.

Existing deployment snapshots and HTTPS credentials are preserved. Existing
clients using manually supplied broad GitHub tokens need an explicit credential
migration; deleting new deploy keys cannot revoke an independently issued token.
This release updates Salt configuration only and does not require image rebuilds.
