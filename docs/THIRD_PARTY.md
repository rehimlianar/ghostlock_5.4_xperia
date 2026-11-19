# Third-party components and attribution

This fork adds RMO-NX1 (HONOR X9a 5G, SM6375) targeting work on top of the
upstream GhostLock 5.4 exploit.

## Sources

### Risuntsy/ghostlock_5.4_xperia (parent repository)

- License: Apache-2.0 (`LICENSE`)
- Used as: the base exploit implementation (`exploit/src/impl54`, `kernelsnitch`,
  `targets/*`). Unmodified except for the added target and workflow files.

### yakidango-official/GhostLock-H80GT (upstream of the parent)

- Reached transitively through the parent fork's history.

### YuKongA/ghostlock-app — `tools/extract_rs`

- License: Apache-2.0
- Used as: kernel offset extractor, built from a pinned upstream commit by
  `.github/workflows/build-extractor.yml`. Source is fetched at build time and
  is not vendored into this repository.

### WJNKAC/CVE-2026-43499-OPPO-A5-PBBT30

- License: **none declared**
- Used as: **documentation reference only** (CVE mechanism notes, dead-end
  inventory). No code is copied or vendored, because the repository carries no
  license granting redistribution or derivative rights.

## Notes

- No firmware images are stored in this repository.
- The RMO-NX1 target file contains placeholder offsets that are explicitly
  gated behind `RMO_NX1_OFFSETS_VERIFIED`; they are placeholders, not derived
  values.
