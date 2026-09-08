# Existing DR playbook package — not yet added to this repo

The following were built and delivered in an earlier working session, but
the actual files aren't present in this environment/repo yet:

- `syncid_monitor.yml` — SyncIQ steady-state monitoring
- `preflight.yml` — pre-failover checks
- `failover_execute.yml` — 3-phase failover execution
- `validate.yml` — post-failover validation
- `discover_shares.yml` — share discovery
- `bootstrap_awx_inwi_dr.yml` — `awx.awx` collection playbook, provisions
  Org/Credential/Inventory/Projects/Job Templates/Workflow in AWX
- `gather_awx_info.yml` — read-only AWX pre-check
- `ARCHITECTURE.md` — Mermaid diagram, render-verified

## To do once retrieved

1. Drop the 5 playbooks into this directory (`playbooks/dr/`)
2. Drop `bootstrap_awx_inwi_dr.yml`, `gather_awx_info.yml`, and
   `ARCHITECTURE.md` into the repo root (or a `docs/`/`awx/` subfolder —
   your call, just update the top-level `README.md` structure section to
   match)
3. Check whether any of those playbooks reference roles that also need to
   be copied into `roles/` — if so add them there
4. Confirm the share/policy naming convention those playbooks expect to
   discover matches what `powerscale_provision_share` /
   `powerscale_provision_replication` actually create (e.g. does
   `discover_shares.yml` expect a specific SMB share naming pattern, a
   specific SyncIQ policy naming pattern like `<share>_replication`, etc.)
   — reconcile if not
5. Known gotcha carried over from the AWX/EE build work earlier in this
   engagement: some AWX init containers hardcode
   `quay.io/ansible/awx-ee:<tag>` regardless of the AWX CR's
   `control_plane_ee_image` field. If `bootstrap_awx_inwi_dr.yml` re-applies
   the AWX CR, double check this alias workaround
   (`k3s ctr images tag ...`) is still in place afterward.
