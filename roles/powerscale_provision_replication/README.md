# powerscale_provision_replication — SKELETON

**Not yet functional.** This role currently only prints what it *would* do.
The real `dellemc.powerscale.synciqpolicy` task is written out in
`tasks/main.yml` but commented out, blocked on 5 confirmations from INWI —
see the header comment in that file for the full detail:

1. Fixed vs. variable replication direction per share
2. Access zone parity between the two clusters
3. Real `network_pool` / `network_subnet` values (currently `CHANGE_ME` in `sites.yml`)
4. Target reachability from AWX's control plane for the *other* API calls this
   repo makes directly against the target cluster (later phases)
5. Whether `synciqpolicy` needs a certificate exchange
   (`target_certificate_id`/`target_certificate_name`) or a
   `target_cluster.password` field — **genuinely unresolved**: two full-page
   fetches of the current upstream docs show no password field, a second
   review disputed that citing source lines that couldn't be independently
   verified. Settle it directly on a box with the real collection installed:
   `ansible-doc dellemc.powerscale.synciqpolicy | grep -A5 target_cluster`

## Required variables (set by the caller)

| Variable | Description |
|---|---|
| `share` | One entry from `shares.yml` |
| `site` | Resolved `sites.yml` entry for `share.primary_site` |
| `target_site` | Resolved `sites.yml` entry for `share.replication.target_site` |
| `synciq_policy_name` | **Optional.** Overrides the default `<share.name>_replication` naming convention. Already wired into the skeleton's planned output (and the commented-out real task) — this part isn't blocked by the 5 open questions above, unlike the actual `synciqpolicy` call itself. |

## Module param corrections already applied in the commented-out task

Two things were wrong in an earlier draft of this skeleton and have since
been corrected (verified against the current upstream module docs):

- `source_network` is nested **under** `source_cluster`, not a sibling key
- `rpo_alert` / `rpo_alert_unit` are wired to `share.replication.rpo_alert_minutes`
  (top-level params — `default(omit)` is safe to use here, unlike the nested
  `permissions` list in `powerscale_provision_share`'s `smb.yml`, where the
  same pattern is NOT reliable)

Whether `target_cluster.password` exists is unresolved (see open question 5
above) — the commented-out task has a ready-to-uncomment line for it, gated
on checking `ansible-doc` directly rather than guessing either way.

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
