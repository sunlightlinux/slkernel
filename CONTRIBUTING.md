# Contributing to slkernel

Thank you for your interest in contributing to slkernel, the kernel of
Sunlight Linux! [README.md](README.md) explains the branches and how a
release is built; this page covers how to change them.

## Reporting Issues

- Use the GitHub issue tracker.
- Kernel issues are hardware- and config-sensitive: include `uname -r` (it
  names the exact release), the config option(s) involved, your firmware mode
  (Sunlight Linux is **UEFI**), the hardware, and the relevant `dmesg` output.

## Where a change goes

| Change | Where |
|---|---|
| Kernel source (patches) | `next` and/or `next-full` in this repository |
| Kernel `.config`, build recipe | [slpkgs]: the `linux-next` and `linux-next-full` templates on its `next` branch |
| Documentation, CI workflows | `main` in this repository |

`next` and `next-full` receive a CI merge whenever upstream moves (checked
twice a day; see README.md), so pull before you commit to them, and push
soon after.

## Commit format

Commits on the kernel branches follow the Android Common Kernel convention,
with Sunlight's own prefix:

```
SUNLIGHT: net: sock: keep 64 KiB of forward allocation on uncharge

Description:
 - What the change does and why, in one or more points.

Bug: N/A
Change-Id: I73c4603f78dcfb932deba391da988b3ff291cbbd
Signed-off-by: Your Name <you@example.com>
```

- `SUNLIGHT:` marks a Sunlight patch, so it stands apart from upstream's
  `ANDROID:`, `UPSTREAM:` and `FROMGIT:` commits.
- `Change-Id` is added by Gerrit's `commit-msg` hook. Install it once per
  clone:
  ```bash
  curl -Lo .git/hooks/commit-msg \
      https://gerrit-review.googlesource.com/tools/hooks/commit-msg
  chmod +x .git/hooks/commit-msg
  ```
- Sign off with `git commit -s`.

Commits on `main` use the plain `area: summary` form (for example
`ci: …` or `docs: …`) with a paragraph on *why*, and no `Change-Id`: commit
them with `--no-verify` so the hook does not add one.

## Porting a patch from another kernel

Sunlight takes useful patches from other distributions' kernels (see
[PATCHES.md](PATCHES.md) for Clear Linux). For each one:

1. **Check it is still needed.** Read the function it changes in the current
   tree; mainline may already do the same thing. "It applies cleanly" does
   not mean "it is still needed".
2. **Apply without guessing.** If it only applies with fuzz, confirm every
   hunk landed in the right function.
3. **Build the files it touches** with the package config.
4. **Credit the original author.** The porter is the commit author. Keep the
   original `Signed-off-by` when the patch had one; when it did not, add
   `Originally-by: Name <email>` instead. Never add a `Signed-off-by` on
   someone else's behalf: it is their certification under the DCO.
5. **Record it** in PATCHES.md, with its effect, or why it was left out.

## Config Conventions

- **One option per documented reason.** Each config change should be traceable
  to a need; explain *why* in the commit/PR.
- **Use `make olddefconfig`**, not `make defconfig`, so you change only the
  symbols you mean to.
- **Don't disable boot-critical options.** UEFI (`CONFIG_EFI*`), the
  filesystems the installer creates, initramfs/block support, and the
  low-latency preemption profile are load-bearing — changing them is a real
  decision.
- **Keep the upstream delta small.** Prefer a config change or a runtime
  setting (sysctl, slinit service options) over carrying an out-of-tree patch.

## Cutting a release

A release is a tag on `next` (`n<N>`) or `next-full` (`nf<N>`):

```bash
git switch next-full && git pull --ff-only
printf -- '-lowlatency-sunlight-nf6\n' > localversion
git commit -s -am 'SUNLIGHT: Update next keyword at localversion' \
    -m 'Description:
 - Update next keyword at localversion' -m 'Bug: N/A'
git tag 7.3.0-lowlatency-sunlight-nf6
git push origin next-full 7.3.0-lowlatency-sunlight-nf6
```

Tags are lightweight, on the `localversion` commit; the release suffix comes
from the `localversion` file alone, not from the tag. Then build the package
from the tag as described in [README.md](README.md#building-a-release).

## Testing

- Build the kernel package with the slpkgs recipe.
- **Boot-test in a VM:** QEMU + OVMF (UEFI) against a scratch image — a broken
  kernel can make the system unbootable.
- Confirm the change took effect: `uname -r`, `/boot/config-*`, and `dmesg`.

## License

Contributions to `main` are licensed under the Apache License 2.0;
contributions to the kernel branches under the kernel's GPL-2.0.

[slpkgs]: https://github.com/sunlightlinux/slpkgs
