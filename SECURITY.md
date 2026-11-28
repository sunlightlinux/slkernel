# Security Policy

## Scope

slkernel holds the **kernel source** of Sunlight Linux — AOSP's
`android-mainline` branches plus Sunlight's patches — and the CI that keeps it
current; the build recipe and `.config` live in [slpkgs]. The kernel runs with
full hardware privileges, so choices here are security-relevant:

- **Kernel hardening options** — e.g. `CONFIG_STRICT_KERNEL_RWX`,
  `CONFIG_FORTIFY_SOURCE`, `CONFIG_RANDOMIZE_BASE`, `CONFIG_INIT_ON_ALLOC_DEFAULT_ON`.
- **Module signing / lockdown** — `CONFIG_MODULE_SIG*`, `CONFIG_SECURITY_LOCKDOWN_LSM`.
- **Source integrity** — upstream is fetched over HTTPS from
  `android.googlesource.com` by the sync workflows, which merge it with the
  repository's own token. A release is a tag; its source tarball is made with
  `git archive` from that tag and pinned by SHA-256 in the slpkgs template
  (not yet in a public slpkgs branch), and the resulting packages are signed
  with the Sunlight repository key.
- **Applied patches** — out-of-tree patches change kernel behavior; each is
  listed with its reason in [PATCHES.md](PATCHES.md). Patches that weaken a
  protection (for example a boot option that allows unsigned modules) are not
  carried.
- **Build reproducibility** — the same tag gives the same tarball, and the
  config is normalized with `make olddefconfig`.

## Reporting a Vulnerability

If you discover a security issue in the Sunlight Linux kernel packaging or
configuration — for example a disabled hardening option, an unverified source,
or a suspicious patch — please report it responsibly.

**Do NOT open a public GitHub issue for security vulnerabilities.**

Instead, please send an email to: ionut_n2001@yahoo.com

Include:
- Description of the issue
- The affected config option / patch / recipe step
- Potential impact
- Suggested fix (if any)

You should receive a response within 48 hours. We will coordinate a fix before
any public disclosure.

> For vulnerabilities in the **upstream Linux kernel** itself (not Sunlight's
> packaging), report them to the upstream kernel security process.

[slpkgs]: https://github.com/sunlightlinux/slpkgs
