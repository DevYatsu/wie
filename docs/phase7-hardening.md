# Phase 7 – Hardening & Cutover

**Date:** 2026-07-18  
**Depends on:** Phases 0–6 (esp. 4.x invalidation, 2/3 mmap).  
**Scope:** Stress residual invalidation, anti-Wine checks, optional `FlushInstructionCache`, default `WIE_MEM=mmap`, RUNBOOK.  
**Does not:** re-implement Phase 4.x; identity mapping; SIGSEGV fault epic.

> **Post-cleanup note:** the great cleanup removed `WIE_MEM` / hash / hybrid. Storage is mmap-only; CLI is `inspect` / `run` / `trace`. Historical dual-backend wording below is preserved as Phase 7 context.

## 7.1 Invalidation residual

Core rules remain in [`phase4-code-invalidation.md`](phase4-code-invalidation.md). Phase 7 adds:

| Item | Status |
| ---- | ------ |
| Multi-region protect + free under Ready | unit: `code_inv_multi_region_protect_and_free` |
| SMC across page boundary | unit: `code_inv_smc_across_page_boundary` |
| `FlushInstructionCache` | host stub + JIT Ready drop (`size==0` → full clear) |

### `FlushInstructionCache`

Microsoft Learn: after software writes to code, flush so subsequent instruction fetches see new bytes.

WIE mapping (clean room):

- Host `KERNEL32!FlushInstructionCache` (extra dispatch) → `CpuEngine::flush_instruction_cache`
- JIT: selective `invalidate_code_range`; `dwSize == 0` → clear all Ready + chain/edge IC
- Iced: success no-op (no native code cache)

Does **not** touch host I-cache for Cranelift output (chaining stays data-plane).

## 7.2 Stress / anti-Wine

| Test | Expect |
| ---- | ------ |
| `phase7_high_va_mmap_roundtrip` | High guest VA soft-translate R/W; host ≠ guest |
| `phase7_map_wraparound_rejected` | Overflow `map` errors (no panic) |
| `phase7_large_reserve_demand_zero_survives` | ≥1 GiB RESERVE ok or clean mmap error; touch one page |
| `phase7_anti_wine_soft_translate` | Host page ptr ≠ guest VA (mmap) |
| `phase7_virtual_alloc_size_overflow_rejected` | Huge size rejected |

**Checklist (manual / CI script):**

- No `mmap` at guest VA / low 4 GiB identity reservation  
- Soft translate only (`host + (guest_va - guest_base)`)  

## 7.3 Default flip → sole path

| Epoch | Storage |
| ----- | ------- |
| Pre Phase 7 | Default `WIE_MEM=hybrid`; force hash/mmap/hybrid |
| Phase 7 | Default **`mmap`**; hash/hybrid overrides for rollback |
| Great cleanup | **mmap only** — no `WIE_MEM`, no hash/hybrid code |

```bash
./scripts/run-micro-suite.sh
```

**Rationale:** pure arenas for all maps (including tiny TEB/stub pages) simplify host layout; soft translate + SPC unchanged.

**Note:** mmap improves memory-helper paths on large arenas; it does **not** reduce idle CPU (that is Phase 6 / `WIE_IDLE`).

## 7.4 RUNBOOK

See [`RUNBOOK.md`](RUNBOOK.md).

## Verification

```bash
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo build -p wie-cli --release
./scripts/run-micro-suite.sh
WIE_RUNTIME_PROFILE=1 ./target/release/wie-cli run micro-exes/out/long_loop.exe
# mem_backend=mmap, ~100% CPU on long_loop
```

## Related

- Phase 4.x inv: [`phase4-code-invalidation.md`](phase4-code-invalidation.md)  
- Phase 2 storage: [`phase2-mmap-backend.md`](phase2-mmap-backend.md)  
- Phase 6 idle: [`phase6-idle.md`](phase6-idle.md)  
- Roadmap: [`../Optimization ROADMAP.md`](../Optimization%20ROADMAP.md)  
