# Sunlight patches

What Sunlight carries on top of AOSP's `android-mainline` branches, and the
Clear Linux patches evaluated for it. Commits are on `next-full` unless noted;
hashes are as of `7.3.0-lowlatency-sunlight-nf5`.

## Versioning (both `next` and `next-full`)

| Patch | Why |
|---|---|
| `SUNLIGHT: Default localversion` | adds the `localversion` file |
| `SUNLIGHT: Append lowlatency keyword at localversion` | `-lowlatency-sunlight…` suffix |
| `SUNLIGHT: scripts/setlocalversion: Move localversion* files to the end` | the suffix ends the release string |
| `SUNLIGHT: scripts/setlocalversion: remove "+" tag for git repo short version` | `uname -r` equals the tag, with no `+` |
| `SUNLIGHT: Append next[-full] keyword at localversion`, `Update next keyword at localversion` | the `n<N>` / `nf<N>` suffix of each release |

## Ported from Clear Linux (`next-full` only)

From the Clear Linux kernel (`clearlinux-pkgs/linux`, 6.15.7, archived in
2025). Each was checked against the current tree before porting: that it
still applies, that mainline does not already do the same, and that the
touched files build.

| Since | Commit | Patch | Effect |
|---|---|---|---|
| nf3 | `bb1f37b` | i8042: decrease debug message level to info | less log noise |
| nf3 | `d248f55` | pci pme wakeups | PME checks every 4 s, not 1 s: fewer laptop wakeups |
| nf3 | `9d5b678` | migrate some systemd defaults to the kernel defaults | no `dummy0` / `bond0` created by default |
| nf3 | `8e44e05` | ata: libahci: ignore staggered spin-up | faster boot; `libahci.ignore_sss=0` restores it |
| nf3 | `6a0cab2` | print CPU that faults | the CPU number in crash reports |
| nf3 | `0bb904d` | don't report an error if PowerClamp run on other CPU | correct messages |
| nf3 | `4130ac4` | crypto: kdf: make the module init call a late init call | slightly faster boot |
| nf3 | `7822725` | ACPI: align slab for improved memory performance | slightly faster ACPICA |
| nf3 | `c8c796e` | thermal: intel: powerclamp: check MWAIT first | correct messages |
| nf3 | `8b32d9d` | readdir: add unlikely hint on len check | branch hint |
| nf4 | `23cba8d` | locking: rwsem: spin faster | waiters notice a released rwsem sooner |
| nf4 | `1086193` | sched/fair: better idle balancing | idle CPUs pull work more often |
| nf4 | `b978103` | init: lower the default timer slack to 50 ns | timers fire closer to their deadline; costs battery |
| nf5 | `4004a4a` | net: sock: increase default number of `_SK_MEM_PACKETS` to 1024 | default socket buffers ~832 KiB, not ~208 KiB |
| nf5 | `9df054c` | net: sock: keep 64 KiB of forward allocation on uncharge | fewer accounting round trips; more memory held per socket |

`4004a4a` changes only the *default* socket buffer sizes
(`net.core.{r,w}mem_default`). Its commit message says the maximums grow as
well; they do not: since a6d4f25888b8 ("net: set net.core.rmem_max and
net.core.wmem_max to 4 MB") the maximums are a fixed 4 MiB.

The timer slack and socket buffer defaults can be changed back where they do
not fit: per service with slinit's `timer-slack-nsec`, and system-wide with
the `net.core.{r,w}mem_default` sysctls.

### Already in mainline

Not ported, because the tree already has them or their effect:

- **0108** smpboot: reuse timer calibration — `calibrate_delay_is_known()`
  already reuses CPU0's calibration when the TSC is constant and stable; the
  patch would only add the unsafe case.
- **0130** itmt2 ADL fixes — `intel_pstate_set_itmt_prio()` already falls back
  to the HWP capabilities when CPPC reports 0xff. The patch still applies
  cleanly, and would duplicate that code.
- **0149**, **0150** select: unlikely branch hints.
- **0168** intel_idle: C1 demotion sysfs knob — now
  `/sys/devices/system/cpu/cpuidle/intel_c1_demotion`.

### Waiting for hardware to test on

- **0128** itmt_epb: use EPB to scale ITMT — changes core selection on hybrid
  Intel CPUs.
- **0173** cpuidle: PSD, sleep demotion prevention for NVMe I/O — a new
  cpuidle feature, enabled by default on every Intel system with NVMe.

### Not ported

| Patch | Why not |
|---|---|
| 0050 revert ext4 EA inode under buffer lock | a 6.15 fix; does not apply |
| 0102 longer ext4 commit age | loses more data on power loss |
| 0106 intel_idle C-state tweaks | 28 hunks only apply with fuzz, in per-CPU tables |
| 0111 TCP memory tuning | a sysctl does it |
| 0112 wait for partition and retry scan | Sunlight boots through an initramfs; `rootwait` covers it |
| 0114 boot option to allow unsigned modules | weakens module signing |
| 0115 stateless firmware loading | Clear's stateless layout |
| 0117 user.* xattrs on symlinks | changes xattr semantics, for Clear Containers |
| 0120 accept() in LIFO order | puts the exclusive waiter first, against the ordering `inet_csk_wait_for_connect()` relies on; no effect on epoll or `SO_REUSEPORT` servers |
| 0127, 0134 raid6 boot benchmark | do not apply after the move to `lib/raid/raid6`; matter only with software RAID |
| 0131 per-CPU page watermark | does not apply |
| 0133 novector | `-O3 -march=westmere` forces a CPU baseline |
| 0135 initcall_debug output | only matters with `initcall_debug` |
| 0137 libsgrowdown | changes the ELF `mmap_base`, i.e. the ASLR layout |
| 0141 EPP retune | one constant, unclear effect |
| 0156 rate-limit sched_yield | can break games running under Proton |
| 0158 clocksource checks only on AMD | weakens the TSC watchdog on Intel |
| 0164 KVM VMX late init | does not apply; small gain |
| 0166 no CPU limit in sched/fair scaling | longer slices on many cores, against low latency |
| 0174 MEMCG_CHARGE_BATCH 128 | does not build: since 6.16 the batch must fit in an `s8` |

Clear itself disabled 0109, 0118, 0125, 0132, 0138, 0147 and 0148; they were
not evaluated further.
