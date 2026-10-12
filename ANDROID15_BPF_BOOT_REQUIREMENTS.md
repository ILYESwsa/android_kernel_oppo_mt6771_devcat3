# Android 15+ BPF Boot Requirements — the source-verified answer

Branch: `bpf-backport` (on top of `clean` @ 35c368b = arraymap fix)
Date: 2026-10 (research session "task 17-B", question: *"a bpf that android 15+ needs to boot"*)

---

## 0. TL;DR

**The kernel-side BPF requirement to boot an Android 15 ROM (LineageOS 22.1 class,
or phh-type GSI) on this MT6771 4.14.336 tree is: stock 4.14 eBPF, enabled in
defconfig. Nothing newer.** That is exactly what this branch already carries:

| Requirement | Where it lives | Status |
|---|---|---|
| eBPF on: `BPF_SYSCALL`, `BPF_JIT`, `BPF_JIT_ALWAYS_ON`, `CGROUP_BPF`, `NET_CLS_BPF`, `NET_ACT_BPF`, `STREAM_PARSER` | Phase 0a commit on this branch | **done** |
| tracing progs (kprobe/perf/tracepoint) loadable | Phase 0b commit on this branch | **done** |
| the OPPO arraymap update bug fixed (netd traffic stats need non-zero-index array updates) | `clean` @ 35c368b | **done** |
| sockaddr hooks (connect4/6, udp sendmsg/recvmsg), bind4/6, sockopt, sock release, ringbuf, BTF, global data, map freeze | — | **NOT required to boot** (A15 skips them on 4.14 by design; see §4) |

The remaining blocker for Android 15 on 4.14 is **ROM-side**, not kernel-side:
AOSP's own tethering loader hard-fails on kernels < 4.19, and LineageOS 22.1
downgraded those checks to warnings (§5). That patching happens in the ROM's
Connectivity module — it is not something this kernel repo can or should carry.

Everything below is measured from the actual loader and BPF-program sources of
Android 15 and LineageOS 22.1 — not from blogs or search snippets.

## 1. Method

Read and grepped the primary sources directly:

- `platform/system/bpf` @ `android-15.0.0_r1` (platform bpfloader + libbpf_android)
- `platform/packages/modules/Connectivity` @ `android-15.0.0_r1`
  (netbpfload — the A15 networking BPF loader; bpf_progs/netd.c, offload.c,
  clatd.c, dscpPolicy.c, block.c, gentle.c — the actual .o files loaded at boot;
  netd/BpfHandler.cpp — runtime cgroup attach logic)
- `LineageOS/android_packages_modules_Connectivity` @ `lineage-22.1`
  (the Connectivity that ships on real Android-15 4.14 phones)
- `LineageOS/android_kernel_oneplus_sm8150` @ `lineage-22.1` (OnePlus 7 kernel —
  the official-LOS-22.1-on-4.14 control specimen)
- `kernel/common` `deprecated/android-4.14*` (Google's final 4.14 ACK — what
  Google itself considered a valid 4.14 Android kernel)

## 2. How Android 15 loads BPF at boot

```
init
 └─ /system/bin/bpfloader            (platform, system/bpf)
 │    loads /system/etc/bpf/*.o + /vendor/etc/bpf/*.o
 │    A15 allowed prog types: KPROBE, PERF_EVENT, SOCKET_FILTER, TRACEPOINT
 │    (networking prog types are NO LONGER loaded here at all)
 │    critical .o failure  → "DO NOT EXPECT SYSTEM TO BOOT SUCCESSFULLY", exit 120
 └─ execve /apex/com.android.tethering/bin/netbpfload
      (Connectivity mainline module, netbpfload/NetBpfLoad.cpp + loader.cpp)
      loads ALL networking BPF:
        netd_shared@mainline.o   (netd.c)        ← CRITICAL "Connectivity and netd"
        netd_offload@mainline.o / offload.o     (tethering offload)
        clatd.o                  (clatd.c)       (CLAT / nat64)
        dscpPolicy.o, block.o, gentle.o, ...
      kernel-version floors enforced here (§5)
 └─ execve .../bin/netbpfnative     (sysctl tuning, warnings only)
 └─ netd (BpfHandler.cpp) attaches loaded progs to cgroup2 at runtime
```

Key loader behaviors (from `Loader.cpp` / `netbpfload/loader.cpp`, A15):

- Maps and programs carry `min_kver`/`max_kver` in their ELF definitions;
  anything outside the running kernel's range is **skipped, not failed**
  (`skipping map/program ... which requires kernel version ...`).
