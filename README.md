# slkernel

**The Linux kernel of Sunlight Linux: AOSP's `android-mainline` branches
plus a small set of Sunlight patches, released as tags.** CI merges
upstream into the kernel branches; each release is a `localversion` tag,
shipped as the `linux-next` and `linux-next-full` packages.

| Project | Details |
|---|---|
| **Latest release** | `7.3.0-lowlatency-sunlight-n2` (`linux-next`) and `7.3.0-lowlatency-sunlight-nf5` (`linux-next-full`), both tagged 2026-11-22; tags only, no GitHub releases |
| **Release contract** | `uname -r` of a kernel built from a tag is the tag name — see [the release-tag spec][tags-spec] |
| **Upstream** | [AOSP kernel/common]: `android-mainline` and `android-mainline-desktop`, Linux 7.3.0 at every tag so far |
| **Requires** | x86_64 with UEFI firmware (Sunlight is UEFI-only); the build recipe and `.config` live in [slpkgs], on its `next` branch |
| **License** | Kernel branches: GPL-2.0 (Linux); documentation and CI on `main`: [Apache 2.0](LICENSE) |

The upstream is Linux mainline as carried by the Android Common Kernel
(AOSP's own commits are prefixed `ANDROID:`, `UPSTREAM:`, `FROMGIT:`).
Sunlight's commits are prefixed `SUNLIGHT:`: on `next`, only the patches
that make `uname -r` equal the tag; on `next-full`, those plus fifteen
performance and log-noise patches ported from Clear Linux, each listed in
[PATCHES.md](PATCHES.md). slkernel takes over from `linux-sunlight`, the
organization's standalone kernel tree, which is being retired at 7.4.

**Deliberately out of scope:** choosing or bumping a Linux release (the
tree is whatever AOSP's branches carry; history is merged, never rebased);
the kernel `.config` and the package recipe, which belong to slpkgs; and
patches with a config-only or runtime alternative (a sysctl, a slinit
service option), or that weaken a protection such as module signing.

> **New here?** Read [Branches](#branches) and
> [Versions and tags](#versions-and-tags) below: which branch is which, and
> what a tag promises. To change a kernel branch, then read
> [CONTRIBUTING.md](CONTRIBUTING.md).

## Branches

The kernel source lives on its own branches; `main` holds only the
documentation and the CI that keeps those branches current.

| Branch | What it is | Package |
|---|---|---|
| `main` | documentation and CI workflows | — |
| `mainline-check` | AOSP `android-mainline`, merged by CI | — |
| `mainline-check-full` | AOSP `android-mainline-desktop`, merged by CI | — |
| `next` | `mainline-check` + Sunlight's versioning patches | `linux-next` (default kernel) |
| `next-full` | `mainline-check-full` + versioning and [PATCHES.md](PATCHES.md) patches | `linux-next-full` |

Read the kernel branches with `git show <branch>:<path>` or a
`git worktree`; switching a `main` checkout to them replaces the whole tree.

## How the source stays current

Two GitHub Actions workflows on `main` move upstream changes forward, each
as a `SUNLIGHT: Mainline <date> - ANDROID` merge commit:

| Workflow | When | Merges |
|---|---|---|
| **Sync android-mainline** | scheduled at 03:00 and 15:00 UTC, or by hand | `android-mainline` → `mainline-check`, `android-mainline-desktop` → `mainline-check-full` |
| **Sync next** | after every run of the first, or by hand | `mainline-check` → `next`, `mainline-check-full` → `next-full` |

GitHub may start a scheduled run hours late or skip it, hence two a day; a
run with nothing new exits early. Both merge with `git merge-tree` in a
blobless clone and push only when there is something new. A merge conflict
fails the job instead of guessing: merge the branches by hand, push, and
the next run carries on. Because `next` and `next-full` move on their own,
pull before committing to them locally.

## Versions and tags

Every release of the kernel is a tag on its branch:

```
7.3.0-lowlatency-sunlight-n<N>    on next       (linux-next)
7.3.0-lowlatency-sunlight-nf<N>   on next-full  (linux-next-full)
```

| Tag | Date | Branch |
|---|---|---|
| `n1`, `n2` | 2026-11-15, 2026-11-22 | `next` |
| `nf1` | 2026-11-15 | `next-full` |
| `nf2` .. `nf5` | 2026-11-22 | `next-full` |

Each tag sits on the commit that writes the `localversion` file at the top
of the tree, and the release suffix comes from that file alone:

```
$ git show 7.3.0-lowlatency-sunlight-nf5:localversion
-lowlatency-sunlight-nf5
$ uname -r          # a kernel built from that tag
7.3.0-lowlatency-sunlight-nf5
```

The tags are lightweight. Upstream's `scripts/setlocalversion` only
considers *annotated* tags when it looks for a tag matching
`localversion`, so it never finds these; the two `setlocalversion`
patches in [PATCHES.md](PATCHES.md) put the file's contents at the end of
the release string and drop the `+` it would otherwise append.

`7.3.0` is the Linux version of the tree; `n<N>` and `nf<N>` are separate
counters, one per branch. The branch tips move past the last tag with
every upstream merge, so a tag, not a branch, names a shipped kernel.

## Building a release

The build recipe is in slpkgs, as `linux-next` and `linux-next-full`
templates (each with `-headers` and `-dbg` subpackages) modelled on
void-packages' `srcpkgs/linux*`, on slpkgs' `next` branch
([`linux-next`][tpl-next], [`linux-next-full`][tpl-next-full] at the
commit that published them).

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

> **Warning:** a broken kernel can leave a system unbootable, and the
> failure shows only after the build succeeds. Boot every new kernel in a VM
> (QEMU + OVMF, since Sunlight is UEFI-only) before publishing it. A newly
> installed kernel sits next to the previous one, which stays in the GRUB
> menu as a fallback.

## Configuration

The kernel `.config` is not on any branch of this repository. It is the
slpkgs file [`srcpkgs/linux-next/files/x86_64-dotconfig`][dotconfig];
`linux-next-full` carries a byte-identical copy. The templates'
`do_configure` copies it to `.config` and runs `make olddefconfig`
against the tag's tree, so the file is only the input: the kernel's
configuration is what `olddefconfig` makes of it for 7.3.0.

| Area | The file in slpkgs | The kernel built from a tag |
|---|---|---|
| Origin | Void's `linux6.18` dotconfig as of void-packages `c1c64954f37` ("linux6.18: update to 6.18.32"), with one line changed | brought up to 7.3.0 by `olddefconfig` |
| Release suffix | `CONFIG_LOCALVERSION=""` (Void's is `"_1"`), so `localversion` alone supplies the suffix | unchanged |
| Preemption | `CONFIG_PREEMPT_VOLUNTARY=y`, `# CONFIG_PREEMPT is not set`, `CONFIG_PREEMPT_DYNAMIC=y`, `CONFIG_HZ=1000` | `CONFIG_PREEMPT=y`, `CONFIG_PREEMPT_DYNAMIC=y`, `CONFIG_HZ=1000`: full preemption by default, `preempt=lazy` the only alternative at boot |
| Storage | disk controllers and filesystems as modules (`CONFIG_EXT4_FS=m`, `CONFIG_SATA_AHCI=m`; `CONFIG_BLK_DEV_NVME=y`), so an initramfs is required; Sunlight's base brings dracut through `linux-base` | unchanged |
| UEFI | `CONFIG_EFI=y`, `CONFIG_EFI_STUB=y`, `CONFIG_EFIVAR_FS=y` | unchanged |

The preemption row differs because in the 7.3.0 tree
`PREEMPT_VOLUNTARY` depends on `!ARCH_HAS_PREEMPT_LAZY`, which
`arch/x86/Kconfig` selects, so `olddefconfig` cannot keep the file's
choice and resolves it to `CONFIG_PREEMPT=y`. The right-hand column was
measured by running `make ARCH=x86_64 olddefconfig` with that file on a
`git archive` of the `n2` and `nf5` tags (both gave the same values);
no built kernel's `/proc/config.gz` was read. On a running kernel,
`zcat /proc/config.gz | grep -E 'CONFIG_PREEMPT[=_ ]'` and the
`Dynamic Preempt:` line in `dmesg` confirm it.

**Correction (2026-11-28).** Before commit `7c4af55` this section
stated `CONFIG_PREEMPT=y` with `PREEMPT_DYNAMIC` as the configuration;
`7c4af55` marked that as not verified, because Void's file sets
`CONFIG_PREEMPT_VOLUNTARY=y`. The original statement is right about the
built kernel and wrong about the file: the file in slpkgs does not set
`CONFIG_PREEMPT=y`; `olddefconfig` does, because it cannot keep
voluntary preemption on x86 in this tree. Setting voluntary preemption
in the file therefore has no effect on these kernels.

What is and is not known about the config is tracked in sldocs'
[kernel configuration record][config-doc]. A dedicated Sunlight config,
in place of the adapted Void one, is still to come.

## Where this fits

slkernel is one of the projects of the Sunlight Linux tree, wired together
by [slmanifests]:

```bash
repo init -u https://github.com/sunlightlinux/slmanifests.git -b main
repo sync -j4
# slkernel lands at kernel/
```

## Documentation

| Document | What it covers |
|---|---|
| [PATCHES.md](PATCHES.md) | the Sunlight patches on `next` and `next-full`, and every Clear Linux patch evaluated for them, with why it was ported or not |
| [CONTRIBUTING.md](CONTRIBUTING.md) | where a change goes, commit format and Change-Id hook, porting a patch, cutting a release |
| [SECURITY.md](SECURITY.md) | what is security-relevant here, and how to report an issue |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | community standards |
| [CLAUDE.md](CLAUDE.md) | guidelines for LLM-assisted work on the kernel and its config |
| [EXAMPLES.md](EXAMPLES.md) | worked examples for those guidelines |
| [Release-tag spec][tags-spec] (sldocs) | what a tag promises a packager, checked against the repository |
| [Kernel configuration record][config-doc] (sldocs) | what is known about the `.config` |

## Testing

There is no test suite in this repository; the test of a kernel is booting
it. Build the package with the slpkgs recipe, boot it in QEMU with OVMF
against a scratch image, and confirm the change took effect with
`uname -r`, `/boot/config-*` (or `/proc/config.gz`) and `dmesg`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Kernel patches go to `next` and/or
`next-full` with a `SUNLIGHT:` subject and a Gerrit `Change-Id`; changes to
the documentation and workflows go to `main`.

## License

The documentation and workflows on `main` are under the
[Apache License 2.0](LICENSE). The kernel on the other branches is Linux,
licensed GPL-2.0 (see `COPYING` there); Sunlight's patches to it are under
the same terms.

[slpkgs]: https://github.com/sunlightlinux/slpkgs
[tpl-next]: https://github.com/sunlightlinux/slpkgs/tree/21a8fc38ba87061cb9328e69643749ba6e5f66a6/srcpkgs/linux-next
[tpl-next-full]: https://github.com/sunlightlinux/slpkgs/tree/21a8fc38ba87061cb9328e69643749ba6e5f66a6/srcpkgs/linux-next-full
[dotconfig]: https://github.com/sunlightlinux/slpkgs/blob/21a8fc38ba87061cb9328e69643749ba6e5f66a6/srcpkgs/linux-next/files/x86_64-dotconfig
[slmanifests]: https://github.com/sunlightlinux/slmanifests
[AOSP kernel/common]: https://android.googlesource.com/kernel/common
[tags-spec]: https://github.com/sunlightlinux/sldocs/blob/1fd7989a20e869d59201d4742159faef991dad4c/specs/slkernel-release-tags.md
[config-doc]: https://github.com/sunlightlinux/sldocs/blob/1fd7989a20e869d59201d4742159faef991dad4c/config/slkernel-kernel-config.md
