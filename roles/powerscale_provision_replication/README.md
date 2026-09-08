# powerscale_provision_replication — SKELETON

**Not yet functional.** This role currently only prints what it *would* do.
The real `dellemc.powerscale.synciqpolicy` task is written out in
`tasks/main.yml` but commented out, blocked on 5 confirmations from INWI —
see the header comment in that file for the full list:

1. Fixed vs. variable replication direction per share
2. Access zone parity between the two clusters
3. Real `network_pool` / `network_subnet` values (currently `CHANGE_ME` in `sites.yml`)
4. Site B reachability from AWX's control plane (direct vs. jump host, same vs. separate credential)
5. Whether SyncIQ target certificate exchange is needed

## Required variables (set by the caller)

| Variable | Description |
|---|---|
| `share` | One entry from `shares.yml` |
| `site` | Resolved `sites.yml` entry for `share.primary_site` |
| `target_site` | Resolved `sites.yml` entry for `share.replication.target_site` |

## To finish this role

Once the 5 questions above are answered:
1. Uncomment the real task block in `tasks/main.yml`
2. Delete the "SKELETON" debug task above it
3. Fill in any additional fields the answers require (e.g. a target access
   zone field, or a separate target credential var)
4. Test with `-e replication_dry_run=true` first, then with
   `ansible-playbook --check` (synciqpolicy supports check_mode/diff), then
   for real against a single low-traffic test share before trusting it for
   production shares
