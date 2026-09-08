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

A second review round caught 3 more (all reproduced and re-tested):
`no_log` missing on the credential overlay's `set_fact` tasks, an
empty-string AWX-injection edge case on optional site fields, and a real
accumulator bug in `smb.yml` across multiple shares. See the third commit's
message for details.

One suggested fix (`target_cluster.password`) was **not** applied after
checking the actual module docs — `synciqpolicy`'s `target_cluster` has no
password field at all; see the role's `tasks/main.yml` header for what that
means for open question 5.

## AWX Job Templates

`playbooks/dr/provision_awx_project_and_templates.yml` creates 6 Job
Templates (plus the Project, Inventory, and — for GitLab-backed setups —
the SCM Credential): see the two `INWI-DR-Deployment-Runbook-*.md` files
for the full deployment sequence.

| Template | Playbook | Notes |
|---|---|---|
| PowerScale - Validate Input | `validate_input.yml` | No cluster contact |
| PowerScale - Provision Share (Dry Run) | `provision_share.yml` | `share_dry_run: true`, optional `share_filter` |
| PowerScale - Provision Share | `provision_share.yml` | Optional `share_filter` |
| PowerScale - Provision Share (NFS Only) | `provision_share.yml` | `protocol_filter: nfs`, required `share_filter` |
| PowerScale - Provision Share (SMB Only) | `provision_share.yml` | `protocol_filter: smb`, required `share_filter` |
| PowerScale - Provision Share (Mixed NFS+SMB) | `provision_share.yml` | `protocol_filter: mixed`, required `share_filter` |
| PowerScale - Provision Replication (SyncIQ Config) | `provision_replication.yml` | Still Phase 2 skeleton — plans only, creates nothing real yet. `synciq_policy_name` survey field is genuinely wired in |

The three protocol-specific templates and the SyncIQ Config template all
have a `synciq_policy_name` survey field — **only the SyncIQ Config
template's actually does anything with it today.** On the three share
templates it's captured for the record only, since Phase 1 doesn't touch
SyncIQ — said explicitly in each survey question's description so it's not
mistaken for working end-to-end before Phase 2 is unblocked.

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
│   ├── awx_credential_overlay/          # merges AWX-injected site credentials, when present
│   ├── powerscale_provision_share/      # Phase 1 — NFS/SMB/quota creation
│   └── powerscale_provision_replication/ # Phase 2 — SKELETON, see role header
├── playbooks/
│   ├── validate_input.yml               # run this alone to sanity-check input
│   ├── provision_share.yml              # Phase 1 entrypoint
│   ├── provision_replication.yml        # Phase 2 entrypoint (skeleton)
│   └── dr/                              # placeholder for the existing 5-playbook package,
│       └── credential_types_powerscale_sites.yml  # provisions the 2 AWX Credential Types
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
including `api_password`. **Do not commit real credentials.**

Two ways to supply real values, both fully implemented:

1. **AWX Credential injection (recommended for production)** — two custom
   AWX Credential Types ("PowerScale - Site A" / "PowerScale - Site B"),
   provisioned once via `playbooks/dr/credential_types_powerscale_sites.yml`.
   Attach one Credential of each type to a Job Template and AWX injects the
   real values as `extra_vars`; `roles/awx_credential_overlay` merges them
   onto `sites` before validation runs. `sites.yml` can keep its `CHANGE_ME`
   placeholders permanently in this mode — real secrets never touch the
   repo. See `roles/awx_credential_overlay/README.md` for the full design
   and how to simulate it locally with `-e` flags before you have AWX set up.
2. **ansible-vault** — encrypt `sites.yml` directly, store the vault
   password as an AWX Credential of type "Vault". Simpler, less AWX-specific
   plumbing, but the vault password itself becomes a secret to rotate/manage.
