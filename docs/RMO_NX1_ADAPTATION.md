# RMO-NX1 adaptation notes

Target: HONOR RMO-NX1 (HONOR X9a 5G), Qualcomm SM6375, arm64.

## Identified kernel

`5.4.274-qgki-g6bced53b6b8b`

Read from the device boot image (`boot.img`) shipped with the firmware set:

```
$ grep -a -o -m1 'Linux version [0-9][0-9.a-zA-Z_-]*' boot.img
Linux version 5.4.274-qgki-g6bced53b6b8b
```

Embedded kernel config (`IKCONFIG` at offset 25,698,608, gzip, 7239 lines):

```
CONFIG_LOCALVERSION="-qgki"
CONFIG_CC_VERSION_TEXT="Android (6877366 based on r383902b1) clang version 11.0.2"
CONFIG_LTO_CLANG=y
CONFIG_ARM64_4K_PAGES=y
CONFIG_RT_MUTEXES=y
CONFIG_FUTEX=y
CONFIG_FUTEX_PI=y
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_SELINUX_DEVELOP=y
CONFIG_RANDOMIZE_BASE=y
CONFIG_KALLSYMS_ALL=y
CONFIG_PANIC_ON_OOPS=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_FORCE=y
CONFIG_MODULE_SIG_KEY="honor_signing_key.pem"
# CONFIG_DEBUG_RT_MUTEXES is not set
```

## Why this target selects `TARGET_ROUTE_54`

The kernel is 5.4 QGKI, so the shared `src/impl54/` implementation
(FP carrier / pselect waiter route) applies. `TARGET_ROUTE_54` is defined in
`target.h`, which is what the Makefile uses to choose `src/impl54` over `src/`.

## Current status: skeleton only

`exploit/src/targets/rmo-nx1-5.4.274/target.h` carries an explicit guard:

```c
#ifndef RMO_NX1_OFFSETS_VERIFIED
#error "RMO-NX1 offsets are unverified placeholders. ..."
#endif
```

Every offset in that file is a placeholder copied from the closest published
5.4 target so the file shape is visible. Building with them would write to the
wrong kernel addresses on a device that panics on any oops.

## Enabling the build

1. Run the offset extraction workflow in the companion fork
   (`rehimlianar/ghostlock-app`, `.github/workflows/extract-rmo-nx1-offsets.yml`)
   against the RMO-NX1 `boot.img` and `xbl_config.elf`.
2. Replace the placeholders in `target.h` with the extracted values.
3. Re-derive `WAITER_*`, `FAKE_TASK_*`, `TASK_CRED_OFF`, `TASK_COMM_OFF` and
   `FP_STATE_OFF` from `CONFIG_DEBUG_RT_MUTEXES=n` layout — they are not
   guaranteed to match the debug-enabled targets in `sog05` / `so52d`.
4. Verify `KIMAGE_TEXT_BASE` (39-bit VA assumption) and the slide anchors.
5. Remove the guard and rebuild.

## Build pipeline

`.github/workflows/build-exploit.yml` builds any target with the Android NDK
on a GitHub-hosted runner. All compilation happens in Actions; nothing is
compiled on the device.

Dispatch it with:

```
gh workflow run build-exploit.yml \
  --repo rehimlianar/ghostlock_5.4_xperia \
  --ref rmo-nx1-5.4.274 \
  -f project=so52d-5.4.295 \
  -f ndk_release=r27c \
  -f api_level=35
```

Artifacts: `exploit`, `exploit_static`, `target.h`, `SHA256SUMS`,
`build-manifest.json`, plus the full build log (also uploaded on failure).

## Known blockers

- `CONFIG_MODULE_SIG_FORCE=y`: no unsigned module can ever load on this device.
  Any module-based persistence path (KernelSU module, APatch/KernelPatch LKM)
  is structurally unavailable. Only in-process root is reachable.
- `CONFIG_PANIC_ON_OOPS=y`: a single bad write reboots the device and changes
  the KASLR slide. Read-only stages must be validated before any write stage.
- SM6375 uses Cortex-A78/A55 cores, not A53. The published slide scan relies on
  A53 prefetch timing; the A78/A55 behaviour is unmeasured.
