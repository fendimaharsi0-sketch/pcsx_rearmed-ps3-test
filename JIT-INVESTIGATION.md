# PCSX-ReARMed PS3 — Lightrec/JIT Investigation (2026-10-07)

## Status: CLOSED — Interpreter-only

After systematic hardware testing, the Lightrec JIT is **not viable** on PS3
(PPC64) with the current GNU Lightning backend. The final build uses
interpreter-only (`DYNAREC=0`). This document records what was tried, what was
learned, and why we stopped — for anyone who wants to continue this work.

## Background

- **Target:** RetroArch PS3 (PSL1GHT/FOSS SDK) Plan A core — static SELF that
  runs in Crystal CE ecosystem (`/dev_hdd0/game/RETROARCH/`).
- **Core:** libretro/pcsx_rearmed, `platform=psl1ght` target.
- **Problem:** With Lightrec enabled (default), loading any game → black screen
  + full PS3 hang (FTP alive, XMB dead, power button double-beep force shutdown).
  Interpreter (`pcsx_rearmed_drc = "disabled"`) boots games but very slow.
- **Hardware:** CFW Cobra 8.5. PicoDrive's PS3 dynarec (ps3mapi-based) works on
  the same machine — proving the hardware supports executable JIT memory.

## What was tried (chronological)

### 1. Executable memory via ps3mapi ✅ (worked)
**Hypothesis:** `malloc()` on PS3 is not executable → JIT code faults.
**Fix:** Allocate the 8 MiB Lightrec code buffer via
`ps3mapi_process_page_allocate()` (Cobra/Mamba syscall 8, `is_executable=1`).
**Result:** Allocation succeeded (`0x60000000`). But JIT still hung.
**Ref:** https://github.com/OsirizX/ps3mapi-lib

Two sub-fixes were needed:
- `getpid()` is broken in PSL1GHT (returns -1). Used LV2 syscall 1 directly
  via `lv2syscall8`.
- `ps3mapi-lib` has **no** `ps3mapi_process_page_free` — pages are leaked
  intentionally (once per process).

### 2. Cache invalidation ❌ (did not fix)
**Hypothesis:** PPC needs explicit `dcbst`/`icbi`/`isync` after writing JIT code
(Wii U uses `LIGHTREC_CODE_INV=1`).
**Fix:** Built with `LIGHTREC_CODE_INV=1` + custom `lightrec_code_inv()` in
`plugin.c` using 128-byte cache lines.
**Result:** Built fine, still hung. Cache coherency alone is not the fix.

### 3. Function descriptor / TOC in `_callr` ❌ (did not fix)
**Hypothesis:** PS3 PPC64 uses function descriptors `[code_addr, TOC]`.
GNU Lightning's `_callr()` only handled descriptors for AIX (`_CALL_AIXDESC`),
not PS3. Indirect calls jumped to descriptor data instead of code.
**Fix:** Ported irixxxx/picodrive#82 pattern to `jit_ppc-cpu.c::_callr()` —
load TOC from `[r0+8]` → r2, code address from `[r0]` → r0, save/restore r2.
**Result:** Patch applied cleanly, still hung.
**Ref:** https://github.com/irixxxx/picodrive/pull/82

### 4. Heartbeat log ✅ (breakthrough diagnostic)
**Change:** Added unbuffered logging to `lightrec_plugin_execute_internal()`
writing `ps3jit: execute start, pc=%08x` to `/dev_hdd0/game/RETROARCH/USRDIR/ps3jit.log`.
**Result:** **The JIT was running!** PC advanced through BIOS:
`bfc00000` → `bfc0002c` → ... → `bfc003b8`, then looped hundreds of times at
`bfc003b8`. The interpreter passed this point (game booted).
**Conclusion:** Not a "JIT hang" — a BIOS polling loop that never exits under JIT.
This redefined the problem from "JIT is broken" to "JIT misbehaves at one point."

### 5. `_calli` force-via-`callr` ❌ (did not fix)
**Hypothesis:** Lightrec calls its C wrapper via `jit_calli()` (emitter.c:1219).
`_calli()` has a direct-`BL` fast path that would jump to the descriptor
instead of code. Forced PS3 to always use `movi`+`callr` (already patched).
**Result:** All three Lightning patches verified in build log, still looped at
`bfc003b8`.

### 6. `hw_read` call diagnostic ✅ (decisive)
**Change:** Instrumented `hw_read_word()` in `plugin.c` to log call count.
**Result:** **3,484 block executions, 0 `hw_read` calls.**
**Conclusion:** The JIT never calls back into C for hardware access. The
JIT→C transition mechanism in Lightning's PPC64 backend is fundamentally broken.
This is not fixable with targeted patches — it requires rewriting the backend.

## Decision

**Stop JIT work. Ship interpreter-only (`DYNAREC=0`).**

Rationale:
- Three targeted fixes failed.
- Diagnostic proves the problem is architectural (JIT→C calls never happen),
  not a single miscompiled instruction.
- GNU Lightning upstream officially supports **PPC 32-bit only**
  (Wikipedia). The PPC64 backend in pcsx_rearmed's bundled copy is
  experimental and untested — we are likely the first to try it on PS3.
- Fixing it means rewriting Lightning's PPC64 call/ABI handling: weeks of work
  with no guarantee.

## What works

- Interpreter boots and runs games (slow, but stable).
- Executable memory allocation via ps3mapi is proven (useful for any future
  PS3 JIT project).
- The heartbeat diagnostic technique (unbuffered `ps3jit.log`) is reusable.

## References

- ps3mapi-lib: https://github.com/OsirizX/ps3mapi-lib
- Mamba/PS3MAPI payload: https://github.com/OsirizX/Mamba
- PicoDrive PS3 dynarec PR (TOC fix): https://github.com/irixxxx/picodrive/pull/82
- GNU Lightning: https://www.gnu.org/software/lightning/
- pcsx_rearmed: https://github.com/libretro/pcsx_rearmed

## For future work

If someone wants to revive the JIT:
1. The `ps3mapi` allocation code is proven — reuse it.
2. Start from the heartbeat diagnostic to verify JIT execution.
3. Focus on Lightning's PPC64 `jit_finishr` / call sequence — the `hw_read`
   diagnostic proves C callbacks never fire.
4. Consider writing a minimal PPC64 JIT test (not full lightrec) to isolate
   the ABI issue.
5. Alternatively, port a hand-written PPC64 dynarec (like PicoDrive's approach)
   instead of fixing Lightning.

---
*Investigation by Fendi (hardware testing) and Muse (analysis/patches).*
*2026-10-07.*
