# RMO-NX1 adaptation notes

Target: HONOR RMO-NX1 (HONOR X9a 5G), Qualcomm SM6375, arm64.

Status: **offsets recovered, target builds in CI.** Nothing has been executed
on the device.

## Identified kernel and system

Confirmed twice — from the boot image and from the device over adb:

```
kernel          : 5.4.274-qgki-g6bced53b6b8b
model / device  : RMO-NX1 / HNRMO-Q
android         : 13 (sdk 33)
fingerprint     : HONOR/RMO-NX1/HNRMO-Q:13/HONORRMO-N21/7.1.0.337C185E7R2P4:user/release-keys
SELinux         : Enforcing (shell domain u:r:shell:s0, uid 2000)
Seccomp (shell) : 0
perf_paranoid   : -1
CPU part        : 0xd05 (Cortex-A55) x8
```

`/proc/kallsyms` and `/proc/iomem` are closed to the shell, so symbol
extraction had to come from the image.

## How the offsets were obtained

The embedded kallsyms table was recovered directly from the uncompressed
arm64 Image (boot.img kernel section, 56,844,288 bytes):

```
offsets    @ 0x01bea510   198,123 x u32, monotonic 0x0 .. 0x3636000
rel_base   @ 0x01cabcc4   0
num_syms   @ 0x01cabccc   198,123
names      @ 0x01cabcd0 .. 0x020de77f
markers    @ 0x020de780   774 x u32 (first 0, ascending deltas)
token_tbl  @ 0x020df398   256 entries
token_idx  @ 0x020df6c0
```

Integrity check: `_text` offset 0 and `_end` offset `0x3636000`, which equals
the Image header's `image_size`. 193,709 unique names decoded.

`ghostlock-extract` (YuKongA, pin `6641116`) refuses this image: it has no 5.4
profile, `CONFIG_DEBUG_INFO_BTF` is unset so its BTF tie-breaker is missing,
and its `_end - _text == image_size` assumption does not match a trimmed vendor
Image.

### Offset convention

kallsyms offsets are relative to `_text`. This file family stores offsets
relative to `KIMAGE_TEXT_BASE`, and `_text = KIMAGE_TEXT_BASE + 0x80000`
(`text_offset` from the Image header). Every value in `target.h` is therefore
`kallsyms offset + 0x80000`.

## What is measured vs. assumed

| Group | Source |
|---|---|
| `INIT_TASK_OFF`, `INIT_CRED_OFF`, `ROOT_TASK_GROUP_OFF`, `SELINUX_ENFORCING_OFF`, `SIG_ENFORCE_OFF`, `KASLR_HOT_OFF`, `SLIDE_*`, `CTL_BOOTID_DATA_OFF`, `BSS_SCRATCH_OFF` | measured from kallsyms in this image |
| `WAITER_*`, `FAKE_TASK_*`, `TASK_CRED_OFF`, `TASK_COMM_OFF`, `MM_OWNER_OFF`, `FP_STATE_OFF` | GKI ABI, identical in `so52d-5.4.295` and `sog05-5.4.210` — not measured here |

The second group relies on GKI KMI stability: `task_struct` and
`rt_mutex_waiter` layout is frozen per KMI regardless of vendor patch level.
Strong evidence, not proof.

## Known gaps

| Field | Status |
|---|---|
| `CTL_BOOTID_ENTRY_OFF` | not located; `kern_table`'s boot_id entry is not statically resolvable. Unused by impl54. |
| `LINK_BOOTID_CTL_ADDR` | same gap; supply at runtime with `LINK_BOOTID_CTL`. |
| `SLIDE_SYSCTL_BOOTID_OFF` | unused by impl54, left 0. |
| `RSCAN_SKIP_FLAG_OFF` | unused, left 0. |

## Build

`.github/workflows/build-exploit.yml` builds this target on a GitHub-hosted
runner. All compilation happens in Actions; the phone is only used for
read-only inspection.

```
gh workflow run build-exploit.yml \
  --repo rehimlianar/ghostlock_5.4_xperia \
  --ref rmo-nx1-5.4.274 \
  -f project=rmo-nx1-5.4.274 \
  -f ndk_release=r27c \
  -f api_level=35
```

Last green run: `44505063446`, `6abcc4c`.

```
exploit        80,832 B  ELF aarch64 PIE, dynamic, NDK r27c
exploit_static 2,339,136 B  ELF aarch64, statically linked
```

## Not verified

- Nothing has been run on the phone. The offsets describe the firmware; they
  are not a demonstration that the exploit works.
- `WAITER_*` / `FAKE_TASK_*` were not measured on this build.
- Slide side-channel behaviour on Cortex-A55 is unmeasured.
- `CONFIG_PANIC_ON_OOPS=y` still means a wrong write reboots the device and
  changes the KASLR slide. First execution needs explicit approval.

## Structural blockers (unchanged)

- `CONFIG_MODULE_SIG_FORCE=y` — no unsigned module can load; module-based
  persistence is unavailable. The `su_runtime` in this tree does not need one,
  it runs a root daemon in-process.
- `CONFIG_SLAB_FREELIST_HARDENED=y` / `_RANDOM=y`.
