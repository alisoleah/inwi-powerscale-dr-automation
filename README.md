# INWI PowerScale Two-Site DR Automation

Input-driven provisioning of NFS/SMB shares and SyncIQ replication across two
PowerScale sites, feeding into a DR failover workflow.

See `INWI-DR-Automation-Plan.md` (delivered separately) for the full phased plan.
This repo currently implements **Phase 0 (input schema + validation)** and
**Phase 1 (share provisioning)**, with **Phase 2 (replication policy)** stubbed
as a skeleton pending confirmation of two-site details from INWI.

## Status

| Phase | What | Status |
|---|---|---|
| 0 | Input schema (`sites.yml` / `shares.yml`) + validation | Done |
| 1 | `powerscale_provision_share` role (NFS/SMB/quota creation) | Done, `--check`-capable |
| 2 | `powerscale_provision_replication` role (SyncIQ policy) | Skeleton only — values pending INWI |
| 3 | Initial sync trigger + monitor | Not started |
| 4 | Post-sync validation | Not started |
| 5 | Wire into existing failover playbooks | **Blocked — see below** |
| 6 | AWX packaging (Job Templates / Survey) | Not started |

## Corrections applied after code review

A review caught 4 real issues, since fixed (see git log for the commit):
- `smb` module only accepts `read`/`write`/`full` for permissions, not
  `read_write`/`full_control` — schema, validation, and example fixed
- `source_network` is nested under `source_cluster` in `synciqpolicy`, not a
  top-level sibling — fixed in the Phase 2 skeleton's commented-out task
- `default(omit)` inside the nested `smb_permissions` list wasn't reliable —
  `smb.yml` now builds each permission entry's exact dict shape explicitly
  instead
- Wired in `rpo_alert`/`rpo_alert_unit` (unused previously despite being in
  scope) and left `target_certificate_id`/`target_certificate_name`
  placeholders for open question 5

One suggested fix (`target_cluster.password`) was **not** applied after
checking the actual module docs — `synciqpolicy`'s `target_cluster` has no
password field at all; see the role's `tasks/main.yml` header for what that
means for open question 5.

## ⚠️ Missing: existing 5-playbook DR package

The 5 playbooks delivered in an earlier session (SyncIQ monitor, preflight,
failover execute 3-phase, validate, discover shares), plus
`bootstrap_awx_inwi_dr.yml`, `gather_awx_info.yml`, and `ARCHITECTURE.md`,
are **not present in this repo** — they were built in a prior working session
whose files aren't available in this environment. See `playbooks/dr/README.md`
for what to drop in once retrieved.

## Repo layout

```
.
├── group_vars/all/
│   ├── sites.yml          # per-site connection + network details (edit CHANGE_ME values)
│   └── shares.yml         # list of shares to provision + replicate
├── roles/
│   ├── validate_input/                  # pure local validation, no cluster contact
│   ├── powerscale_provision_share/      # Phase 1 — NFS/SMB/quota creation
│   └── powerscale_provision_replication/ # Phase 2 — SKELETON, see role header
├── playbooks/
│   ├── validate_input.yml               # run this alone to sanity-check input
│   ├── provision_share.yml              # Phase 1 entrypoint
│   ├── provision_replication.yml        # Phase 2 entrypoint (skeleton)
│   └── dr/                              # placeholder for the existing 5-playbook package
├── inventory/hosts.yml                  # localhost only — modules talk to PowerScale via REST API
└── requirements.yml                     # dellemc.powerscale >=3.10.0
```

## Running

```bash
# 1. Edit group_vars/all/sites.yml and shares.yml first.

# 2. Validate input only (no cluster contact, safe to run anytime):
ansible-playbook playbooks/validate_input.yml

# 3. Dry-run share provisioning (reports planned actions, touches nothing):
ansible-playbook playbooks/provision_share.yml -e share_dry_run=true

# 4. Real run, Site A only recommended for first test:
ansible-playbook playbooks/provision_share.yml

# 5. Run against ONE share only (e.g. the low-traffic test share you're
#    trying first), instead of every share in shares.yml:
ansible-playbook playbooks/provision_share.yml -e share_filter=finance_data
```

`share_filter` works the same way on `playbooks/provision_replication.yml`.
A `share_filter` value that doesn't match any share name in `shares.yml`
fails fast with the list of known names, rather than silently doing nothing.

## check_mode / `--check` notes

Native Ansible `--check` is safe to use — any module here that doesn't support
check_mode is automatically **skipped** by Ansible under `--check` (no live
changes), but won't produce a meaningful diff for that step. Confirmed support
status per the upstream `dellemc.powerscale` docs (verify against your
installed 3.10.0 collection with `ansible-doc dellemc.powerscale.<module>`,
since this varies by collection version):

- `filesystem` — check_mode supported (as of recent collection versions)
- `nfs` — check_mode/diff supported
- `smb` — check_mode **not** supported (confirmed) — use `share_dry_run=true` for a reliable plan instead
- `synciqpolicy` — check_mode/diff supported (relevant once Phase 2 is implemented)

The `share_dry_run=true` extra-var is the reliable, module-agnostic dry-run
path used by this repo's roles — it skips every state-changing task entirely
and only prints what would happen, regardless of per-module check_mode
support.

## Credentials

`sites.yml` documents the expected shape with `CHANGE_ME` placeholders,
including `api_password`. **Do not commit real credentials.** Use
`ansible-vault` to encrypt the file, or (preferred, matching the AWX pattern
already used elsewhere in this engagement) pass credentials via AWX
Credentials + `extra_vars` / Survey instead of a committed file.
