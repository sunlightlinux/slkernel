# Security Policy

## Scope

slkernel holds the **kernel packaging and configuration** for Sunlight Linux —
the build recipe and the kernel `.config` that produce the distribution's
kernel. The kernel runs with full hardware privileges, so configuration and
packaging choices here are security-relevant:

- **Kernel hardening options** — e.g. `CONFIG_STRICT_KERNEL_RWX`,
  `CONFIG_FORTIFY_SOURCE`, `CONFIG_RANDOMIZE_BASE`, `CONFIG_INIT_ON_ALLOC_DEFAULT_ON`.
- **Module signing / lockdown** — `CONFIG_MODULE_SIG*`, `CONFIG_SECURITY_LOCKDOWN_LSM`.
- **Source integrity** — the upstream tarball/tag must be verified (checksum /
  signature) before building.
- **Applied patches** — out-of-tree patches change kernel behavior; each should
  be justified and reviewed.
- **Build reproducibility** — an auditable, reproducible config and recipe.

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
