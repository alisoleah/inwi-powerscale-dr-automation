# powerscale_provision_share

Provisions **one** share (directory + optional quota + NFS export + SMB
share) on a PowerScale cluster. Operates on a single `share`/`site` variable
pair per invocation — the calling playbook (`playbooks/provision_share.yml`)
loops over `shares.yml` and includes this role once per entry.

## Required variables (set by the caller, not here)

| Variable | Description |
|---|---|
| `share` | One entry from `shares.yml` |
| `site` | The resolved `sites.yml` entry for `share.primary_site` |

## Variables this role owns

| Variable | Default | Description |
|---|---|---|
| `share_dry_run` | `false` | When `true`, skips every state-changing task and prints a plan instead |
| `protocol_filter` | unset (= `mixed`) | `nfs` \| `smb` \| `mixed`. Restricts what gets created THIS RUN even if the share declares more than one protocol in `shares.yml` — e.g. `protocol_filter=nfs` against a share with `protocol: [nfs, smb]` creates only the NFS export. Set via a global extra_var (AWX Survey or `-e`), not passed explicitly through `include_role` — see `tasks/main.yml` header. |

## Task flow

1. `filesystem.yml` — create/ensure the directory, with an inline quota
   spec if `share.quota_gb` is set
2. `nfs.yml` — create/ensure the NFS export, only if `'nfs' in share.protocol`
3. `smb.yml` — create/ensure the SMB share + permissions, only if
   `'smb' in share.protocol`

## check_mode caveat

The `smb` module does **not** support `check_mode` — see `tasks/smb.yml` for
details. Ansible will safely skip that task under `ansible-playbook --check`
(nothing runs against the cluster), but you won't get a diff for it. Use
`-e share_dry_run=true` for a dry-run that works identically across all
three tasks regardless of per-module check_mode support.

## Testing

Test against a single site by only including shares in `shares.yml` whose
`primary_site` points at that site's cluster. There's no cross-site
dependency in this role — Phase 2 (`powerscale_provision_replication`) is
what needs both sites.
