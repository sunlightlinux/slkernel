# slkernel

The kernel of **Sunlight Linux**: a low-latency Linux kernel built from the
Android Common Kernel's mainline branches, carrying a small set of Sunlight
patches, and packaged for the distribution by [slpkgs].

The default kernel of a Sunlight install comes from here (`linux-next`), as
does the optional `linux-next-full`.

## Branches

The kernel source lives on its own branches; `main` holds only this
documentation and the CI that keeps those branches current.

| Branch | What it is | Package |
|---|---|---|
| `main` | documentation and CI workflows | — |
| `mainline-check` | AOSP `android-mainline`, merged daily | — |
| `mainline-check-full` | AOSP `android-mainline-desktop`, merged daily | — |
| `next` | `mainline-check` + Sunlight patches | `linux-next` (default kernel) |
| `next-full` | `mainline-check-full` + Sunlight patches | `linux-next-full` |

The upstream is [AOSP kernel/common]: Linux mainline plus the Android Common
Kernel's own changes (commits prefixed `ANDROID:`, `UPSTREAM:`, `FROMGIT:`).
`next-full` also carries the performance patches listed in
[PATCHES.md](PATCHES.md); `next` carries only the versioning patches.

## How the source stays current

Two GitHub Actions workflows on `main` move upstream changes forward, each as
a `SUNLIGHT: Mainline <date> - ANDROID` merge commit:

1. **Sync android-mainline** (daily, 03:00 UTC):
   `android-mainline` → `mainline-check`,
   `android-mainline-desktop` → `mainline-check-full`.
2. **Sync next** (after every run of the first):
   `mainline-check` → `next`, `mainline-check-full` → `next-full`.

Both merge with `git merge-tree` in a blobless clone and push only when there
is something new. A merge conflict fails the job instead of guessing: merge
the branches by hand, push, and the next run carries on. Because `next` and
`next-full` move on their own, pull before committing to them locally.

## Versions and tags

Every release of the kernel is a tag on its branch:

```
7.3.0-lowlatency-sunlight-n<N>    on next       (linux-next)
7.3.0-lowlatency-sunlight-nf<N>   on next-full  (linux-next-full)
```

The suffix comes from the `localversion` file at the top of the tree, so the
running kernel reports the tag it was built from:

```
$ uname -r
7.3.0-lowlatency-sunlight-nf5
```

`7.3.0` is the Linux version of the tree; `n<N>`/`nf<N>` counts Sunlight
releases of it. The package version stays `7.3.0` and its revision follows
`<N>`.

## Building a release

The build recipe is in slpkgs, not here: `srcpkgs/linux-next` and
`srcpkgs/linux-next-full` (each with `-headers` and `-dbg` subpackages),
modelled on void-packages' `srcpkgs/linux*`.

1. Commit the new suffix to `localversion` and tag the commit (see
   [CONTRIBUTING.md](CONTRIBUTING.md#cutting-a-release)).
2. Make the source tarball from the tag:
   ```bash
   git archive --format=tar.gz --prefix=linux-next-full-7.3.0/ \
       7.3.0-lowlatency-sunlight-nf5 > linux-next-full-7.3.0-nf5.tar.gz
   ```
   The prefix is the package name and version, without the suffix. Each tag
   gets its own tarball name, so earlier revisions keep their checksums.
3. Publish the tarball to the Sunlight mirror's `sources/` directory.
4. In the slpkgs template, bump `revision`, the distfile name, `checksum`
   and `_kernver` (which must equal the `uname -r` above).
5. Build with `./xbps-src pkg linux-next-full`, then sign and publish the
   packages to the mirror.

> ⚠️ A broken kernel can leave a system unbootable, and the failure shows only
> after the build succeeds. Boot every new kernel in a VM (QEMU + OVMF, since
> Sunlight is UEFI-only) before publishing it. A newly installed kernel sits
> next to the previous one, which stays in the GRUB menu as a fallback.

## Configuration

The kernel `.config` is kept in slpkgs as
`srcpkgs/linux-next*/files/x86_64-dotconfig`. It started from void-packages'
`linux6.18` config (with `CONFIG_LOCALVERSION=""`, since `localversion`
supplies the suffix) and is brought up to the tree's version with
`make olddefconfig` at build time. The profile that matters:

- **Low latency:** `CONFIG_PREEMPT=y` with `PREEMPT_DYNAMIC` (full preemption
  at runtime) and `CONFIG_HZ=1000`.
- **Modular storage:** disk controllers and filesystems are modules, so the
  kernel needs an initramfs. Sunlight's base depends on `linux-base`, which
  brings dracut; the kernel's install hooks generate the initramfs.
- **UEFI boot:** the EFI options and the filesystems the installer creates
  must stay enabled.

A dedicated Sunlight config, in place of the adapted Void one, is still to
come.

## Where this fits

slkernel is one of the projects of the Sunlight Linux tree, wired together
by [slmanifests]:

```bash
repo init -u https://github.com/sunlightlinux/slmanifests.git -b main
repo sync -j4
# slkernel lands at kernel/
```

## Documentation

- [PATCHES.md](PATCHES.md) — the Sunlight patches, and the Clear Linux patches
  evaluated for them
- [CONTRIBUTING.md](CONTRIBUTING.md) — commit format, patches, cutting a release
- [SECURITY.md](SECURITY.md) — reporting security issues
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) — community standards
- [CLAUDE.md](CLAUDE.md) / [EXAMPLES.md](EXAMPLES.md) — guidelines for
  LLM-assisted work on the kernel

## License

The documentation and workflows on `main` are under the
[Apache License 2.0](LICENSE). The kernel on the other branches is Linux,
licensed GPL-2.0 (see `COPYING` there); Sunlight's patches to it are under
the same terms.

[slpkgs]: https://github.com/sunlightlinux/slpkgs
[slmanifests]: https://github.com/sunlightlinux/slmanifests
[AOSP kernel/common]: https://android.googlesource.com/kernel/common
