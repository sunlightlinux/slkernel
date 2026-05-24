## Description

Brief description of the config / packaging change.

## Type of Change

- [ ] Config change (enable/disable/modify a symbol)
- [ ] Driver enablement
- [ ] Upstream version bump / rebase
- [ ] Patch add/drop
- [ ] Packaging / recipe change
- [ ] Documentation update

## Testing

- [ ] Config normalized with `make olddefconfig` (not `make defconfig`)
- [ ] `git diff` of the config is exactly the intended symbols
- [ ] Kernel package builds
- [ ] Boot-tested in a VM (QEMU + OVMF, UEFI)
- [ ] Change confirmed via `zcat /proc/config.gz` / `dmesg`

## Safety Checklist

- [ ] No boot-critical option disabled (UEFI `CONFIG_EFI*`, root filesystem,
      block/initramfs)
- [ ] Low-latency preemption profile preserved
- [ ] Upstream delta kept small and justified
- [ ] Source integrity verified (checksum / signature) for version bumps
