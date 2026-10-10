# vault-approle-action

A reusable CloudBees Unify custom action: logs in to HashiCorp Vault via
AppRole and reads one field from a KV v2 secret, so any workflow in any
component can retrieve a Vault secret with one step instead of
copy-pasting the AppRole login/read logic everywhere.

There's no confirmed native CloudBees Unify action for Vault (checked
docs.cloudbees.com and the `cloudbees-io` GitHub org — the only
documented CloudBees↔Vault integration is a credentials-provider plugin
for CloudBees CI/Jenkins, also AppRole-based). This wraps the official
`vault` CLI in the same AppRole auth method as a reusable action instead.

## Usage

```yaml
steps:
  - name: Check out
    uses: cloudbees-io/checkout@v1

  - name: Get a secret from Vault
    id: vault
    uses: cb-unify-enterprise-poc/vault-approle-action@main
    with:
      vault-addr: ${{ vars.VAULT_ADDR }}
      role-id: ${{ vars.VAULT_ROLE_ID }}
      secret-id: ${{ secrets.VAULT_SECRET_ID }}
      secret-path: secret/demo
      secret-field: password

  - name: Use the secret later in the workflow
    uses: docker://alpine:3.20
    shell: sh
    env:
      DB_PASSWORD: ${{ steps.vault.outputs.value }}
    run: |
      echo "Got a secret from Vault - length: ${#DB_PASSWORD} characters"
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `vault-addr` | Yes | — | Vault server address, e.g. `http://1.2.3.4:8200` |
| `role-id` | Yes | — | AppRole `role_id` |
| `secret-id` | Yes | — | AppRole `secret_id` — a real credential, always pass as `${{ secrets.* }}`, never commit it |
| `secret-path` | Yes | — | KV v2 secret path, e.g. `secret/demo` |
| `secret-field` | No | `password` | Field name to read from the secret |

## Outputs

| Output | Description |
|---|---|
| `value` | The retrieved secret field value |

**Status: unverified.** The `outputs:` block in `action.yaml` is modeled on
how `cloudbees-io/kaniko@v1` exposes its own output to a calling
workflow, since `mcp__cloudbees__workflow_validate` only validates
`kind: workflow`, not `kind: action` — there was no way to check this
mechanically before a real run. If `${{ steps.vault.outputs.value }}`
comes back empty in a real run, that's the part to fix first.

## Example Vault setup

Vault OSS, dev mode, deployed via the official Helm chart — dev mode
only (in-memory, auto-unsealed, root-token auth), fine for development
and testing, never for production:

```bash
vault kv put secret/demo username=demo-user password=demo-s3cr3t
vault auth enable approle
vault policy write demo-policy - <<EOF
path "secret/data/demo" {
  capabilities = ["read"]
}
EOF
vault write auth/approle/role/demo-role token_policies="demo-policy" token_ttl=1h token_max_ttl=4h
vault read auth/approle/role/demo-role/role-id
vault write -f auth/approle/role/demo-role/secret-id
```
