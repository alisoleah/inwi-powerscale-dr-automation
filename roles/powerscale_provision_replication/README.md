# powerscale_provision_replication — SKELETON

**Not yet functional.** This role currently only prints what it *would* do.
The real `dellemc.powerscale.synciqpolicy` task is written out in
`tasks/main.yml` but commented out, blocked on 5 confirmations from INWI —
see the header comment in that file for the full detail:

1. Fixed vs. variable replication direction per share
2. Access zone parity between the two clusters
3. Real `network_pool` / `network_subnet` values (currently `CHANGE_ME` in `sites.yml`)
4. Target reachability from AWX's control plane for the *other* API calls this
   repo makes directly against the target cluster (later phases) — unrelated
   to authenticating the `synciqpolicy` call itself, see point 5
5. Whether a SyncIQ certificate exchange (`target_certificate_id` /
   `target_certificate_name`) is needed — **note:** `synciqpolicy`'s
   `target_cluster` has no username/password field at all, so this isn't a
   credential question, it's whether a cert-trust step needs to run first

## Required variables (set by the caller)

| Variable | Description |
|---|---|
| `share` | One entry from `shares.yml` |
| `site` | Resolved `sites.yml` entry for `share.primary_site` |
| `target_site` | Resolved `sites.yml` entry for `share.replication.target_site` |

## Module param corrections already applied in the commented-out task

Two things were wrong in an earlier draft of this skeleton and have since
been corrected (verified against the current upstream module docs):

- `source_network` is nested **under** `source_cluster`, not a sibling key
- `rpo_alert` / `rpo_alert_unit` are wired to `share.replication.rpo_alert_minutes`
  (top-level params — `default(omit)` is safe to use here, unlike the nested
  `permissions` list in `powerscale_provision_share`'s `smb.yml`, where the
  same pattern is NOT reliable)

## To finish this role

Once the 5 questions above are answered:
1. Uncomment the real task block in `tasks/main.yml`
2. Delete the "SKELETON" debug task above it
3. If a certificate exchange is needed (question 5), uncomment whichever of
   `target_certificate_id` / `target_certificate_name` applies, sourced from
   wherever that value ends up living (likely a new field on `sites.yml`'s
   target site entry)
4. Test with `-e replication_dry_run=true` first, then with
   `ansible-playbook --check` (synciqpolicy supports check_mode/diff), then
   for real against a single low-traffic test share before trusting it for
   production shares
