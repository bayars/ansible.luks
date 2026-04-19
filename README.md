# bayars.luks

Ansible collection that provisions and manages LUKS2-encrypted LVM logical volumes.

**Supports:** Debian · Ubuntu · Rocky Linux · CentOS Stream · RHEL  
**Galaxy:** `ansible-galaxy collection install bayars.luks`

---

## Features

- Partitions disks, creates LVM volume groups and logical volumes
- Formats each LV as a LUKS2 container (Argon2id PBKDF)
- Creates filesystems, mounts volumes, and writes `/etc/crypttab` entries for auto-unlock at boot
- **Multi-OS:** uses `apt` on Debian/Ubuntu, `dnf` on RHEL/Rocky/CentOS
- **Key storage backends:** local filesystem or remote filesystem (rsync over SSH)
- **Decrypt action:** unlock and mount already-provisioned volumes without reprovisioning

---

## Requirements

```bash
ansible-galaxy collection install -r requirements.yml
```

`requirements.yml`:
```yaml
collections:
  - name: community.general
    version: ">=8.0.0"
  - name: community.crypto
    version: ">=2.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
```

---

## Quick start

### Encrypt (provision)

```bash
ansible-playbook playbooks/encrypt.yml -i inventory/hosts
```

### Decrypt (unlock after reboot)

```bash
ansible-playbook playbooks/decrypt.yml -i inventory/hosts
```

---

## Role variables

### `spec` (required)

List of disks to provision. Each entry:

| Field | Required | Description |
|---|---|---|
| `disk_name` | yes | Block device path, e.g. `/dev/sdb` |
| `pv_number` | yes | Partition number to create |
| `vg_name` | yes | LVM volume group name |
| `vg_path` | yes | Physical volume path (the partition created above) |
| `keys_path` | yes | Directory on the target host where key files are stored |
| `lv_specifications` | yes | List of logical volumes (see below) |
| `key_storage` | yes | Key storage configuration (see below) |

Each `lv_specifications` entry:

| Field | Required | Default | Description |
|---|---|---|---|
| `lv_name` | yes | — | LV name; also used as the key filename and LUKS mapper name |
| `mount_point` | yes | — | Where to mount the volume |
| `size` | yes | — | LVM size string, e.g. `5G`, `100%FREE` |
| `vg_group` | yes | — | Must match the parent `vg_name` |
| `fstype` | no | `ext4` | Filesystem type (`ext4`, `xfs`) |

`key_storage` fields:

| Field | Required | Description |
|---|---|---|
| `type` | yes | `filesystem` or `remote_filesystem` |
| `remote_host` | if remote | Hostname / IP of the remote key manager |
| `remote_user` | if remote | SSH user on the remote host |
| `remote_path` | if remote | Destination path on the remote host |
| `remote_port` | if remote | SSH port (default `22`) |
| `remote_key` | if remote | Path to the SSH private key on the target host |

### Other defaults

| Variable | Default | Description |
|---|---|---|
| `luks_action` | `encrypt` | `encrypt` to provision, `decrypt` to unlock |
| `hide_logs` | `true` | Suppress key paths from Ansible output |
| `lvm_install` | `true` | Install LVM and LUKS packages |
| `clevis_install` | `false` | Install Clevis TPM2 packages |

---

## Example playbook

```yaml
- hosts: storage_nodes
  become: true
  vars:
    luks_action: encrypt
    hide_logs: true
    spec:
      - disk_name: '/dev/sdb'
        pv_number: '1'
        vg_name: 'DATA'
        vg_path: '/dev/sdb1'
        keys_path: "/var/lib/luks-keys/{{ ansible_hostname }}"
        lv_specifications:
          - lv_name: "data"
            mount_point: "/data"
            size: "100%FREE"
            vg_group: "DATA"
            fstype: "ext4"
        key_storage:
          type: filesystem
  roles:
    - role: bayars.luks.luks
```

To use remote key backup:

```yaml
        key_storage:
          type: remote_filesystem
          remote_host: "10.0.0.5"
          remote_user: "keystore"
          remote_path: "/vault/{{ ansible_hostname }}/secrets"
          remote_port: "22"
          remote_key: "/root/.ssh/keymanager_ed25519"
```

---

## Key storage

See [docs/key_storage.md](docs/key_storage.md) for security recommendations, key rotation, and how to restore keys from a remote host before running the decrypt playbook.

---

## License

MIT

## Author

Safa Bayar
