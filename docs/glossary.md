# Components and terminology

| Term | Meaning |
| --- | --- |
| Oduflow Platform / control plane | Odoo 19 and the addons in this repository; orchestrates infrastructure and customer lifecycle. |
| Oduflow Stack | The deployment mechanism hosting control Odoo and platform services. Older reports and tool names may call this Megaflow. |
| Client instance | The control-plane record for one dedicated customer VM and its resources. Its permanent identity is a UUID. |
| Client Oduflow | Software on the client VM that manages the customer's production and development stacks. It is separate from control Odoo. |
| Production Odoo | The customer's business application, normally at `<slug>.<domain>`. |
| Client IDE | The development workspace on a client VM. The UI calls it **IDE**; its implementation is based on Paseo. |
| Shared control IDE | An administrative workspace with client repository working copies. It is a separate service and does not expose client VM files. |
| Paseo | The upstream/fork and retained technical identifiers, including `paseo.service`, the `paseo` user, Salt keys and repository name. |
| Slug | The naming identifier used for the client repository and DNS subtree; distinct from its permanent instance UUID. |
| Client release | An immutable Git SHA of `oduflow/oduflow-client`, with compatibility metadata in that release's `release.json`. |
| Provisioning snapshot | Frozen initial plan, names, resource choices and policies. Later settings and configuration revisions do not rewrite it. |
| Configuration revision | A frozen request to apply settings, bound to its own encrypted credentials and execution identity. |
| Dispatch receipt | Durable evidence binding an external request to its exact target and inputs; used to reconcile a lost response. |
| Unknown / uncertain | An outcome that cannot yet be proven. It does not mean that nothing changed or that execution can be repeated. |
| Golden image | An optional VM installation accelerator, without client identity or data; Salt still supplies real configuration. |
| IDE build | A verified runtime archive for a specific IDE commit, Node version, OS and architecture; it is not a VM image. |

## Names and compatibility

New client IDE addresses use `ide.<slug>.<domain>`. Older instances can retain
`paseo.<slug>.<domain>` from their provisioning snapshot. Use the instance's
recorded addresses; [hostname migration](ide-installation.md) is an explicit
configuration operation.

Use **IDE** in user-facing prose. Preserve literal commands, paths, repository
names, environment variables, pillar keys and historical evidence containing
`paseo`. A documentation rename does not rename a service or migrate an address.

## Source and runtime paths

Paths in platform guides are relative to the platform repository unless a section
states otherwise. Client source lives in the `client/` submodule. Commands executed
on a VM use the selected release under `/opt/oduflow/client/releases/<SHA>`;
instructions must say when the working directory changes. Initialize the submodule
before local checks; see [client releases](client-releases.md).

Package pins belong to the selected release's `salt/states/client_apps/artifacts.json`.
An installed version, a tested historical version and a platform dependency pin
are different facts. Consult the selected release and deployment record instead
of treating a version mentioned in prose as a release selector.
