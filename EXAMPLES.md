# EXAMPLES.md

Concrete examples for each of the four principles in [CLAUDE.md](CLAUDE.md),
grounded in real slkernel scenarios (kernel `.config`, Kconfig symbols, the
build recipe). Snippets use Kconfig / shell / diffs.

> Derived from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)
> (MIT). Examples rewritten for kernel packaging + slkernel.

---

## 1. Think Before Coding

### Example 1 — Ambiguous Feature Request

**User request:** "Enable Bluetooth."

**What LLMs often do (wrong assumption):**

```diff
-# CONFIG_BT is not set
+CONFIG_BT=y
```

**Problem:** "enable Bluetooth" underspecifies several things, and each choice
changes the result:

- **Built-in (`=y`) or module (`=m`)?** A low-latency desktop kernel usually
  ships such drivers as modules to keep the image lean.
- **Which sub-options?** `CONFIG_BT_RFCOMM`, `CONFIG_BT_BNEP`, `CONFIG_BT_HIDP`,
  and the controller driver (`CONFIG_BT_HCIBTUSB`) are all separate symbols.
- **Dependencies:** `CONFIG_BT` pulls in `CONFIG_CRYPTO`, `CONFIG_RFKILL`, etc.

**What should happen (surface the choice):**

> "Enable Bluetooth" spans several symbols. Before I touch the config:
>
> 1. Built-in or module? (I'd default to `=m`, matching the low-latency
>    profile.)
> 2. Just core `CONFIG_BT` + USB controller (`CONFIG_BT_HCIBTUSB`), or also
>    RFCOMM/BNEP/HIDP?
>
> I'll change only those symbols and run `make olddefconfig` so dependencies
> resolve without churning the rest of the config.

### Example 2 — Multiple Interpretations

**User request:** "Make the kernel smaller."

**What LLMs often do (pick silently):**

```diff
-CONFIG_MODULES=y
+# CONFIG_MODULES is not set
```

(Disabling modules entirely — a huge, breaking change.)

**What should happen:**

> "Smaller" could mean different things with different blast radius:
>
> 1. **Trim drivers** you don't ship (built-in → module, or disable unused
>    ones) — biggest safe win, but needs a target hardware list.
> 2. **Drop debug options** (`CONFIG_DEBUG_INFO`, etc.) — shrinks the image and
>    build time, low risk.
> 3. **Compression / `CONFIG_CC_OPTIMIZE_FOR_SIZE`** — modest, global effect.
>
> Which target — image size, module count, or memory footprint? I'd start with
> debug options + obviously-unused drivers, each boot-tested.

---

## 2. Simplicity First

### Example 1 — Over-abstraction

**User request:** "Add a way to toggle the low-latency option."

**What LLMs often do (overengineered):**

A whole config-fragment system: `config/base.cfg`, `config/lowlatency.cfg`,
`config/server.cfg`, plus a merge script and a recipe variable to select a
"profile" — to flip a single preemption symbol.

**Problem:** there is one shipping kernel: the low-latency desktop build. A
profile/fragment framework for one symbol is indirection nobody needs yet.

**What should happen (minimum change):**

Set the symbol directly in the one config:

```diff
-# CONFIG_PREEMPT is not set
+CONFIG_PREEMPT=y
+# CONFIG_PREEMPT_VOLUNTARY is not set
```

Then `make olddefconfig`. Introduce a fragment system if and when a *second*
shipping profile actually exists.

### Example 2 — Speculative Features

**User request:** "We use ext4; make sure the kernel can mount the root
filesystem."

**What LLMs often do (feature creep):**

```diff
+CONFIG_EXT4_FS=y
+CONFIG_BTRFS_FS=y
+CONFIG_XFS_FS=y
+CONFIG_F2FS_FS=y
+CONFIG_ZFS=y
+CONFIG_REISERFS_FS=y
+CONFIG_NILFS2_FS=y
```

**Problem:** the ask was ext4. The rest are unrequested filesystems that bloat
the kernel and widen the attack surface.

**What should happen:**

```diff
+CONFIG_EXT4_FS=y
```

Built-in (`=y`, not `=m`) so the root filesystem mounts without an initramfs
module. Add other filesystems when the installer actually offers them.

---

## 3. Surgical Changes

### Example 1 — Drive-by Regeneration

**User request:** "Enable the i915 graphics driver."

**What LLMs often do (too much):**

```
$ make defconfig      # regenerates the WHOLE config from arch defaults
$ # ...also bumps the recipe to a newer upstream tag "while here"
```

Result: hundreds of symbols change, the carefully-tuned low-latency config is
gone, and the version bump is mixed into a one-driver change.

**What should happen (surgical):**

```diff
-# CONFIG_DRM_I915 is not set
+CONFIG_DRM_I915=m
```

