# Contributing to slkernel

Thank you for your interest in contributing to slkernel, the kernel packaging
and configuration for Sunlight Linux!

> **Status: early bootstrap.** This repository is being seeded. It will hold the
> kernel package build recipe and the Sunlight Linux kernel config(s), tracking
> an upstream Linux release with a small, documented set of changes.

## Reporting Issues

- Use the GitHub issue tracker.
- Kernel issues are hardware- and config-sensitive: include the kernel version,
  the config option(s) involved, your firmware mode (Sunlight Linux is
  **UEFI**), the hardware, and the relevant `dmesg` output.

## Submitting Changes

1. Fork the repository
2. Create a feature branch (`git checkout -b kernel/my-change`)
3. Make your change (config and/or recipe)
4. Normalize the config with `make olddefconfig` (don't hand-regenerate the
   whole file)
5. Build the kernel package and **boot-test it in a VM** (QEMU + OVMF for UEFI)
6. Commit with a clear message and open a Pull Request

## Config Conventions

- **One option per documented reason.** Each config change should be traceable
  to a need; explain *why* in the commit/PR.
- **Use `make olddefconfig`**, not `make defconfig`, so you change only the
  symbols you mean to.
- **Don't disable boot-critical options.** UEFI (`CONFIG_EFI*`), the
  filesystems the installer creates, initramfs/block support, and the
  low-latency preemption profile are load-bearing — changing them is a real
  decision.
- **Keep the upstream delta small.** Prefer upstreamable config over carrying
  out-of-tree patches.

## Testing

- Build the kernel package (recipe-driven).
- **Boot-test in a VM:** QEMU + OVMF (UEFI) against a scratch image — a broken
  config can make the system unbootable.
- Confirm the change took effect: check `zcat /proc/config.gz` (or
  `/boot/config-*`) and `dmesg`.

## License

By contributing, you agree that your contributions will be licensed under the
Apache License 2.0.