- Program load failure is fatal only if the program is not marked `optional`;
  netd_shared.o is marked `CRITICAL("Connectivity and netd")`.
- Map names in `BPF_MAP_CREATE` are only passed on 4.15+ (graceful on 4.14).
- `BPF_OBJ_GET_INFO_BY_FD` map verification requires 4.14+ (we are 4.14 ✓).
- Map freezing (`BPF_MAP_FREEZE`) is **gone from the A15 loader entirely** —
  no freeze call exists in netbpfload; nothing to backport.
- BTF sections in the .o files are disabled on user builds
  (`DISABLE_BTF_ON_USER_BUILDS()` in netd.c et al.) — kernel BTF not needed.

## 3. What actually loads on a 4.14 kernel — the critical set (all stock)

Every item below is marked mandatory/critical in A15's netd_shared.o /
tethering .o sources, gated at kernel ≥ 4.14 or lower, and ALL of them resolve
to features already present in stock 4.14 (= this tree, once defconfig is on):

| Android 15 boot item | Program/map (source) | Kernel need | Stock 4.14 |
|---|---|---|---|
| Per-UID traffic accounting | `cgroupskb/ingress/stats$4_14`, `.../egress/stats$4_14` (netd.c keeps an explicit 4.14–4.18 variant) | `BPF_PROG_TYPE_CGROUP_SKB` + `BPF_CGROUP_INET_INGRESS/EGRESS` attach (4.10+) | ✓ |
| Socket creation tagging | `cgroupsock/inet_create` | `BPF_CGROUP_INET_SOCK_CREATE` (4.10+) | ✓ |
| netd stats/tag maps | `configuration_map`, `cookie_tag_map`, `stats_map_A/B`, `app_uid_stats_map`, `iface_stats_map`, ... | `BPF_MAP_TYPE_HASH` + `ARRAY` (+ percpu available) | ✓ |
| IPv6 tethering offload | `schedcls/tether_{down,up}stream6_rawip$4_14` (offload.c) | `SCHED_CLS` + `bpf_redirect()` helper (4.4+) | ✓ |
| CLAT / 464XLAT | `schedcls/ingress6/clat_{ether,rawip}$4_14` (+ `$4_9` fallbacks) (clatd.c) | `SCHED_CLS` + `bpf_redirect()` | ✓ |
| xt_bpf / iptables coexistance | socket-filter progs pinned for reuse | `SOCKET_FILTER` | ✓ |
| Map pinning/verification | bpffs pins + fd info checks | `BPF_FS` (comes with `BPF_SYSCALL`), `BPF_OBJ_GET_INFO_BY_FD` (4.14) | ✓ |
| cgroup attach at runtime | BpfHandler attaches to cgroup2 | `CONFIG_CGROUPS` + `CGROUP_BPF` (cgroup2 mount, 4.5+) | ✓ |

The **only** vendor-tree-specific bug in this critical path was OPPO's arraymap
index check (non-zero-index array updates returned `-EINVAL`), which breaks
netd's stats/configuration array updates — fixed on `clean` @ 35c368b.
The 4.14–4.18 stats variant intentionally uses `bpf_skb_load_bytes()` (absolute
load): AOSP's own comment notes wifi/L2-header traffic parsing is imperfect on
< 4.19 ("basically a lost cause" for ethernet-offset parsing), but stats still
account — this is a known and accepted AOSP degradation, not a boot problem.

## 4. What Android 15 references but SKIPS on 4.14 — by design, gracefully

