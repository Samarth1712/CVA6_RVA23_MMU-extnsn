## [#Issue3253](https://github.com/openhwgroup/cva6/issues/3253) rtl: Svpbmt support missing — pbmte unwired in menvcfg, MT bits absent from PTW/TLB/cache path
### Summary

Svpbmt (page-based memory types) is mandatory in the RVA23 profile but Svpbmt support appears to be currently unimplemented in CVA6. This issue tracks the full implementation. A source-level audit of the codebase identified the exact gaps listed below.

### Current state

- `riscv_pkg.sv:133` — `pbmte` is declared in `envcfg_rv_t` with comment *"not implemented — requires Svpbmt extension"*
- `csr_regfile.sv:1695` — `CSR_MENVCFG` write block exists but has **no `pbmte` case**; writes to bit 62 are silently dropped
- `cva6_mmu/cva6_ptw.sv:430` — the reserved-bit fault check `(|pte.reserved && CVA6Cfg.XLEN == 64)` treats MT bits (PTE[63:62]) as illegal, causing a page fault on any Svpbmt-typed page
- `riscv_pkg.sv:305–318` — `pte_t.reserved[9:0]` subsumes the MT field; bits 63:62 need to be split out as `logic [1:0] mt`
- `cva6_mmu/cva6_mmu.sv:124` — `tlb_update_cva6_t` has no `pbmt` field; MT bits cannot propagate from PTW through TLB to LSU
- `load_store_unit.sv:291` — MMU output carries only `lsu_paddr_o`; no `lsu_pbmt_o[1:0]` signal exists to carry MT to cache backends
- `cache_subsystem/wt_dcache.sv`, `std_nbdcache.sv`, `cva6_hpdcache_wrapper.sv` — none enforce MT=01 (non-cacheable) or MT=10 (I/O) bypass behaviour

No `SvpbmtEn` config flag exists anywhere in the repo (`grep -rn "SvpbmtEn" .` returns nothing).

### Required changes (file : line)

| File | Location | Change |
|------|----------|--------|
| `include/riscv_pkg.sv` | 305–318 | Split `pte_t.reserved[9:0]` → `mt[1:0]` + `reserved[7:0]` |
| `include/config_pkg.sv` | ~351 | Add `bit SvpbmtEn` after `SvnapotEn` (both user and built config structs) |
| `include/build_config_pkg.sv` | 29 | Add `cfg.SvpbmtEn = CVA6Cfg.SvpbmtEn` propagation |
| `cva6_mmu/cva6_mmu.sv` | 124 | Add `logic [1:0] pbmt` to `tlb_update_cva6_t` |
| `cva6_mmu/cva6_ptw.sv` | 430, 206 | Relax reserved-bit fault for MT field when `SvpbmtEn`; assign `update_o.pbmt` from `pte.mt` |
| `cva6_mmu/cva6_tlb.sv` | ~395 | Store `pbmt` in TLB tag; output on hit |
| `cva6_mmu/cva6_shared_tlb.sv` | 297, 308 | Propagate `pbmt` in `itlb_update_o` / `dtlb_update_o` |
| `cva6_mmu/cva6_mmu.sv` | output ports | Add `lsu_pbmt_o[1:0]` alongside `lsu_paddr_o` |
| `csr_regfile.sv` | 132, 1695 | `mpbmte_d/q`, `spbmte_d/q` registers; MENVCFG/SENVCFG write case gated on `SvpbmtEn` |
| `load_store_unit.sv` | ~291 | Wire `lsu_pbmt_o` into `dcache_req_ports_o` |
| `cache_subsystem/wt_dcache.sv` | cache ctrl | MT=01 NC bypass; MT=10 IO bypass |
| `cache_subsystem/std_nbdcache.sv` | cache ctrl | Same MT enforcement |
| `cache_subsystem/cva6_hpdcache_wrapper.sv` | cache ctrl | Same MT enforcement (validate independently — HPDcache has MSHR) |

### Implementation notes

The Zicbom implementation already proves the CSR wiring pattern. Fields `cbcfe`/`cbie` are live in `csr_regfile.sv` (registers at lines 321–322, write at line 1700, read at line 636). `pbmte` at bit 62 follows the identical pattern.

`tlb_update_cva6_t` is a `localparam type` defined only at `cva6_mmu.sv:124` and passed as a `parameter type` to `cva6_ptw`, `cva6_tlb`, and `cva6_shared_tlb`. Adding `logic [1:0] pbmt` there is a single change that propagates to all three submodules automatically — the same mechanism used for `is_napot_64k` (Svnapot).

All existing configs will default `SvpbmtEn=0` — no functional change to current behaviour.

### Related

- RISC-V Privileged Spec v1.12 — Svpbmt extension
- RVA23 profile — Svpbmt mandatory
- Svnapot (`SvnapotEn`) — existing implementation is the direct structural template