```
$ make olddefconfig   # resolve only the new dependencies
$ git diff .config     # confirm: i915 + its deps, nothing else
```

One driver, dependencies resolved, version untouched.

### Example 2 — Disabling a Boot-Critical Option

**User request:** "Disable the floppy driver, we don't need it."

**What LLMs often do (collateral damage):**

```diff
-CONFIG_BLK_DEV_FD=m
+# CONFIG_BLK_DEV_FD is not set
-CONFIG_EFI=y
+# CONFIG_EFI is not set        # "more cleanup of stuff we don't use"
-CONFIG_EFI_STUB=y
+# CONFIG_EFI_STUB is not set
```

**Problem:** disabling the floppy driver is fine — but the "while I'm here"
removal of `CONFIG_EFI*` makes the kernel **unbootable on a UEFI-only distro**.
The failure won't appear until install/boot.

**What should happen:**

```diff
-CONFIG_BLK_DEV_FD=m
+# CONFIG_BLK_DEV_FD is not set
```

Only the requested symbol. Boot-critical options (`CONFIG_EFI*`, the root
filesystem, block/initramfs) are never touched incidentally.

---

## 4. Goal-Driven Execution

### Example 1 — Reproduce, Then Fix

**User request:** "The new kernel builds but won't boot in UEFI."

**What LLMs often do (fix without reproducing):**

```diff
+CONFIG_EFI_STUB=y
+CONFIG_FB_EFI=y
+CONFIG_EFI_VARS=y      # plus a few more "EFI-sounding" symbols on a hunch
```

Maybe it boots, maybe not — and the config now carries cargo-cult symbols.

**What should happen (reproduce in a VM):**

```
Plan:
1. Reproduce under QEMU + OVMF (UEFI firmware) with the built kernel + a
   scratch image.
   Verify: firmware can't boot it / it panics — reproduced.
2. Diagnose: is CONFIG_EFI_STUB set? Is the root filesystem driver built-in
   (=y, not =m, if there's no initramfs)? Is CONFIG_EFIVAR_FS present? Compare
   against a known-good config.
   Verify: identify the exact missing symbol (e.g. root fs was =m with no
   initramfs).
3. Fix that one cause.
   Verify: rebuilt kernel boots to a login prompt in the VM.
4. Re-run a clean VM boot.
   Verify: boots cleanly; check dmesg for the relevant subsystem.
```

No "EFI-sounding" symbols added on a hunch — find the one that's actually
missing.

### Example 2 — Multi-Step Upstream Rebase

**User request:** "Move to the next upstream Linux release."

**What LLMs often do (all at once):**

Bump the version, regenerate the config with `make defconfig`, re-roll all
patches, and commit — untested, with the low-latency tuning lost.

**What should happen (incremental):**

```
Plan:
1. Bump the upstream version in the recipe; update the source checksum.
   Verify: the source fetches and its checksum/signature verifies.
2. Carry the existing config forward: `make olddefconfig`.
   Verify: review every NEW symbol it prompts/sets — accept deliberately, not
   blindly; git diff is understandable.
3. Re-apply the patch set; drop patches now upstream.
   Verify: each patch applies cleanly or is removed with a note.
4. Build the package.
   Verify: build succeeds.
5. Boot-test in QEMU + OVMF.
   Verify: boots to login; low-latency profile intact (CONFIG_PREEMPT still y);
   spot-check key drivers in dmesg.
```

Each step is independently checkable; the low-latency config is preserved, not
regenerated.

---

## Anti-patterns Summary

| Principle | Anti-pattern | Fix |
|---|---|---|
| Think Before Coding | `CONFIG_BT=y` for "enable Bluetooth" | Ask built-in vs module + which sub-options; `make olddefconfig` |
| Simplicity First | Enable a zoo of filesystems for an ext4 request | Set only `CONFIG_EXT4_FS=y` |
| Surgical Changes | `make defconfig` (+ version bump) to add one driver | Set the one symbol, `make olddefconfig`, diff |
| Goal-Driven | Add "EFI-sounding" symbols on a hunch for a no-boot | Reproduce in QEMU+OVMF, find the one missing symbol, fix it |

## Key Insight

The "overcomplicated" examples aren't obviously wrong — extra filesystems,
profile frameworks, and "EFI-sounding" symbols all look thorough. The problem
is **timing and blast radius**: in a kernel config, premature breadth and
incidental edits can make **every install unbootable**, and the failure surfaces
only after a successful build.

- A bad symbol doesn't fail the build — it fails the boot. Boot-test in a VM.
- Keep the delta minimal and the diff legible (`make olddefconfig`, not
  `defconfig`).
- Boot-critical options (UEFI, root filesystem, block/initramfs) are never
  touched as a side effect.

**Good slkernel changes are the smallest config/recipe delta that solves the
need while keeping the kernel bootable on a UEFI-only system — verified in a
VM, not assumed.**
