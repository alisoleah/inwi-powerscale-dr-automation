# awx_credential_overlay

Bridges local/dev credential handling (`group_vars/all/sites.yml`) and
production AWX credential injection, so `provision_share.yml` and
`provision_replication.yml` don't need two code paths.

## How it works

1. `sites.yml` is loaded as usual via `vars_files` (may contain real dev/test
   values, or `CHANGE_ME` placeholders)
2. This role runs next, in `pre_tasks`, **before** `validate_input`
3. If AWX injected `site_a_onefs_host` / `site_a_api_user` /
   `site_a_api_password` (all three) as extra_vars — via the "PowerScale -
   Site A" Credential Type, see
   `playbooks/dr/credential_types_powerscale_sites.yml` — those values
   overwrite `sites.site_a`'s corresponding fields. Same pattern for Site B.
4. `validate_input` then runs against whatever `sites` ends up being — so a
   misconfigured/missing AWX Credential still surfaces as the normal
   `CHANGE_ME` validation failure, not a silent skip.

## Testing this locally without AWX

Simulate what AWX would inject with `-e`:

```bash
ansible-playbook playbooks/provision_share.yml \
  -e share_dry_run=true \
  -e site_a_onefs_host=10.10.1.10 \
  -e site_a_api_user=admin \
  -e site_a_api_password=realpassword \
  -e site_a_network_pool=pool0 \
  -e site_a_network_subnet=subnet0
```

Even with `sites.yml` still containing `CHANGE_ME` for `site_a`, this
should pass validation — the overlay replaces those fields before
`validate_input` runs. `site_b` (untouched by the `-e` flags above) would
still correctly fail if any share references it, proving the overlay is
per-site and doesn't mask a genuinely missing credential.

## Setting this up for real

1. Run `playbooks/dr/credential_types_powerscale_sites.yml` once against
   your AWX instance to create the two Credential Types (idempotent, safe
   to re-run)
2. In the AWX UI, create one Credential of each type (Resources →
   Credentials → Add), filled in with the real Site A / Site B values
3. Attach both Credentials to the relevant Job Templates
   (`provision_share`, `provision_replication`, and eventually the failover
   workflow once it also needs live site connection info)
4. `sites.yml` in the AWX Project can then keep its `CHANGE_ME` placeholders
   permanently — real values never need to be committed anywhere
