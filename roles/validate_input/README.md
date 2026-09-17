# validate_input

Pure local validation of `sites.yml` / `shares.yml`. Contains **no**
`dellemc.powerscale` module calls — nothing here ever contacts a PowerScale
cluster. Import this role first in any playbook that consumes those two
files, so malformed input fails immediately with a clear message instead of
failing halfway through a run against a real cluster.

## What it checks

- `sites` is a non-empty dict; every **active** site (see `target_site`
  below) has `onefs_host`, `api_user`, `api_password`, `network_pool`,
  `network_subnet`, and none are left as the `CHANGE_ME` placeholder
- `shares` is a non-empty list; every share name is unique
- Per-share: valid `name`/`path`/`protocol`/`primary_site`
- `quota_gb` is a positive integer when set
- `nfs_clients` present when `protocol` includes `nfs`
- `smb_permissions` present and each entry valid when `protocol` includes `smb`
- `replication` block is internally consistent (`target_site` exists and
  differs from `primary_site`, `schedule` set, `rpo_alert_minutes` positive
  if set)

## Restricting to one site: `target_site`

Optional extra_var (AWX Survey field, or `-e target_site=site_a`),
defaulting to `all`. When set to a real key from `sites.yml`, this role
(and, via the `active_sites`/`active_shares` facts it sets, every
playbook/role downstream) only requires *that* site's connection fields
to be populated - the other site can keep `CHANGE_ME` placeholders without
blocking the run. `provision_share.yml` / `provision_replication.yml` use
the same `active_sites` fact to skip shares whose `primary_site` isn't the
active one. Invalid values (anything not `all` or a real site key) fail
fast with the list of valid options.

## Usage

Standalone:
```bash
ansible-playbook playbooks/validate_input.yml
```

Or import at the top of any other playbook:
```yaml
pre_tasks:
  - name: Validate input before touching any cluster
    import_role:
      name: validate_input
```
