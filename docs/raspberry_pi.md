# Raspberry Pi Integration

This collection runs on Raspberry Pi OS (Debian-based) and Ubuntu for Raspberry Pi with no extra configuration. This page covers Pi-specific considerations.

## Tested platforms

| Board | OS | Status |
|---|---|---|
| Raspberry Pi 4 | Raspberry Pi OS Bookworm (64-bit) | ✓ |
| Raspberry Pi 4 | Ubuntu 22.04 LTS (arm64) | ✓ |
| Raspberry Pi 5 | Raspberry Pi OS Bookworm (64-bit) | ✓ |
| Raspberry Pi CM4 | Raspberry Pi OS Bookworm (64-bit) | ✓ |

## Requirements

### Enable dm-crypt in the kernel

Raspberry Pi OS includes dm-crypt but the module may not auto-load. Add it to `/etc/modules`:

```bash
echo dm-crypt | sudo tee -a /etc/modules
```

Or load it immediately:
```bash
sudo modprobe dm_crypt
```

### Initramfs with cryptsetup support

For auto-unlock at boot via `crypttab`, `cryptsetup` must be in the initramfs:

```bash
# Raspberry Pi OS / Debian
echo "CRYPTSETUP=y" | sudo tee -a /etc/cryptsetup-initramfs/conf-hook
sudo update-initramfs -u -k all
```

This is handled automatically by the `update_initramfs` handler in the role when `lvm_install: true`.

## Performance tuning

The SD card and Pi's CPU are slower than a server. Reduce the Argon2id PBKDF memory and iteration count to avoid a long unlock delay at boot:

```yaml
lv_specifications:
  - lv_name: "data"
    mount_point: "/data"
    size: "100%FREE"
    vg_group: "DATA"
    fstype: "ext4"
    pbkdf_memory: 131072      # 128 MB (default 1 GB — too slow for Pi)
    pbkdf_iterations: 3       # default 4
```

| Board | Recommended `pbkdf_memory` |
|---|---|
| Pi 4 (4 GB) | `524288` (512 MB) |
| Pi 4 (1–2 GB) | `131072` (128 MB) |
| Pi 5 | `524288` (512 MB) |
| Pi Zero / CM4 lite | `65536` (64 MB) |

## Example playbook

```yaml
- hosts: raspberrypis
  become: true
  vars:
    luks_action: encrypt
    hide_logs: true
    spec:
      - disk_name: /dev/sda          # USB SSD attached to the Pi
        pv_number: '1'
        vg_name: 'PIDATA'
        vg_path: /dev/sda1
        keys_path: /boot/luks-keys   # on the SD card (separate partition)
        lv_specifications:
          - lv_name: data
            mount_point: /data
            size: 100%FREE
            vg_group: PIDATA
            fstype: ext4
            pbkdf_memory: 524288
            pbkdf_iterations: 3
        key_storage:
          type: filesystem            # or remote_filesystem for key backup
  roles:
    - role: bayars.luks
```

## TPM2 on Raspberry Pi

The Raspberry Pi 5 has a built-in RP1 chip but no native TPM2. Add a SPI-connected TPM2 module (e.g., Infineon SLB9670 breakout) to use `key_storage.type: tpm2`.

Enable the TPM2 overlay in `/boot/config.txt`:
```
dtoverlay=tpm-slb9670
```

Verify:
```bash
ls /dev/tpm*   # should show /dev/tpm0 and /dev/tpmrm0
```

Then set `clevis_install: true` and `key_storage.type: tpm2` as documented in [tpm2.md](tpm2.md).

## Key storage on the boot SD card

Storing `keys_path` on the SD card at `/boot/luks-keys/` means the encrypted data volume (USB SSD or NVMe) can only be unlocked by the specific SD card. This provides hardware-binding without a TPM2 chip — losing either the SD card or the drive renders the data inaccessible.

```yaml
keys_path: /boot/luks-keys/{{ ansible_hostname }}
```
