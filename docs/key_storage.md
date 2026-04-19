# Key Storage

Each LUKS volume requires a random key file (512 bytes from `/dev/urandom`) that is generated on first run. The `key_storage.type` field on each disk entry controls how those key files are persisted.

## Types

### `filesystem` (default)

Key files are stored on the target host under `keys_path`.

```
keys_path/
  <lv_name>     # 512-byte key file, mode 0400, root:root
```

The `keys_path` directory is created with mode `0700` (root-only access).

This key is referenced directly in `/etc/crypttab` so the system can unlock the volume at boot without a passphrase prompt.

**Suitable for:** single-host deployments, dev/staging, hosts where the OS partition is itself trusted (e.g., hardware with Secure Boot).

### `remote_filesystem`

Same as `filesystem`, but after generating the key files the entire `keys_path` directory is rsynced to a remote key manager host via SSH. The local copy is retained so `crypttab` can still unlock volumes at boot.

```yaml
key_storage:
  type: remote_filesystem
  remote_host: "10.0.0.5"
  remote_user: "keystore"
  remote_path: "/vault/{{ ansible_hostname }}/secrets"
  remote_port: "22"
  remote_key: "/root/.ssh/keymanager_ed25519"
```

The remote host must have the SSH public key of the Ansible controller in its `authorized_keys`.

**Suitable for:** multi-host deployments where you want a centralised backup of all key material.

## Decrypt / unlock

When running `luks_action: decrypt`, key files must be present locally at `keys_path`. If you use `remote_filesystem` and the local files were deleted, restore them from the remote first:

```bash
rsync -az -e "ssh -i /path/to/key" keystore@10.0.0.5:/vault/hostname/secrets/ /var/lib/luks-keys/hostname/
```

## Security recommendations

| Recommendation | Why |
|---|---|
| Set `hide_logs: true` in production | Prevents `keys_path` values from appearing in Ansible output or log files |
| Use a dedicated non-root SSH key for remote sync | Limits blast radius if the key is compromised |
| Restrict `keys_path` to the root filesystem | Key files must survive reboots to allow auto-unlock via `crypttab` |
| Rotate keys periodically with `cryptsetup luksChangeKey` | Limits exposure window of a compromised key |
