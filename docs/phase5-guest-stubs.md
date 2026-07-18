# Phase 5 – Guest Stub Expansion

**Date:** 2026-07-18  
**Policy:** Microsoft Learn semantics only. No “always success” shortcuts that can break real apps/emulators.

## What is in-guest

| Export | Notes |
| ------ | ----- |
| Prior set (GetACP/OEMCP, PID/TID, TickCount, ProcessHeap, GetLastError, FLS, CS, …) | Unchanged |
| `GetSystemDefaultLangID` / `GetUserDefaultLangID` | `0x0409` (fixed en-US guest) |
| `GetCommandLineA` / `GetCommandLineW` | Return published env buffers |
| `GetCurrentDirectoryW` | Guest cwd blob; Learn return-value rules (chars excl. NUL on success; required size incl. NUL if too small) |
| `GetSystemMetrics` / `GetSysColor` | Guest tables (same values as host) |
| `GetSysColorBrush` / `GetDesktopWindow` | Stable fake handles |
| `GetFileSize` / `SetFilePointer` | Existing guest I/O accelerator |

## Intentionally host-only

| Export | Why |
| ------ | --- |
| `VirtualProtect` | Learn: NULL `lpflOldProtect` **fails**. Host corrected accordingly. Real protect = Phase 3 RegionTable. |
| `VirtualQuery` | Must describe real regions; fake “always committed” would mislead probes. |
| `LocalAlloc` / `GlobalAlloc` | `LMEM_MOVEABLE` / lock / size-0 discard ≠ thin `HeapAlloc`. |
| `SetUnhandledExceptionFilter` | Must return previous filter for chaining. |
| Module APIs | Need loader state. |

## Layout

- `guest_stub_data_base` (`0x7000_0004_3000`): metrics\[256\], colors\[32\], cwd wide blob  
- OOL helpers: remainder of `guest_io_code` after `+0x600`

## Verification

- `cargo clippy --workspace --all-targets -- -D warnings` clean  
- `cargo test --workspace` green  
- `./scripts/run-micro-suite.sh` (JIT) green  

## Performance (qualitative)

Phase 5 reduces **host_stops** (track C), not pure-compute wall (`long_loop` unchanged).  
On guests that hammer metrics/colors/LangID/cmdline/cwd, each avoided stop removes resolve + handler + re-enter overhead; short micros stay init-dominated.
