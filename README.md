# aap-subscription-artifacts

Ansible playbooks to obtain local credentials for Ansible Automation Platform (AAP):

1. **Red Hat offline token** (also used as the Automation Hub token)
2. **SCA subscription manifest** (`.zip`) for an allocation

Artifacts are written under `~/Documents/aap-subscription-artifacts/` by default.

## Requirements

- Ansible 2.14+
- Network access to `sso.redhat.com` and `api.access.redhat.com`
- A Red Hat account with an Ansible Automation Platform subscription

When creating allocations or using `manifest_backend: foreman`:

```bash
ansible-galaxy collection install -r requirements.yml
```

You need `rh_username` / `rh_password` for Customer Portal create. Portal TLS uses `foreman_validate_certs: false` by default (no private RHSM CA).

## SCA model (no pools)

Simple Content Access matches the Hybrid Cloud Console: create an allocation, enable SCA, export. There is **no** pool listing or entitlement attach in this role. `/allocations/{uuid}/pools` is not used (it often 404s for SCA/portal allocations).

Flow:

1. Find `allocation_name` or create it (portal preferred; RHSM create often 403s)
2. `PUT` `simpleContentAccess=enabled`
3. Export the manifest zip
4. Write the Hub offline token

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/fetch_artifacts.yml \
  -e allocation_name=aap-local \
  -e rh_username=YOUR_RH_USER \
  -e rh_password=YOUR_RH_PASSWORD
```

An SCA allocation may show **zero attached subscriptions** in the UI; that is expected. Content access comes from the org’s SCA entitlements, not per-allocation pool attaches.

## Offline token

```bash
ansible-playbook playbooks/generate_offline_token.yml
```

## Useful variables

| Variable | Default | Description |
|----------|---------|-------------|
| `rh_offline_token` | (file) | Offline token; falls back to `rh-offline-token` file |
| `rh_username` / `rh_password` | unset | Required to **create** allocations via Customer Portal |
| `artifacts_dir` | `~/Documents/aap-subscription-artifacts` | Output directory |
| `allocation_name` | `aap-local` | Preferred allocation name |
| `allocation_uuid` | unset | Explicit allocation UUID to export |
| `reuse_existing_allocation` | `true` | Prefer existing allocations for download |
| `create_allocation_if_missing` | `true` | Create `allocation_name` when missing |
| `enable_simple_content_access` | `true` | PUT `simpleContentAccess=enabled` |
| `manifest_backend` | `rhsm_api` | `rhsm_api` or `foreman` |
| `foreman_validate_certs` | `false` | Portal TLS verify for theforeman |

## Outputs

| File | Purpose |
|------|---------|
| `rh-offline-token` | Reusable Red Hat offline token |
| `automation-hub-token` | Same token for Hub / `ansible-galaxy` |
| `aap-manifest.zip` | SCA subscription manifest |

Aligned with [aap-starter-kit](https://github.com/validatedpatterns/aap-starter-kit) `values-secret` shapes (`aap-manifest` + `automation-hub-token`).

## License

Apache-2.0
