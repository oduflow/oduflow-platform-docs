# Vultr SSH key selection

Fetching the Vultr server plan catalogue also fetches every page of
`GET /v2/ssh-keys`. Both catalogues are applied only after both downloads succeed.
Names and provider IDs are stored locally; public key bodies are discarded.
Missing keys are archived and returning IDs reuse the same record.

The plan's **SSH Key** field selects an optional key by name. No free-text key
creation is offered. Synchronization maps historical plan key IDs to confirmed
catalogue entries. Before preparing new clients, an archived or unresolved
historical selection must be corrected. Clearing the selection disables the
operator key. Preparation freezes the provider ID, and VM creation continues
to send `sshkey_id` as an array. Existing prepared snapshots are unchanged.

Managers may refresh the catalogues; observers can only read them. Neither
group can directly create or edit catalogue records through RPC.

API reference: <https://docs.vultr.com/reference/vultr-cli/ssh-keys/list>.
