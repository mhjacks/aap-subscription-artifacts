# aap-subscription-artifacts

Ansible playbooks to obtain local credentials for Ansible Automation Platform (AAP):

1. **Red Hat offline token** (also used as the Automation Hub token)
2. **AAP subscription manifest** (`.zip`) from an SCA Satellite-style allocation

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

`theforeman.foreman.redhat_manifest` talks to `subscription.rhsm.redhat.com` and, when `validate_certs=true`, hardcodes `/etc/rhsm/ca/redhat-uep.pem`. That CA is mirrored publicly at [Katello's redhat-uep.pem](https://raw.githubusercontent.com/Katello/katello/master/ca/redhat-uep.pem); the role can install it with `--become`.

On current OpenSSL/Python, verification often still fails with `Basic Constraints of CA cert not marked critical`. For that reason `foreman_validate_certs` defaults to `false`. Set it to `true` only if your runtime accepts that CA.

For verified TLS without the UEP CA, use the default `manifest_backend=rhsm_api` (offline token → `api.access.redhat.com`).

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
2. Find an existing SCA allocation (`allocation_name`, else the sole allocation if only one exists)
3. Ensure `simpleContentAccess=enabled` and export the manifest zip
4. Write / refresh `automation-hub-token`

By default the role does **not** create allocations (`create_allocation_if_missing: false`). If you already have an allocation, it just downloads its manifest. SCA is always used — there is no pool-attach path.

### Create an SCA allocation (console-compatible)

Same flow as the Hybrid Cloud Console (no pool picking):

1. `POST /allocations?Name=...&version=...`
2. `PUT /allocations/{uuid}` with `{"simpleContentAccess": "enabled"}`
3. `GET /allocations/{uuid}/export`

```bash
ansible-playbook playbooks/fetch_artifacts.yml \
  -e create_allocation_if_missing=true \
  -e allocation_name=aap-local
```

### Download via `theforeman.foreman`

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/fetch_artifacts.yml \
  -e manifest_backend=foreman \
  -e rh_username=YOUR_RH_USER \
  -e rh_password=YOUR_RH_PASSWORD \
  -e @creds.yml
```

Uses `content_access_mode: org_environment` (SCA). Set `allocation_name` or `allocation_uuid` if you have more than one allocation.

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
| `allocation_name` | `aap-local` | Preferred allocation name |
| `allocation_uuid` | unset | Explicit allocation UUID to export |
| `allocation_version` | latest from API | Satellite version for new allocations |
| `reuse_existing_allocation` | `true` | Prefer existing allocations for download |
| `create_allocation_if_missing` | `false` | Create `allocation_name` when none can be selected |
| `enable_simple_content_access` | `true` | PUT `simpleContentAccess=enabled` |
| `manifest_backend` | `rhsm_api` | `rhsm_api` or `foreman` (`satellite_module` alias) |
| `open_token_urls` | `true` | Open browser during interactive generate |
| `foreman_validate_certs` | `false` | Portal TLS verify for theforeman backend |

## Outputs

| File | Purpose |
|------|---------|
| `rh-offline-token` | Reusable Red Hat offline token |
| `automation-hub-token` | Same token for Hub / `ansible-galaxy` / Validated Patterns |
| `aap-manifest.zip` | SCA subscription manifest for AAP |

These paths match the shapes expected by [aap-starter-kit](https://github.com/validatedpatterns/aap-starter-kit) `values-secret` (`aap-manifest` + `automation-hub-token`). This repo does **not** write `values-secret.yaml` for you.

## Security notes

- Token and zip files are created mode `0600`; the artifacts directory is `0700`
- Tasks that touch credentials use `no_log: true`
- Never commit `creds.yml`, token files, or manifest zips (see `.gitignore`)

## License

Apache-2.0