| Feature | Mainline kernel | How AOSP A15 marks it | How LOS 22.1 marks it | Effect on 4.14 |
|---|---|---|---|---|
| connect4/connect6 hooks | 4.17 | `KVER_4_14` + MANDATORY (netd.c) — loads only if kernel actually has `BPF_PROG_TYPE_CGROUP_SOCK_ADDR` | **`KVER_4_19`** → skipped on 4.14 | none on LOS; would matter only for unpatched-AOSP connectivity (§5, §8) |
| udp4/6 sendmsg + recvmsg hooks | 4.18 | same as above (`KVER_4_14`, mandatory, `BPF_PROG_TYPE_CGROUP_SOCK_ADDR`) | **`KVER_4_19`** → skipped | none on LOS |
| bind4/bind6 (blocked ports) | 4.16 | block.c `KVER_4_19` | `KVER_4_19` | skipped |
| getsockopt/setsockopt progs | 5.3/5.4 | netd.c `KVER_5_4` | `KVER_5_4` | skipped |
| inet_release + `bpf_get_sk_cookie` | 5.10 | netd.c `KVER_5_10` | `KVER_5_10` | skipped |
| packet-trace ringbuf (`packet_trace_ringbuf`) | 5.8 | netd.c `KVER_5_8` + `BPFLOADER_U_VERSION` map floor | same | skipped (debug-only feature) |
| IPv4 tethering offload | — | offload.c keeps only `$5_8` variant in A15 | LOS 22.1 **restored `$4_14`..`$5_8` variants** | skipped on AOSP apex; **works on LOS 22.1** |
| dscpPolicy | 5.15 | `schedcls/set_dscp_ether` `KVER_5_15` | same | skipped (whole module inert) |
| sockops/`skfilter/accept` (gentle.o) | 4.14- | plain | plain | loads fine |
| BTF in .o files | 5.1+ | `DISABLE_BTF_ON_USER_BUILDS()` | same | not loaded at all |
| global data / `.rodata` / `BPF_F_MMAPABLE` | 5.2/5.5 | not used by android-loader-format .o files | same | not used |
| map freeze (`BPF_MAP_FREEZE`) | 5.2 | **removed from A15 loader** | same | not called |
| `BPF_F_RDONLY_PROG` map flags | 4.18 | not set by any boot .o map (only DEVMAP comment in loader) | same | not used |

Conclusion: **no post-4.14 kernel feature is a boot requirement.** The loader's
min_kver machinery skips exactly the things stock 4.14 lacks.

## 5. The version floors — where the *real* Android-15/4.14 wall lives

`netbpfload/NetBpfLoad.cpp`:

| Check | AOSP android-15.0.0_r1 | LineageOS lineage-22.1 |
|---|---|---|
| T requires 4.9 | `return 1` (fatal) | `ALOGW` (warning) |
| U requires 4.14 | `return 1` (fatal) | `ALOGW` (warning) |
| **V requires 4.19** | **`return 1` (fatal)** | **`ALOGW` (warning)** |
| 32-bit-kernel ≥ 5.16 | fatal (not TV) | fatal (not TV) — irrelevant on arm64 |
| LTS sublevel floors (`4.19.236`, `5.4.186`, ...) | warning only (`bad` flag, no return) | warning only |
| `unprivileged_bpf_disabled` write | fatal only on 5.13+; pre-5.13 defaults to 0 | same — 4.14 ignores |

So: **stock AOSP A15 refuses to boot on ANY 4.14 kernel** — this is a
userspace policy decision in the Connectivity module, not a missing kernel
feature. LineageOS 22.1 (the Android 15 that actually ships on 4.14 phones)
relaxed exactly these three lines. Any serious A15-on-4.14 effort (GSI or
custom ROM) carries the same class of ROM-side patch; the kernel cannot
satisfy it by any amount of BPF backporting (short of lying about
`KERNEL_VERSION`, which would be a betrayal of the module ABI).

Note the floor is not arbitrary: AOSP's sockaddr progs are marked `KVER_4_14`
but **cannot** load on a stock 4.14 kernel (prog type 4.17+, udp hooks 4.18+).
On LOS 22.1 the same progs are re-marked `KVER_4_19`, i.e. LOS aligned the .o
markers with what 4.14-class kernels can really do. phh-type GSIs boot A15 on
4.14 devices through equivalent compat patches in their system side; that is
ROM work too.

## 6. Empirical proof — the 4.14 phone that officially runs Android 15

`LineageOS/android_kernel_oneplus_sm8150` @ `lineage-22.1` (OnePlus 7 / 7 Pro,
official LineageOS 22.1 = Android 15):

- `include/uapi/linux/bpf.h` measured: **stock 4.14 BPF ceiling** — 16 map
  types, 14 prog types, **zero** sockaddr hooks / `BPF_MAP_FREEZE` /
  `BPF_F_RDONLY_PROG` / `BPF_MAP_TYPE_RINGBUF`.
- Same BPF feature level as this MT6771 tree after Phase 0.
- Boots official Android 15, networking fully functional (traffic stats,
  tethering, CLAT, VPN) with the skip-list of §4.

This kills the "4.14 needs a modern BPF backport to run Android 15" hypothesis
outright. What the 4.14 phone world actually does: enable stock eBPF in
defconfig (our Phase 0 — also what hrihq/Positron-Sweet2 does on Redmi Note 12
Pro 4G), let the ROM's loader skip the post-4.14 features.

