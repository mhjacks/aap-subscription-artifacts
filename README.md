# aap-subscription-artifacts

Ansible playbooks to obtain local credentials for Ansible Automation Platform (AAP):

1. **Red Hat offline token** (also used as the Automation Hub token)
2. **AAP subscription manifest** (`.zip`) from a Satellite-style subscription allocation

Artifacts are written under `~/Documents/aap-subscription-artifacts/` by default.

## Requirements

- Ansible 2.14+
- Network access to `sso.redhat.com` and `api.access.redhat.com`
- A Red Hat account with an Ansible Automation Platform subscription

When using `manifest_backend: foreman` (download via `theforeman.foreman.redhat_manifest`):

```bash
ansible-galaxy collection install -r requirements.yml
```

You also need `rh_username` / `rh_password` for the Customer Portal.

## Quick start

### 1. Generate an offline token

```bash
ansible-playbook playbooks/generate_offline_token.yml
```

This will:

- Best-effort attempt a password grant if you pass `rh_username` / `rh_password`
- Otherwise open the Hybrid Cloud Console token pages and prompt you to paste a token
- Validate the token against SSO and write:
  - `~/Documents/aap-subscription-artifacts/rh-offline-token`
  - `~/Documents/aap-subscription-artifacts/automation-hub-token`

Token pages:

- Automation Hub: <https://console.redhat.com/ansible/automation-hub/token>
- RHSM API Tokens: <https://access.redhat.com/management/api>

Offline tokens expire after **30 days of inactivity**. Re-run generate or fetch periodically to refresh usage.

### 2. Fetch the AAP manifest and Hub token

```bash
# Using the token file written by generate:
ansible-playbook playbooks/fetch_artifacts.yml

# Or pass the token explicitly:
ansible-playbook playbooks/fetch_artifacts.yml \
  -e rh_offline_token="$(cat ~/Documents/aap-subscription-artifacts/rh-offline-token)"
```

Fetch will:

1. Exchange the offline token for an RHSM API access token
2. Scan existing subscription allocations for one that already has an AAP subscription attached
3. If found, export and download that manifest (prefers `allocation_name` when it is AAP-entitled)
4. Otherwise create (or reuse by name) allocation `aap-local`, attach an AAP pool, then export
5. Write / refresh `automation-hub-token`

Set `reuse_existing_aap_allocation: false` to skip the scan and always use the named allocation create/attach path.

### Download via `theforeman.foreman`

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/fetch_artifacts.yml \
  -e manifest_backend=foreman \
  -e rh_username=YOUR_RH_USER \
  -e rh_password=YOUR_RH_PASSWORD \
  -e @creds.yml
```

With an offline token present, the role still discovers an existing AAP-entitled allocation over the RHSM API, then downloads it with `theforeman.foreman.redhat_manifest` (uuid + portal credentials). If none exists, pass `aap_pool_id` so the module can create/attach `allocation_name` and export the zip.

### One-shot (generate + fetch)

```bash
ansible-playbook playbooks/fetch_all.yml
```

## Credentials file example

```bash
cp inventory/group_vars/all.yml.example creds.yml
# edit secrets in creds.yml (gitignored)
ansible-playbook playbooks/fetch_artifacts.yml -e @creds.yml
```

## Useful variables

| Variable | Default | Description |
|----------|---------|-------------|
| `rh_offline_token` | (file) | Offline token; falls back to `rh-offline-token` file |
| `rh_username` / `rh_password` | unset | Password-grant generate; **required** for `manifest_backend=foreman` |
| `artifacts_dir` | `~/Documents/aap-subscription-artifacts` | Output directory |
| `allocation_name` | `aap-local` | Preferred allocation name (create target / preference among AAP-entitled) |
| `allocation_uuid` | unset | Explicit allocation UUID to export |
| `allocation_version` | latest from API | Satellite version for new allocations (RHSM API backend) |
| `reuse_existing_aap_allocation` | `true` | Reuse any existing allocation that already has AAP attached |
| `aap_pool_id` | unset | Explicit pool id (required for foreman create path) |
| `aap_pool_name_regex` | `(?i)ansible.?automation.?platform` | Pool product name match |
| `aap_pool_quantity` | `1` | Quantity to attach |
| `manifest_backend` | `rhsm_api` | `rhsm_api` or `foreman` (`satellite_module` alias) |
| `open_token_urls` | `true` | Open browser during interactive generate |

## Outputs

| File | Purpose |
|------|---------|
| `rh-offline-token` | Reusable Red Hat offline token |
| `automation-hub-token` | Same token for Hub / `ansible-galaxy` / Validated Patterns |
| `aap-manifest.zip` | Subscription manifest for AAP entitlement |

These paths match the shapes expected by [aap-starter-kit](https://github.com/validatedpatterns/aap-starter-kit) `values-secret` (`aap-manifest` + `automation-hub-token`). This repo does **not** write `values-secret.yaml` for you.

## Security notes

- Token and zip files are created mode `0600`; the artifacts directory is `0700`
- Tasks that touch credentials use `no_log: true`
- Never commit `creds.yml`, token files, or manifest zips (see `.gitignore`)

## License

Apache-2.0
