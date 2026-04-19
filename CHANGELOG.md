# Changelog

## [1.0.0] - 2026-04-18

### Added
- Initial release of `bayars.luks` Ansible collection
- LUKS2-encrypted LVM volume provisioning (`encrypt` action)
- Volume unlocking and mounting (`decrypt` action)
- LUKS container close/unmount (`close` action)
- Key rotation with zero-downtime slot swap (`rotate` action)
- Key storage backends: `filesystem`, `remote_filesystem`, `tpm2` (Clevis), `vault` (HashiCorp Vault KV v2)
- Multi-OS support: Debian/Ubuntu, RHEL/Rocky/CentOS, Alpine Linux
- Filesystem support: ext4, xfs, btrfs, swap
- Molecule scenarios: default, rotate, remote_filesystem
- GitHub Actions CI/CD pipeline with syntax check, molecule tests, and Galaxy publish
- Documentation for TPM2, Vault, and Raspberry Pi integration
