# validate_input

Pure local validation of `sites.yml` / `shares.yml`. Contains **no**
`dellemc.powerscale` module calls — nothing here ever contacts a PowerScale
cluster. Import this role first in any playbook that consumes those two
files, so malformed input fails immediately with a clear message instead of
failing halfway through a run against a real cluster.

## What it checks

- `sites` is a non-empty dict; every site has `onefs_host`, `api_user`,
  `api_password`, `network_pool`, `network_subnet`, and none are left as the
  `CHANGE_ME` placeholder
- `shares` is a non-empty list; every share name is unique
- Per-share: valid `name`/`path`/`protocol`/`primary_site`
- `quota_gb` is a positive integer when set
- `nfs_clients` present when `protocol` includes `nfs`
- `smb_permissions` present and each entry valid when `protocol` includes `smb`
- `replication` block is internally consistent (`target_site` exists and
  differs from `primary_site`, `schedule` set, `rpo_alert_minutes` positive
  if set)

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
