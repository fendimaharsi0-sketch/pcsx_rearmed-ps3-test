# pcsx_rearmed PS3 Test

Build & test repo for **PCSX-ReARMed on PlayStation 3** (RetroArch).

## What is this?

A GitHub Actions workflow that builds PCSX-ReARMed as a PS3 SELF
(`pcsx_rearmed_libretro_ps3.SELF`) for use with RetroArch PSX Crystal CE.
Each SELF = RetroArch frontend + one libretro core, statically linked.

**Status:** Interpreter-only. See [JIT-INVESTIGATION.md](JIT-INVESTIGATION.md)
for the full Lightrec/JIT debugging story (spoiler: JIT doesn't work on PS3
PPC64 with current GNU Lightning — interpreter is the stable path).

## Quick start

1. Go to **Actions** → **Build pcsx_rearmed PS3 SELF** → **Run workflow**.
2. Download the `pcsx_rearmed_libretro_ps3` artifact.
3. FTP `pcsx_rearmed_libretro_ps3.SELF` to
   `/dev_hdd0/game/RETROARCH/USRDIR/cores/` on your PS3.
4. Copy `assets/rgui/` to `/dev_hdd0/game/RETROARCH/USRDIR/assets/rgui/`
   and set `menu_driver = "rgui"` in `retroarch.cfg`.
5. Put a real PS1 BIOS (`scph5500.bin`, `scph5501.bin`, or `scph5502.bin`)
   in `/dev_hdd0/game/RETROARCH/USRDIR/system/`.
6. Launch from the CE menu like any other core.

## Workflows

| File | Purpose |
|------|---------|
| `.github/workflows/build-pcsx-rearmed-ps3.yml` | Main build (interpreter). Paste manually — GitHub App lacks Workflows permission. |
| `build-pcsx-rearmed-ps3-final.yml` | Clean final workflow (local draft, same as above). |

## Docs

- [JIT-INVESTIGATION.md](JIT-INVESTIGATION.md) — Full Lightrec/JIT investigation
  log: ps3mapi executable memory, cache invalidation, PPC64 TOC/descriptor
  patches, heartbeat diagnostics, and why we stopped at interpreter.

## Requirements

- PS3 with CFW (any, for running homebrew SELF via CE).
- Crystal CE (RetroArch PSX Community Edition) installed.
- Real PS1 BIOS (Lightrec/HLE not applicable — interpreter needs real BIOS too
  for best compatibility).

## Credits

- Fendi — hardware testing, ps3mapi breakthrough, all the real work.
- OsirizX — ps3mapi-lib / Mamba payload.
- irixxxx — PicoDrive PR #82 (PS3 TOC/descriptor fix reference).
