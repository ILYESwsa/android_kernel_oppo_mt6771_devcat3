# BPF Backport Project — OPPO F11 (MT6771 / Helio P60, kernel 4.14.336)

Branch: `bpf-backport` (based on `clean` @ 35c368b, which carries the arraymap.c fix)
Status: **Phase 0 — enablement (landing on this branch)**
Method: **curated upstream cherry-picks** (the Cilium model). NOT a subsystem merge
(the `bpf-5.10` approach — wholesale 5.10 BPF grafts onto a 4.14 vendor tree — is
documented below as the anti-pattern that broke this SoC family's vendor glue).

---

## 1. Goal

Bring this 4.14.336 Android vendor kernel from **zero eBPF** to a modern, tooling-ready
BPF platform, in small verifiable steps, without breaking the vendor ABI
(vermagic must stay `4.14.336` so the 9 stock `.ko` modules keep loading) and
without touching the vendor-glue subsystems (mm, sched, vmalloc) that killed the
previous attempt.

## 2. Current state of this tree (measured, not guessed)

Measured on `clean` (d1396341 + arraymap fix):

| Item | State |
|---|---|
| `oppo_f11_defconfig` BPF lines | **0** — no `CONFIG_BPF_SYSCALL`, no JIT, no cgroup-bpf. The kernel ships with eBPF completely **off** (OPPO relied on xt_qtaguid for stats; Android 9 allowed that fallback) |
| Tracing config | **0** — no FTRACE / KPROBES / PERF_EVENTS lines in defconfig |
| `include/uapi/linux/bpf.h` | stock 4.14 BPF uapi: **16 map types, 14 prog types, ~55 helpers** (full native 4.14 set incl. SOCKMAP, LRU, LPM_TRIE, map-in-map, XDP) |
| `kernel/bpf/Makefile` | full native 4.14 object list incl. `arraymap.c` (with our non-zero-index fix), sockmap, stackmap, lpm, lru |
| arm64 JIT | `HAVE_EBPF_JIT` selected (arch/arm64/Kconfig:86); `BPF_JIT_ALWAYS_ON` available (init/Kconfig:1561) |
| Probe deps on arm64 | `HAVE_KPROBES`, `HAVE_PERF_EVENTS`, `HAVE_REGS_AND_STACK_ACCESS_API` all selected (arch/arm64/Kconfig:109-115) |
| Debug info | `CONFIG_DEBUG_INFO=y` already on → vmlinux BTF is **generatable with pahole** (needed for Phase 2, no kernel changes) |
| Kernel headers on device | `CONFIG_IKHEADERS=m` (kheaders.ko) already building |
| SELinux | `CONFIG_SECURITY_SELINUX_DEVELOP=y` (permissive-able; see §6) |
| android-4.14-stable BPF backports | Google merged only **fixes**, no features, through 4.14.336 — our uapi proves it (no SOCKHASH/QUEUE/STACK/CGROUP_STORAGE etc.); re-verified directly on the archived `deprecated/android-4.14` branch (zero feature files in kernel/bpf) |

Meaning: **flipping on 4.14-native eBPF is pure defconfig work** — every option
resolves in this tree (the exact line set below was proven to build on this SoC
family by the earlier bpf-5.10-branch build, which used the same Kconfig plumbing).

## 3. Prior art — what everyone else already learned

1. **Cilium (the canonical precedent).** Cilium v1.0–v1.8 supported kernels
   ≥ 4.9.17 and kept 4.9/4.14 alive by **cherry-picking upstream BPF commits onto
   stable trees** (their v1.8 "advanced features" table pins features to specific
   stable versions like 4.19.57 / 5.1.16 / 5.2 — those pins ARE backports).
   Their repos: `cilium/linux` (feature branches) + `cilium/kernel-bpf-ci` (CI).
   Timeline of official support: cilium/ebpf dropped 4.9 CI in Apr 2023 (#1007),
   removed 4.14/4.19 from CI in Feb 2025 (#1699, both EOL) while keeping exported
   APIs; Cilium now wants ≥ 5.10 (or RHEL's 4.18, which brings us to #2).
2. **RHEL 8** — a 4.18 base that Cilium maintainers describe as "functionally
   similar to a v5.4 kernel" thanks to *massive* curated BPF backports. Proof that
   a vendor kernel can carry 5.x-level BPF under an old version number — exactly
   our plan.
3. **Amazon Linux 2** — a **4.14** kernel maintained for years (ALAS2 advisories
   into 2025-2026) with BPF features backported; it is also inside BTFHub's
   supported range (4.14+), i.e. BTF on 4.14 is a solved problem there (kernel-side
   backport, see Phase 4).
4. **BTFHub / BTFGen (Aqua Security)** — the key architectural insight: libbpf can
   take an **external BTF file** (support added in libbpf commit 4920031c8809) and
   resolve CO-RE relocations in userspace. **CO-RE does not need kernel-side BTF.**
   BTFHub ships BTF for distro kernels 4.14 → 6.x that lack embedded BTF. Tracee
   and eunomia ("bpf-compatible") productionize this pattern.
5. **iovisor bcc `docs/kernel-versions.md`** — the canonical feature → kernel
   version matrix, with **upstream commit SHAs** — our cherry-pick source map
   (§4).
6. **iovisor-dev list, Dec 2018, "Backporting ebpf"** — community thread where
   someone backported eBPF even onto RHEL 7.3's 3.10 kernel via a
   `bpf_3.18_features` branch; documents that backporting BPF to old vendor
   kernels is an established (if painful) practice.
7. **Android Common Kernel model** — Google's own ACK = LTS + "backports and
   cherry-picks of upstream functionality needed for Android features"
   (source.android.com). Cherry-picking into an old base is the *sanctioned*
   Android way; subsystem merges are not.
8. **The anti-pattern (this repo's own history)** — real-omar's `bpf-5.10`
   branch grafted the 5.10 BPF stack onto this same 4.14.336 vendor tree. It
   failed to build on vendor glue: 3-arg `__vmalloc` vs 5.9 signature,
   `mm->nr_ptes` removal by the mm backport, dead 4.14 `LINUX_VERSION_CODE`
   paths in vendor files. Lesson recorded: **the breakage came from the mm/timer
   dependency fan-out, not from BPF itself.** Every phase below avoids that class.
9. **Phone-kernel survey (task 17 — every 4.14 phone family, checked directly
   in-repo, not via search snippets).**

   | Phone tree | Base | kernel/bpf carries | Reading |
   |---|---|---|---|
   | Google ACK `deprecated/android-4.14` — base of ALL 4.14 phones | 4.14 | stock set only (btf/cgroup_storage/queue_stack/ringbuf all absent) | Google never shipped BPF *features* into android-4.14 — fixes only |
   | Pixel 4 `android-msm-coral-4.14-android10` | 4.14 | stock set only | Google's own 4.14 phone runs Android 10's eBPF networking (traffic accounting etc.) entirely on stock 4.14 BPF |
   | OnePlus 7 stock OSS (`OnePlusOSS/android_kernel_oneplus_sm8150`) | 4.14 | stock set only | vendor tree untouched |
   | Xiaomi Mi 9 (`LineageOS/android_kernel_xiaomi_sm8150` @lineage-23.2) | 4.14.356-openela-rc1 | stock set only | OpenELA base alone adds NO BPF features — additions below are device-maintainer work |
   | Redmi Note 8 Pro / begonia, Helio G90T — closest MTK cousin (`MiCode begonia-p-oss`) | 4.14 | stock set only (file list identical to this tree) | the whole MTK 4.14 generation ships stock BPF, off in defconfig |
   | `hrihq/Positron-Sweet2` — Redmi Note 12 Pro 4G custom kernel (2025) | 4.14.355-openela | stock set only — its "BPF enabled" = **defconfig enablement** | what the phone world calls "bring BPF to a 4.14 phone" is exactly our Phase 0 |
   | **Galaxy S10/Note10/F62 (`LineageOS/android_kernel_samsung_exynos9820` @lineage-23.2)** | **4.14.356-openela-rc1** | **curated feature backports, all built: BTF (`btf.o` + `sysfs_btf.o`), queue/stack maps, cgroup local storage, xskmap, cpumap, reuseport_array, disasm** | **THE phone precedent for this project's method.** btf.c = 40-commit UPSTREAM/BACKPORT cherry-pick series 2018→2023 (Var/DataSec `4ad16b1bc3e6`, spin_lock `09b33fdc6cc8`, `BPF_BTF_GET_NEXT_ID` `5d4bb93d4f43`); queue/stack entered via `dd4b104f5150` + 7 fix commits; defconfig = BPF_SYSCALL + JIT_ALWAYS_ON + CGROUP_BPF + kprobes/uprobes/perf/BPF_EVENTS, i.e. our Phase 0a/0b |
   | **OnePlus 7 (`LineageOS/android_kernel_oneplus_sm8150` @lineage-24.0)** | **4.14.357-openela** | **near-5.10-class stack: iterators, ringbuf, trampolines, bpf_lsm, struct_ops, dispatcher, BPF_PRELOAD** | phone-world proof that even the "hard" tier is *possible* on 4.14 (Qualcomm vendor glue, large community, commits through mid-2025). Deliberately NOT our path — see guardrails |

   No MediaTek-based phone kernel was found on an OpenELA or BPF-backport base
   anywhere — this branch is first-of-kind for MTK 4.14.
10. **OpenELA (Open Enterprise Linux Association — CIQ/Oracle/SUSE/Canonical,
    founded 2023).** In March 2024 OpenELA committed to maintaining 4.14 past
    its EOL (`openela/kernel-lts` on GitHub); the series continues past 4.14.336
    (4.14.35x today) and the phone custom-kernel scene is already rebasing onto
    it (Velvet/sm6250, Xcalibur/violet, Positron/sweet2). Optional future
    batch for this tree: cherry-pick OpenELA 4.14.337+ fixes with `SUBLEVEL`
    pinned at 336 to preserve the module ABI (vermagic guardrail).

## 4. Feature → kernel-version matrix (cherry-pick source map)

From bcc `kernel-versions.md` (commit SHAs = upstream cherry-pick targets):

| Feature | Upstream | Commit | Backport difficulty on our tree |
|---|---|---|---|
| cgroup device controller | 4.15 | – | easy (net/core/filter.c + cgroup.c) |
| bpf2bpf calls (verifier) | 4.16 | `cc8b0b92a169` | medium (needs arm64 JIT subprog support too — verify exact arm64 commit) |
| connect4/connect6 hooks | 4.17 | – | easy (self-contained: uapi + filter.c + cgroup.c) |
| bind4/bind6 hooks | 4.17 | `4fb7d0b8e2a1` | easy |
| sk_msg (SK_MSG verdict) | 4.17 | – | easy |
| BTF (kernel-side) | 4.18 | – | **hard but phone-proven** (LineageOS S10 carries it on 4.14-openela — Phase 4 only) |
| AF_XDP | 4.18 | – | medium (new net family; drivers opt-in, generic mode works) |
| SOCKHASH map | 4.18 | `81110384441a` | easy |
| `get_current_cgroup_id` | 4.18 | – | trivial helper |
| CGROUP_STORAGE map | 4.19 | `de9cbbaadba5` | easy-medium |
| sendmsg4/6 hooks | 4.19 | – | easy |
| reuseport prog + map | 4.19 | – | medium |
| QUEUE / STACK maps | 4.20 | – | easy — phone-proven donor: LineageOS S10 `dd4b104f5150` + fix series |
| spin_lock helpers | 5.1 | – | hard (needs BTF in map values → Phase 4) |
| SK_STORAGE map | 5.2 | `6ac99e8f23d4` | hard (BTF-based map) → Phase 4 |
| cgroup sysctl hook | 5.2 | – | medium |
| getsockopt/setsockopt hooks | 5.3 | – | easy-medium |
| bounded loops | 5.3 | – | **DO NOT** (verifier core rework) |
| send_signal helper | 5.3 | – | medium |
| fentry/fexit + trampolines | 5.5 | – | **DO NOT** (arch trampoline + BTF) |
| mmap() array maps / global data (kernel side) | 5.5 | – | medium |
| RINGBUF map | 5.8 | `457f44363a88` | medium (self-contained; enterprise kernels backport it routinely) |
| sk_lookup hook | 5.9 | – | medium |
| iterators | 5.8 | – | **DO NOT** (BTF + seq infra) |
| BPF LSM | 5.7 | – | **DO NOT** |
| kfuncs | 5.12+ | – | **DO NOT** |

Note for userspace: **libbpf global variables (`.data/.rodata/.bss`) already work
on 4.14** — libbpf implements them as internal ARRAY maps initialized at load
time; kernel mmap support (5.5) is only a performance optimization.

## 5. The plan

### Phase 0 — Enable native 4.14 eBPF (defconfig only, this branch starts here)
- **0a**: `CONFIG_BPF=y`, `BPF_SYSCALL=y`, `BPF_JIT=y`, `BPF_JIT_ALWAYS_ON=y`,
  `CGROUP_BPF=y`, `NET_CLS_BPF=y`, `NET_ACT_BPF=y`, `STREAM_PARSER=y`,
  `BPF_STREAM_PARSER=y`, `TEST_BPF=m`
- **0b** (separate commit, bisectable): `FTRACE=y`, `KPROBES=y`,
  `KPROBE_EVENTS=y`, `UPROBES=y`, `UPROBE_EVENTS=y`, `PERF_EVENTS=y`,
  `BPF_EVENTS=y`, `STACKTRACE=y`, `FTRACE_SYSCALLS=y`
- Full build validation (vermagic unchanged: `4.14.336 SMP preempt mod_unload
  modversions aarch64`), then on-device smoke tests: `modprobe test_bpf`,
  map create/update via bpftool/libbpf, first kprobe program.
- Success criterion: a libbpf "minimal" kprobe example runs on the phone.

### Phase 1 — Networking backports (4.15–4.20 window, the Cilium playbook)
One upstream feature per commit batch, build + boot-test each:
1. connect4/connect6 + `BPF_PROG_TYPE_CGROUP_SOCK_ADDR` (4.17)
2. bind4/bind6 (4.17)
3. SOCKHASH (4.18, `81110384441a`)
4. CGROUP_STORAGE + percpu variant (4.19, `de9cbbaadba5` + `b741f1630346`)
5. sendmsg4/6 (4.19)
6. QUEUE/STACK maps (4.20)
All live in `kernel/bpf/` + `net/core/filter.c` + `include/uapi` — no mm, no
timer API, no sched changes.
   Donor shortcut: the LineageOS S10 tree already carries 4.14-adapted cherry-
   picks of several Phase 1 items (queue/stack maps, cgroup local storage) —
   see §3 item 9.

### Phase 2 — CO-RE without kernel-side BTF (userspace only, zero risk)
- Generate BTF for **our exact kernel build** with pahole from the vmlinux we
  already produce (`CONFIG_DEBUG_INFO=y` is on).
- Ship it (e.g. `/data/local/tmp/btf/vmlinux` or embedded via BTFGen) and point
  libbpf at it (`bpf_object__set_btf_custom_path` / `BTF_PATH`).
- Result: modern CO-RE tools (libbpf-tools, bpftrace-lite class) run on this
  kernel with **no kernel code changes at all**.
- Toolchain pinning: libbpf ≤ 1.4-era (last releases CI-testing 4.14-class
  kernels) or feature-probing current builds.

### Phase 3 — Quality-of-life backports (medium risk, each optional)
- RINGBUF (5.8, `457f44363a88`) — self-contained map; enterprise distros
  backport it routinely, and LineageOS's OnePlus 7 tree proves it on a 4.14
  phone (4.14.357-openela).
- mmap() array maps / true global data (5.5 series).
- bpf2bpf calls — verifier side `cc8b0b92a169` + arm64 JIT subprog bits
  (pin exact arm64 commit before starting).

### Phase 4 — Kernel-side BTF (optional, hard, AL2-proven)
Backport BTF (4.18+ commits) so the kernel itself carries BTF
(`/sys/kernel/btf/vmlinux`), unlocking SK_STORAGE, spin_locks, BTF-typed maps.
AL2 proved 4.14 can carry it, and LineageOS's Galaxy S10 tree carries it on a
4.14 phone (btf.o built with BPF_SYSCALL; sysfs_btf.o exists but their
defconfig leaves CONFIG_DEBUG_INFO_BTF off — syscall-side BTF only). Touches
kallsyms/module glue — the risk class that
killed bpf-5.10 — so it is deliberately last and optional (Phase 2 makes it
unnecessary for CO-RE).

### Not attempting (documented refusal)
bounded loops (verifier core rework), fentry/fexit/trampolines, BPF LSM,
iterators, kfuncs, task/inode storage. Each rewrites core verifier/arch
internals — that is the wholesale-subsystem trap, and this project is about
curated steps.

## 6. Guardrails

- One feature per commit batch; every batch = full build (vermagic check) + zip.
- No mm/vmalloc/sched/timer API consumption. If a cherry-pick touches those, it
  gets adapted or dropped — never the other way around.
- `LOCALVERSION` untouched → module ABI (`/vendor` blobs, our 9 `.ko`) stays
  loadable.
- SELinux: ColorOS is enforcing; unconfined `bpf()` from shell needs permissive
  mode (`setenforce 0` on a `SELINUX_DEVELOP=y` build) or a custom policy domain.
  Kernel-side we changed nothing about policy — this is a runtime/user decision.
- Unprivileged BPF: 4.14 predates `kernel.unprivileged_bpf_disabled=2` semantics;
  set it in an init script if the ROM allows.
- Every phase has a rollback: revert the batch, rebuild — tree returns to clean.

## 7. Validation protocol (per batch)

1. Sandbox build: chunked `make` (clang-r383902, `CLANG_TRIPLE=aarch64-linux-gnu-`,
   `KCFLAGS=-Wno-strict-prototypes`), defconfig → Image.gz-dtb + 9 modules,
   vermagic check on all modules.
2. On-device: boot test → `uname -r` unchanged → `modprobe test_bpf` →
   bpftool `prog`/`map` listing → libbpf sample for the specific feature.
3. Any boot-loop: revert batch (previous zip stays flashable).

## 8. References

- bcc feature matrix: https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md
- Cilium v1.8 requirements (4.9.17 era + backport pins): https://docs.cilium.io/en/v1.8/operations/system_requirements/
- Cilium current: https://docs.cilium.io/en/stable/operations/system_requirements
- cilium/ebpf CI dropping 4.14/4.19: https://github.com/cilium/ebpf/issues/1699 (and 4.9: #1007)
- BTFHub: https://github.com/aquasecurity/btfhub + btfhub-archive
- libbpf external BTF commit: https://github.com/libbpf/libbpf/commit/4920031c8809
- libbpf: https://github.com/libbpf/libbpf (mirror of tools/lib/bpf)
- CO-RE explainer: https://nakryiko.com/posts/bpf-portability-and-co-re/
- Android common kernel model: https://source.android.com/docs/core/architecture/kernel/gki-android-common-kernels
- iovisor-dev "Backporting ebpf" thread (RHEL 7.3 precedent): https://lists.iovisor.org (Dec 2018)
- Phone survey (task 17, direct in-repo checks):
  ACK (stock BPF only): https://android.googlesource.com/kernel/common/+/refs/heads/deprecated/android-4.14
  Pixel 4 (stock BPF runs A10 eBPF): https://android.googlesource.com/kernel/msm/+/refs/heads/android-msm-coral-4.14-android10
  Galaxy S10/Note10/F62 (BTF + queue/stack + xskmap backports on 4.14.356-openela): https://github.com/LineageOS/android_kernel_samsung_exynos9820
  OnePlus 7 (near-5.10 BPF stack on 4.14.357-openela): https://github.com/LineageOS/android_kernel_oneplus_sm8150
  Stock-BPF controls: https://github.com/LineageOS/android_kernel_xiaomi_sm8150 + https://github.com/MiCode/Xiaomi_Kernel_OpenSource (begonia-*-oss) + https://github.com/OnePlusOSS/android_kernel_oneplus_sm8150
  BPF-enabled phone kernel, stock features + defconfig: https://github.com/hrihq/Positron-Sweet2
- AOSP eBPF traffic monitoring replacing xt_qtaguid (why old phone kernels want BPF on): https://source.android.com/docs/core/connectivity/ebpf
- OpenELA 4.14 continuation: https://openela.org + https://github.com/openela/kernel-lts
- Previous attempt post-mortem (bpf-5.10 vendor-glue failures): this repo's history / build logs
