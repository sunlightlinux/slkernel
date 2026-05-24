# slkernel

Kernel packaging and configuration for **Sunlight Linux** — the recipe and
config that produce the distribution's kernel (the low-latency `sunlight`
variant).

> **Status: early bootstrap.** This repository is being seeded. It will hold the
> kernel package build recipe and the Sunlight Linux kernel config(s), tracking
> an upstream Linux release with a small, documented set of changes.

## What it does

slkernel builds the Sunlight Linux kernel package: it pins an upstream Linux
version, applies the Sunlight kernel configuration (a low-latency desktop
profile), optionally applies a small set of patches, and packages the result
for installation.

## Scope

- **Config** — the kernel `.config` / Kconfig fragments for Sunlight Linux.
- **Packaging** — the build recipe that fetches, configures, builds, and
  packages the kernel.

This repository does **not** vendor the Linux source tree; it references an
upstream release.

## Requirements / invariants

- **UEFI boot.** Sunlight Linux is UEFI-only — the config must keep the options
  required to boot under UEFI (`CONFIG_EFI*`, efivars, the filesystems the
  installer creates, initramfs/block support).
- **Low-latency profile.** The Sunlight kernel is a low-latency build; the
  preemption-related options are deliberate, not incidental.
- **Small, documented upstream delta.** Track upstream closely; justify every
  config change and patch.

## Building

Implementation is in progress; the exact flow is TBD and recipe-driven (an
`xbps-src`-style template, consistent with [slpkgs], or a Makefile).

> ⚠️ A broken kernel config can produce an **unbootable** system. Always
> boot-test a built kernel in a VM (QEMU + OVMF for UEFI) before shipping it.

## Where this fits

slkernel is one of the five projects in the Sunlight Linux tree, wired up via
[slmanifests]:

```bash
repo init -u https://github.com/sunlightlinux/slmanifests.git -b main
repo sync -j4
# slkernel lands at kernel/
```

## Documentation

- [CONTRIBUTING.md](CONTRIBUTING.md) — how to contribute (config + recipe)
- [SECURITY.md](SECURITY.md) — reporting security issues
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) — community standards
- [CLAUDE.md](CLAUDE.md) / [EXAMPLES.md](EXAMPLES.md) — guidelines for
  LLM-assisted work on the kernel config and packaging

## License

[Apache License 2.0](LICENSE).

[slpkgs]: https://github.com/sunlightlinux/slpkgs
[slmanifests]: https://github.com/sunlightlinux/slmanifests