## 7. Verdict for this tree

1. `clean` @ 35c368b (arraymap fix) — required; without it netd's stat array
   updates return `-EINVAL`.
2. `bpf-backport` Phase 0a + 0b (this branch) — **the complete kernel-side BPF
   requirement for Android 15.** Stop here for boot purposes.
3. Nothing else in `kernel/bpf/`, `net/core/filter.c` or uapi needs to change
   to boot Android 15/LOS-22.1-class ROMs on this device.
4. Remaining Android-15-on-F11 work is outside this repo's scope: ROM-side
   Connectivity patches (already standard in LOS 22.1), vendor sepolicy for
   `/sys/fs/bpf`, and the usual Treble/vendor-blobs issues.
5. Build & flash validation of Phase 0 is still pending (chunked clang build,
   vermagic must stay `4.14.336`), per BPF_BACKPORT.md §7.

## 8. Optional Phase 1A (future-proofing only — NOT needed to boot)

If one ever wants *unpatched-AOSP-style* connectivity to work on this kernel
(e.g. running raw AOSP's netd_shared@mainline.o with only the 4.19 floor
patched), the minimal kernel delta is the cgroup sockaddr hook set:

1. `BPF_PROG_TYPE_CGROUP_SOCK_ADDR` + connect4/connect6 attach (upstream 4.17
   series: uapi `bpf.h`, `net/core/filter.c`, `kernel/bpf/cgroup.c`)
2. udp4/udp6 sendmsg + recvmsg attach types (upstream 4.18 series)
3. (bind4/bind6, 4.16/4.17 series — optional)

~all self-contained in the BPF/networking layer, no vendor glue — matches the
Phase 1 plan in BPF_BACKPORT.md §5. But be honest about the payoff: LOS 22.1
gates these progs at `KVER_4_19`, so on real-world 4.14 ROMs they stay skipped
even if the kernel gains them. Do it for fun/education or AOSP-purist builds,
not "because Android 15 needs it".

## 9. Explicitly NOT needed to boot Android 15 (with evidence)

- BTF (kernel-side) — `DISABLE_BTF_ON_USER_BUILDS()`; loader tolerates absence
- `BPF_MAP_TYPE_RINGBUF` — only the 5.8-gated packet-trace debug feature
- global data / `.rodata` / `BPF_F_MMAPABLE` maps — android .o format doesn't use them
- map freeze / `BPF_F_RDONLY_PROG` flags — not set, not frozen, in A15
- LRU maps / CGROUP_STORAGE / SOCKHASH / QUEUE / STACK maps — not used by A15 boot .o
- sockaddr/bind/sockopt/release hooks — skipped on 4.14 (§4)
- bpf_iter, freplace, fentry/fexit, atomics, kfuncs — never referenced

## 10. Sources (primary, read directly)

- system/bpf @ android-15.0.0_r1: `loader/BpfLoader.cpp` (platform prog-type
  allowlist, exit 120), `loader/Loader.cpp` (min_kver skip machinery, 4.15
  map-name gate, DEVMAP downgrades)
- Connectivity @ android-15.0.0_r1: `netbpfload/NetBpfLoad.cpp` (version
  floors, LTS sublevel warnings, unpriv sysctl), `netbpfload/loader.cpp`,
  `bpf_progs/netd.c` (`$4_14` variants, sockaddr KVER markers, ringbuf 5.8,
  `CRITICAL("Connectivity and netd")`), `bpf_progs/offload.c`, `bpf_progs/clatd.c`,
  `bpf_progs/dscpPolicy.c`, `bpf_progs/block.c`, `netd/BpfHandler.cpp`
  (runtime attach gates)
- LineageOS Connectivity @ lineage-22.1: `bpf/loader/NetBpfLoad.cpp` (floors
  downgraded to `ALOGW`), `bpf/progs/netd.c` (sockaddr re-marked `KVER_4_19`),
  `bpf/progs/offload@mainline.c` (`$4_14` IPv4 tethering restored),
  `bpf/netd/BpfHandler.cpp` (attach gated 4.19)
- LineageOS OnePlus 7 kernel @ lineage-22.1: `include/uapi/linux/bpf.h`
  (stock 4.14 ceiling, measured)
- kernel/common `deprecated/android-4.14-q-release`: `include/uapi/linux/bpf.h`
  (Google's final 4.14 ACK has no sockaddr/freeze/RDONLY_PROG backports —
  Google never considered them part of a valid 4.14 Android kernel)
