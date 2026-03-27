The following analysis is derived entirely from grep, sed, and directory inspection of the CVA6
repository. Every claim maps to a specific file and line number. 

| Extension | RVA23 | Status | Key evidence |
|-----------|-------|--------|--------------|
| Svnapot | Mandatory | Complete | 40+ hits across all MMU files. Gated on CVA6Cfg.SvnapotEn. |
| Svpbmt | Mandatory | Missing | Zero hits in MMU. pbmte stub at riscv_pkg.sv:133. Writes dropped in csr_regfile.sv:1695. |
| Svinval | Mandatory | Missing | Zero hits anywhere in repository. sinval.vma not decoded. |
| Svadu | Optional | Partial | Svade faults only (ptw.sv:487-509). No hardware writeback. ptw.sv:289 comment describes missing logic. |
| Sv48 | Mandatory | Unverified | Parametric on PtLevels but build_config_pkg.sv:29 hardcodes PtLevels=3 for XLEN64.
| Sv57 | Optional | Out of scope | Deferred — not required for RVA23 mandatory compliance. |

### Key Structural Findings
#### The tlb_update_cva6_t struct (cva6_mmu.sv:124)
The TLB update packet type is defined as a localparam type directly inside cva6_mmu.sv at line
124, not in any shared package. Adding a pbmt field requires a single change at this location, which
propagates automatically to all submodules (ptw, tlb, shared_tlb) that receive it as a parameter
type:
```
 localparam type tlb_update_cva6_t = struct packed {
 logic valid;
 ... // vpn, asid, vmid, is_page fields
 logic is_napot_64k; // Svnapot — line 126
 logic [1:0] pbmt; // Svpbmt — TO BE ADDED
};
```
#### The envcfg_rv_t struct and proven CSR pattern (riscv_pkg.sv:130-140)
The Zicbom team already proved the menvcfg wiring pattern. Fields cbze/cbcfe/cbie are live in
csr_regfile.sv. The pbmte field at bit 62 follows the identical pattern — declare d/q registers, add a
write case gated on SvpbmtEn, add to CSR read path, output to MMU:
```
 if (CVA6Cfg.SvpbmtEn) begin
 mpbmte_d = csr_wdata[62]; // mirrors: mcbcfe_d = csr_wdata[6]
end
``` 
#### The PtLevels hardcoding (build_config_pkg.sv:29)
Sv48 requires modifying the PtLevels derivation and adding a Sv48En user config flag:
```
// Current: PtLevels = (CVA6Cfg.XLEN == 64) ? 3 : 2
// Proposed: PtLevels = (CVA6Cfg.XLEN == 32) ? 2 : CVA6Cfg.Sv48En ? 4 : 3
```
#### The missing MT signal path (load_store_unit.sv:291)
The current MMU output carries only lsu_paddr_o. Svpbmt requires adding lsu_pbmt_o[1:0]
alongside it, wired through load_store_unit.sv into dcache_req_ports_o, and enforced in all three
cache backends.
