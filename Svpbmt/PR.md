## Summary

Adds scaffolding for Svpbmt (page-based memory types), which is
mandatory in the RVA23 profile and currently completely absent
from CVA6. This PR is the first of several — it makes four
inert structural changes that have zero functional impact when
`SvpbmtEn=0` (the default for all existing configs).

Closes [#Issue3253](https://github.com/openhwgroup/cva6/issues/3253)

## Changes



| File | Change |
|------|--------|
| `include/config_pkg.sv` | Add `bit SvpbmtEn` after `SvnapotEn` in both user and built config structs |
| `include/build_config_pkg.sv` | Propagate `SvpbmtEn` into built config (mirrors `SvnapotEn` pattern) |
| `include/riscv_pkg.sv` | Split `pte_t.reserved[9:0]` → `mt[1:0]` + `reserved[7:0]` per Svpbmt spec |
| `core/cva6_mmu/cva6_mmu.sv` | Add `logic [1:0] pbmt` to `tlb_update_cva6_t` (line 124) |

## Why this structure

- `SvpbmtEn` follows the `SvnapotEn` pattern exactly — optional,
  parameter-gated, disabled by default ("parachute" rule satisfied)
- `pte_t.reserved[9:0]` currently treats PTE bits 63:62 as reserved,
  causing a page fault on any Svpbmt-typed page (`ptw.sv:430`).
  Naming them `mt[1:0]` is spec-correct and enables the fault
  relaxation in the next commit
- `tlb_update_cva6_t` is a `localparam type` defined only at
  `cva6_mmu.sv:124` and passed as `parameter type` to all three
  submodules — one addition propagates everywhere automatically,
  the same mechanism used for `is_napot_64k` (Svnapot)

## Testing

With `SvpbmtEn=0` (all existing configs): no functional change.
CI should pass unmodified. Functional testing will accompany
subsequent commits that wire MT bits through PTW → TLB → LSU →
cache backends.
