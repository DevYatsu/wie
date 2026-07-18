# Phase 4.x – Selective Code Invalidation (X-loss / SMC / VirtualProtect)

**Date:** 2026-07-18  
**Depends on:** Phase 4.0–4.3 (foundation, pins, coherency data plane, string bulk).  
**Scope:** Keep the JIT code cache, chain table, edge IC, and region pins coherent when guest protection or guest code bytes change.  
**Does not:** free Cranelift native code pages; patch finalized host opcodes; implement `FlushInstructionCache` WinAPI.

## Why this exists

Phase 4 soft-translate paths (sticky TLB, region pins, block-wide super path) and late-bound chaining assume compiled blocks stay valid. On real software:

1. **`VirtualProtect` X-loss** must stop execution of host translations for that range.
2. **Self-modifying code (SMC)** — guest stores into previously compiled bytes — must drop Ready blocks before the next entry.
3. **`VirtualFree` / decommit** must not leave Ready blocks over released pages.
4. **Edge IC** must not dispatch to a dropped guest VA after selective invalidation.

Without these rules, pins + JIT are unsafe on packers, runtime code gen, and protect flip patterns (RX → RW → patch → RX).

## Invariants

1. Slow path (`GuestMemory::{read,write}` + SPC) remains the data oracle.
2. A Ready block may run only while its guest range is still executable **and** has not been written since compile.
3. Successful store overlapping a Ready range → that range is not Ready after the next safe point.
4. Protect that removes execute → no Ready block overlaps the range.
5. Free / decommit / release → no Ready block overlaps the freed span.
6. Chain table + edge IC never dispatch to a non-Ready guest VA.
7. **No W-capable soft-translate on executable pages** (sticky W / pin W / host_span write) — forces code stores through `GuestMemory::write` so SMC is observable.

## Mechanisms

### Code-page index

```text
code_pages: HashMap<page_key, refcount>
```

- Updated on Ready insert / drop / full clear.
- Write path fast-reject: if no page in the store range is present, skip the O(cache) scan.

### `invalidate_code_range(addr, len)`

1. Fast reject via `code_pages`.
2. Drop Ready entries whose `[guest_start, guest_end)` overlaps the range.
3. Clear **chain table + edge IC + shadow** (`invalidate_chain_and_shadow`).
4. Rebuild chain table from remaining Ready entries (when chaining is on).
5. Bump `JitStats.code_invs`.

### Triggers

| Event | Action |
| ----- | ------ |
| Host `mem_write` | After write: drain pending → `invalidate_code_range` |
| Guest store (`GuestMemory::write`) | Notes coalesced pending range; drained after compiled block / iced step |
| `VirtualProtect` with `!allows_execute(new)` | `invalidate_code_range` (X-loss) + TLB flush |
| `VirtualProtect` that keeps X | TLB only (content unchanged) |
| `VirtualFree` RELEASE / DECOMMIT | `invalidate_code_range` over allocation / decommit span + TLB |
| Hook reinstall / full flush | `clear_compiled` + chain/edge clear |

### No W on X (soft-translate policy)

| Path | Behaviour |
| ---- | --------- |
| `page_tlb_entry` / sticky install | `allow_w = write && !execute` |
| Region pin | Intersection; any X page clears W |
| `host_span(..., write=true)` | `None` if any page in span allows execute |
| Stack super path | Unaffected (stack is RW, not X) |

### Pending drain (re-entrancy)

Stores during a native block only **note page keys** on `GuestMemory` (`pending_write_pages`).  
`JitCpu` applies `invalidate_code_range` per page after the block returns (and after iced fallback steps) so the running frame is never torn down mid-call.

**Do not** coalesce stores into a single VA bounding box — CRT/stack+heap traffic would span gigabytes of empty pages and hang in the overlap scan. Cap recorded pages (overflow → full code flush).

## What stays forbidden

- Patching finalized Cranelift output (chaining remains data-plane).
- Assuming host I-cache ops for *guest* code (re-decode on next compile).
- Using sticky/pin W into RWX code pages.

## Interaction with Phase 4

| Feature | Interaction |
| ------- | ----------- |
| `mem_gen` / TLB / pins | Unchanged; protect still bumps generation and clears TLB/pins |
| Block-wide super path | Stack-only; no X pages → no SMC path |
| Edge IC / chain | Cleared+rebuild on any selective drop |
| REP host_span | Write bulk refuses X pages; element path uses `write` + pending |

## Kill-switches / bisect

```bash
WIE_CPU=iced             # no JIT cache
WIE_JIT_CHAIN=0          # no chain/edge benefit (inv still correct)
WIE_JIT_MEM=slow         # helper-only mem; SMC still drained
```

## Verification

```bash
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
# unit: jit::tests::code_inv_* , host_span_write_denied_on_executable, page_tlb_entry_rw_allows_w_rx_denies_w
WIE_CPU=jit ./scripts/run-micro-suite.sh
WIE_CPU=iced ./scripts/run-micro-suite.sh
```

## Acceptance sequences

| ID | Sequence | Expect |
| -- | -------- | ------ |
| S1 | Plant Ready → `VirtualProtect` RO | Ready gone; edge IC clear |
| S2 | Plant Ready → store into range | Ready gone |
| S3 | Plant Ready → data write elsewhere | Ready kept; `code_invs` unchanged |
| S4 | Plant Ready → `VirtualFree` | Ready gone; code_pages empty |
| S5 | Plant Ready → protect RX (keep X) | Ready kept |
| S6 | RWX page → TLB `allow_w == false` | Stores miss soft-translate |

## Related

- Foundation: [`phase4-foundation.md`](phase4-foundation.md)
- Pins: [`phase4-region-pins.md`](phase4-region-pins.md)
- Coherency / edge IC: [`phase4-jit-coherency.md`](phase4-jit-coherency.md)
- Roadmap: [`Optimization ROADMAP.md`](../Optimization%20ROADMAP.md)
