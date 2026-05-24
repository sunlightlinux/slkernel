# CLAUDE.md

Behavioral guidelines for LLM-assisted work on **slkernel** — the kernel
packaging and configuration for Sunlight Linux (a UEFI-only distribution).

> Derived from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)
> (MIT), based on [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876)
> on LLM coding pitfalls. Adapted for slkernel: kernel `.config` correctness,
> packaging recipes, and the fact that a bad config yields an **unbootable**
> system.

> **Status:** early bootstrap. The repo will hold the kernel build recipe and
> the Sunlight kernel config(s), tracking an upstream Linux release with a
> small, documented delta. The packaging model follows void-packages'
> `srcpkgs/linux*` templates (slpkgs is a void-packages fork).

**Tradeoff:** These guidelines bias toward caution over speed. A single wrong
Kconfig symbol can make every install fail to boot, and the failure shows up
*after* the build succeeds. When in doubt, boot-test in a VM and ask.

See [EXAMPLES.md](EXAMPLES.md) for concrete examples grounded in kernel config
and packaging scenarios.

---

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before changing config or the recipe:
- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
  Kernel work has many near-synonyms:
  - built-in (`=y`) vs module (`=m`) vs disabled (`# ... is not set`).
  - `make defconfig` (regenerate from a defconfig) vs `make olddefconfig`
    (resolve only new symbols) vs a config fragment merge.
  - the **upstream version** (Linux tag) vs the **package revision** (rebuild
    of the same source).
  - `CONFIG_PREEMPT` (low-latency desktop) vs `CONFIG_PREEMPT_RT` (real-time)
    vs `CONFIG_PREEMPT_VOLUNTARY`.
  Surface which is meant.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.
- **Check parity first.** For a feature/driver request, check how
  void-packages' `srcpkgs/linux*` templates and the upstream `Kconfig` help
  text handle it before inventing. Match their semantics.

## 2. Simplicity First

**Minimum change that solves the problem. Nothing speculative.**

- Change only the config symbols the request needs. Don't enable a whole
  subsystem to get one driver.
- No config-fragment "framework" for a single toggle.
- Don't carry out-of-tree patches unless there is no config-only way; prefer
  upstreamable changes.
- One option per documented reason — each change traceable to a need.
- Don't enable debug/instrumentation options (`CONFIG_DEBUG_*`, `CONFIG_KASAN`,
  lockdep) in the shipping config unless explicitly asked — they cost size and
  performance.

Ask yourself: "Would a kernel maintainer say this delta is bigger than it needs
to be?" If yes, shrink it.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing config or the recipe:
- **Use `make olddefconfig`, never `make defconfig`,** to normalize after a
  change — so the diff is the symbols you intended, not hundreds of churned
  ones.
- Don't reorder or reformat `.config`; don't bump the upstream version while
  fixing an unrelated symbol.
- Match the existing recipe style (shell + the template's conventions).
- Keep patch series minimal and named by intent.

**Kernel-specific — the rule that overrides convenience:**
- **Never disable boot-critical options.** UEFI (`CONFIG_EFI`, `CONFIG_EFI_STUB`,
  `CONFIG_EFIVAR_FS`), the root/boot **filesystems** the installer creates,
  block/initramfs support, and the low-latency preemption profile are
  load-bearing. A change that turns one of these off — even as a side effect of
  trimming "unused" options — can make every install unbootable.

The test: every changed symbol should trace directly to the user's request, and
the result must still boot.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Enable driver X" → "A VM kernel boots with X present (`=y`/`=m`) and the
  device works; nothing else in the config changed."
- "Fix the boot failure" → "Reproduce in a VM, identify the missing symbol,
  confirm a rebuilt kernel boots."
- "Bump to upstream Y" → "Config resolves with `make olddefconfig`, the package
  builds, and a VM boots cleanly."

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

**Verification in slkernel:**
- `make olddefconfig` then `git diff` the config — the delta is exactly what you
  meant.
- Build the kernel package (recipe-driven).
- **Boot-test in a VM** — QEMU + OVMF (UEFI firmware) against a scratch image.
  This is the only honest test that the kernel boots the distro.
- Confirm the symbol took effect: `zcat /proc/config.gz` (or `/boot/config-*`)
  and `dmesg`.

Strong success criteria let you loop independently. Weak criteria ("make it
work") require constant clarification.

---

## slkernel Project Context

### Positioning
- Kernel packaging + config for Sunlight Linux: pin an upstream Linux version,
  apply the Sunlight (low-latency) config, optionally a small patch set, and
  package it.
- Does **not** vendor the kernel source tree; references an upstream release.

### Invariants
- **UEFI-bootable.** Keep the options required to boot under UEFI and to mount
  what the installer creates.
- **Low-latency profile.** Preemption-related options are deliberate.
- **Small, documented upstream delta.** Config over patches; justify each.
- **Auditable, reproducible build.** Verified source, normalized config.

### Reference sources
- **void-packages** `srcpkgs/linux*`:
  <https://github.com/void-linux/void-packages> — packaging model (slpkgs is a
  fork of it). Check it first for *how the package is built*.
- **Upstream Linux** + `Documentation/`: authoritative for Kconfig semantics
  and option help text.
- **slinstaller / slmanifests**: the installer that lays the kernel down, and
  the tree that wires the projects together.

### Commit style
- **Bundle, don't over-split.** One commit per session of work. Commit
  messages: `area: short summary` + a paragraph on *why*, not *what* (the diff
  already shows what).
