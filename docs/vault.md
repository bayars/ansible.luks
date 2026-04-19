# HashiCorp Vault Key Storage

When `key_storage.type: vault`, LUKS key files are stored in a HashiCorp Vault KV v2 secret engine. The local key file is also retained so `crypttab` can auto-unlock at boot.

## How it works

**During `encrypt`:**
1. A 512-byte random key is generated at `keys_path/<lv_name>` (mode `0400`)
2. The key content (base64-encoded) is written to `vault_mount/vault_path/<lv_name>`
3. LUKS is formatted with the local key; `crypttab` points to the local copy

**During `decrypt`:**
1. The key is retrieved from Vault and written to a temp file
2. LUKS is opened with the temp key
3. The temp file is deleted

## Requirements

- HashiCorp Vault server accessible from the Ansible controller
- KV v2 secrets engine mounted (default: `secret/`)
- `community.hashi_vault` collection (included in `requirements.yml`)

## Vault setup

```bash
# Enable KV v2 (if not already enabled)
vault secrets enable -path=secret kv-v2

# Create a policy for Ansible
vault policy write ansible-luks - <<EOF
path "secret/data/luks/*" {
  capabilities = ["create", "update", "read"]
}
EOF

# Create a token with that policy
vault token create -policy=ansible-luks -ttl=1h
```

## Configuration

```yaml
spec:
  - disk_name: /dev/sdb
    ...
    keys_path: "/var/lib/luks-keys/{{ ansible_hostname }}"
    key_storage:
      type: vault
      vault_url: "https://vault.example.com"
      vault_auth_method: token          # token | approle | aws | ldap
      vault_token: "{{ vault_token }}"  # from env var or ansible-vault encrypted var
      vault_mount: "secret"             # KV v2 mount point
      vault_path: "luks/{{ ansible_hostname }}"
```

### Supplying the token securely

Option 1 — environment variable (recommended for CI):
```bash
export VAULT_TOKEN=hvs.XXXXXXX
ansible-playbook playbooks/encrypt.yml -i inventory/hosts
```

Option 2 — Ansible Vault encrypted variable:
```bash
ansible-vault encrypt_string 'hvs.XXXXXXX' --name vault_token >> group_vars/all.yml
```

Option 3 — AppRole auth (recommended for production):
```yaml
key_storage:
  type: vault
  vault_url: "https://vault.example.com"
  vault_auth_method: approle
  vault_role_id: "{{ approle_role_id }}"
  vault_secret_id: "{{ approle_secret_id }}"
  vault_mount: "secret"
  vault_path: "luks/{{ ansible_hostname }}"
```

## Secret structure in Vault

Each LV gets its own secret at `vault_mount/data/vault_path/<lv_name>`:

```json
{
  "key": "<base64-encoded 512-byte key>",
  "lv_name": "data",
  "vg_name": "DATA",
  "hostname": "myserver"
}
```

## Key rotation with Vault

Running `luks_action: rotate` generates a new key, re-slots LUKS, and updates the Vault secret atomically. The old secret version is preserved in Vault history (KV v2 versioning).
