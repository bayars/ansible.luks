# TPM2 / Clevis Key Storage

When `key_storage.type: tpm2`, the LUKS container is bound to the machine's TPM2 chip via Clevis. The volume unlocks automatically at boot as long as the measured boot state (PCR values) matches the values at bind time. No manual passphrase or key file is needed on the boot path.

## How it works

1. A random key file is generated locally at `keys_path/<lv_name>` (backup slot, mode `0400`)
2. The LUKS2 container is formatted with that key file
3. `clevis luks bind` adds a second LUKS key slot backed by the TPM2
4. `/etc/crypttab` is written with `password: none` — systemd-cryptsetup delegates to Clevis at boot
5. Clevis (via `clevis-systemd`) intercepts the unlock request, queries the TPM2, and opens the container automatically

## Requirements

- A physical or emulated TPM2 chip (check: `ls /dev/tpm*`)
- Packages installed automatically when `clevis_install: true`:
  - Debian/Ubuntu: `clevis clevis-tpm2 clevis-systemd clevis-luks clevis-dracut`
  - RHEL/Rocky: same names via `dnf`

## Configuration

```yaml
spec:
  - disk_name: /dev/sdb
    ...
    key_storage:
      type: tpm2
      tpm2_pcr_bank: "sha256"   # hash algorithm (sha1 or sha256)
      tpm2_pcr_ids: "1,7"       # PCR indices to bind against
```

### PCR index reference

| PCR | Measures |
|-----|----------|
| 0 | BIOS/firmware code |
| 1 | BIOS configuration |
| 7 | Secure Boot state |
| 4 | Boot loader |
| 8–9 | Kernel and initrd |

**Recommended:** `"1,7"` for Secure Boot environments. Add PCR 4 if you want to bind to the specific boot loader.

## Backup key file

The local key file at `keys_path/<lv_name>` remains as a backup LUKS slot. Store it securely (e.g., offline USB drive or a key manager). If the TPM2 is replaced or PCR values change (firmware update, Secure Boot key change), you can re-bind using the backup key:

```bash
# Re-bind after hardware or firmware change
clevis luks bind -y \
  -k /path/to/backup/keyfile \
  -d /dev/VG/lv_name \
  tpm2 '{"pcr_bank":"sha256","pcr_ids":"1,7"}'
```

## Decrypt action with TPM2

The `decrypt` action calls `clevis luks unlock` instead of `cryptsetup open`:

```bash
ansible-playbook playbooks/decrypt.yml -i inventory/hosts -e luks_action=decrypt
```

This works even if the TPM2 has already unsealed the key — it is idempotent.

## Raspberry Pi + TPM2

Raspberry Pi 4/5 with a TPM2 daughterboard (e.g., Infineon SLB9670) is supported. See [raspberry_pi.md](raspberry_pi.md) for Pi-specific setup.
