# ansible.luks

Ansible role that provisions LUKS2-encrypted LVM logical volumes. For each disk it:

1. Partitions the disk (GPT, LVM flag)
2. Creates an LVM volume group
3. Generates a random key file per logical volume and stores it under `keys_path`
4. Creates the logical volumes
5. Formats each logical volume as a LUKS2 container using the generated key file
6. Opens the container, creates an ext4 filesystem, mounts it, and adds a `crypttab` entry

## Requirements

- Debian / Ubuntu target host
- Packages installed by this role: `lvm2`, `cryptsetup`, `rsync`, `initramfs-tools`
- Ansible >= 2.10
- `community.general` collection (for `lvg`, `lvol`, `crypttab`, `parted`, `filesystem`, `mount`)

## Role Variables

All variables are set through the `spec` list. Each item describes one disk:

```yaml
spec:
  - disk_name: '/dev/sda'          # block device path
    pv_number: '1'                 # partition number to create
    vg_name: 'ROOT'                # LVM volume group name
    vg_path: '/dev/sda1'          # PV path (partition created above)
    keys_path: "/vault/{{ ansible_hostname }}/secrets"  # local key storage path
    lv_specifications:
      - lv_name: "root"            # logical volume name (also used as key filename)
        mount_point: "/mnt/root"   # where to mount after LUKS open
        size: "5G"                 # LVM size (supports 100%FREE)
        vg_group: "ROOT"           # must match vg_name above
    key_storage:
      local_storage: true          # keep key files on target host
      remote_storage: false        # rsync keys to a remote key manager
      remote_host: "192.168.1.1"
      remote_user: "root"
      remote_path: "/vault/{{ ansible_hostname }}/secrets"
      remote_port: "22"
      remote_key: "/vault/{{ ansible_hostname }}/secrets/ansible_rsa"
```

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `hide_logs` | `true` | Suppress task output containing key material |
| `lvm_install` | `true` | Install LVM/LUKS packages |
| `clevis_install` | `false` | Install Clevis TPM2 packages |

## Example Playbook

```yaml
- hosts: storage_nodes
  become: true
  vars:
    hide_logs: true
    spec:
      - disk_name: '/dev/sdb'
        pv_number: '1'
        vg_name: 'DATA'
        vg_path: '/dev/sdb1'
        keys_path: "/vault/{{ ansible_hostname }}/secrets"
        lv_specifications:
          - lv_name: "data"
            mount_point: "/data"
            size: "100%FREE"
            vg_group: "DATA"
        key_storage:
          local_storage: true
          remote_storage: false
  roles:
    - role: ansible.luks
```

## Security Notes

- Key files are generated with `dd if=/dev/urandom` (4 KB) and stored with mode `0400`
- The `keys_path` directory is created with mode `0700` (root only)
- Set `hide_logs: true` in production to prevent key paths from appearing in Ansible output
- Key files are never deleted by this role; back them up before reprovisioning

## License

MIT

## Author

Safa Bayar
