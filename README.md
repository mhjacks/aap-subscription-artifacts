# aap-subscription-artifacts

Ansible playbooks to put an **AAP subscription manifest** (and Hub token) on the local filesystem.

Default output: `~/Documents/aap-subscription-artifacts/aap-manifest.zip`

## Goal

Export a Satellite-style manifest zip that includes an Ansible Automation Platform entitlement.

## Default behavior

Given `allocation_name`:

1. **Exists and has AAP** → export it (skip attach)
2. **Exists without SKU `aap_sku`** → attach that SKU, then export
3. **Does not exist** → create it, attach SKU `aap_sku` (default `MCT4701`), then export

Pool discovery matches Customer Portal pools whose `productId` equals `aap_sku`. Entitlements are never copied from other allocations.

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/fetch_artifacts.yml \
  -e allocation_name=my-aap-manifest \
  -e rh_username=YOUR_RH_USER \
  -e rh_password=YOUR_RH_PASSWORD
```

## Requirements

- Ansible 2.14+
- Network to `sso.redhat.com`, `api.access.redhat.com`, and (for attach/create) `subscription.rhsm.redhat.com`
- Red Hat account with an Ansible Automation Platform subscription
- Offline token at `~/Documents/aap-subscription-artifacts/rh-offline-token` (or `rh_offline_token`)

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/generate_offline_token.yml   # first time
```

## Useful variables

| Variable | Default | Description |
|----------|---------|-------------|
| `allocation_name` | `new-aap-allocation` | Create if missing; attach AAP if needed; export |
| `prefer_aap_entitled_allocation` | `false` | If true and create is off, export any ready AAP alloc |
| `attach_subscriptions` | `true` | Attach AAP when selected alloc lacks it |
| `create_allocation_if_missing` | `true` | Create `allocation_name` when it does not exist |
| `aap_sku` | `MCT4701` | SKU / productId to attach (only this pool) |
| `pool_ids` | `[]` | Explicit pool IDs (skips SKU discovery) |
| `pool_quantity` | `1` | Quantity to attach |
| `artifacts_dir` | `~/Documents/aap-subscription-artifacts` | Output directory |
| `foreman_validate_certs` | `false` | Portal TLS verify |

## Outputs

| File | Purpose |
|------|---------|
| `aap-manifest.zip` | Manifest with AAP entitlement |
| `rh-offline-token` / `automation-hub-token` | Offline token for Hub / RHSM |

Aligned with [aap-starter-kit](https://github.com/validatedpatterns/aap-starter-kit) `values-secret` shapes.

## License

Apache-2.0
