# CVA6 RVA23 Audit — Grep Outputs
# Repository: cva6/core/ (master branch, March 2026)
# Purpose: Evidence base for GSoC 2026 gap analysis
# All commands run from: ~/cva6/core/

---

## 1. Svnapot — naturally aligned huge pages

**Command:**
```bash
grep -in "napot\|svnapot\|contiguous\|NAPOT" cva6_mmu/cva6_mmu.sv cva6_mmu/cva6_ptw.sv cva6_mmu/cva6_tlb.sv cva6_mmu/cva6_shared_tlb.sv
```

**Result (selected key lines):**
```
cva6_mmu/cva6_mmu.sv:126:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_mmu/cva6_ptw.sv:101:  //Field for SVNAPOT
cva6_mmu/cva6_ptw.sv:102:  logic is_napot_64k;
cva6_mmu/cva6_ptw.sv:206:    shared_tlb_update_o.is_napot_64k = is_napot_64k;
cva6_mmu/cva6_ptw.sv:309:    is_napot_64k            = 1'b0;
cva6_mmu/cva6_ptw.sv:430:          if (!pte.v || (!pte.r && pte.w) || (|pte.reserved && CVA6Cfg.XLEN == 64) || (!CVA6Cfg.SvnapotEn && pte.n) || (CVA6Cfg.SvnapotEn && !(pte.r || pte.x) && pte.n))
cva6_mmu/cva6_ptw.sv:440:              if (CVA6Cfg.SvnapotEn && pte.n) begin
cva6_mmu/cva6_ptw.sv:442:                is_napot_64k = (pte.ppn[3:0] == 4'b1000) && (ptw_lvl_q[0] == 2);
cva6_mmu/cva6_tlb.sv:72:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_mmu/cva6_tlb.sv:98:  logic [TLB_ENTRIES-1:0] napot_tag_match;
cva6_mmu/cva6_tlb.sv:168:      assign napot_tag_match[i_gen] = (CVA6Cfg.SvnapotEn && tags_q[i_gen].is_napot_64k) ? ...
cva6_mmu/cva6_tlb.sv:259:          if (tags_q[i].is_napot_64k && CVA6Cfg.SvnapotEn) begin
cva6_mmu/cva6_tlb.sv:381:        if (update_i.is_napot_64k && CVA6Cfg.SvnapotEn) begin
cva6_mmu/cva6_tlb.sv:395:          CVA6Cfg.SvnapotEn ? update_i.is_napot_64k : 1'b0
cva6_mmu/cva6_shared_tlb.sv:92:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_mmu/cva6_shared_tlb.sv:297:          itlb_update_o.is_napot_64k = shared_tlb_update_i.is_napot_64k;
cva6_mmu/cva6_shared_tlb.sv:308:          dtlb_update_o.is_napot_64k = shared_tlb_update_i.is_napot_64k;
cva6_mmu/cva6_shared_tlb.sv:443:  assign shared_tag_wr.is_napot_64k = shared_tlb_update_i.is_napot_64k;
```

**Verdict:** COMPLETE — 40+ hits. N-bit decode, TLB hit logic, VPN masking, PPN patching, flush logic all present. Gated on `CVA6Cfg.SvnapotEn`.

---

## 2. Svpbmt — page-based memory types

**Command:**
```bash
grep -in "pbmt\|svpbmt\|memory.type\|MT_bits\|menvcfg\|senvcfg" cva6_mmu/cva6_mmu.sv cva6_mmu/cva6_ptw.sv cva6_mmu/cva6_tlb.sv cva6_mmu/cva6_shared_tlb.sv
```

**Result:**
```
(no output)
```

**Verdict:** COMPLETELY MISSING from MMU — zero hits in all four files.

---

## 3. Svinval — fine-grained TLB invalidation

**Command:**
```bash
grep -in "svinval\|sinval\|sfence.w\|hinval\|hfence" cva6_mmu/cva6_mmu.sv cva6_mmu/cva6_ptw.sv cva6_mmu/cva6_tlb.sv cva6_mmu/cva6_shared_tlb.sv
```

**Result (only sfence/hfence comments, no sinval):**
```
cva6_mmu/cva6_tlb.sv:350:          // flush everything if current VMID matches and ASID is 0 and vaddr is 0 ("SFENCE.VMA/HFENCE.VVMA x0 x0" case)
cva6_mmu/cva6_tlb.sv:353:          // flush vaddr in all addressing space if current VMID matches ("SFENCE.VMA/HFENCE.VVMA vaddr x0" case)
cva6_mmu/cva6_tlb.sv:356:          // ("SFENCE.VMA/HFENCE.VVMA vaddr asid" case)
cva6_mmu/cva6_tlb.sv:359:          // ("SFENCE.VMA/HFENCE.VVMA 0 asid" case)
cva6_mmu/cva6_tlb.sv:366:          // ("HFENCE.GVMA x0 x0" case)
cva6_mmu/cva6_tlb.sv:368:          // ("HFENCE.GVMA gpaddr x0" case)
```

**Full repo search:**
```bash
grep -rn "sinval\|SINVAL" ../
```
**Result:** `(no output)`

**Verdict:** COMPLETELY MISSING — `sinval.vma`, `sfence.w.inval`, `sfence.inval.ir` not decoded anywhere in the repo.

---

## 4. Svadu / Svade — accessed/dirty bit management

**Command:**
```bash
grep -in "svade\|svadu\|accessed\|dirty\|pte\.a\|pte\.d\|hw_update\|page.fault" cva6_mmu/cva6_ptw.sv cva6_mmu/cva6_mmu.sv
```

**Key result lines:**
```
cva6_mmu/cva6_ptw.sv:289:  //    Set pte.a to 1, and, if the memory access is a store, set pte.d to 1.
cva6_mmu/cva6_ptw.sv:487:                if ((!pte.x && (!CVA6Cfg.RVH || ptw_stage_q != G_INTERMED_STAGE)) || !pte.a
cva6_mmu/cva6_ptw.sv:504:                  (pte.a && ((pte.r && !hlvx_inst_i) || ...))
cva6_mmu/cva6_ptw.sv:509:                    && (!lsu_is_store_i || (pte.w && pte.d) || ...)
```

**Verdict:** PARTIAL (Svade only) — PTW checks A/D bits and faults if unset (lines 487–509). Comment at line 289 explicitly describes the missing write-back logic but it is not implemented. No hardware write-back path exists.

---

## 5. Sv48 / Sv57 — page table depth

**Command:**
```bash
grep -in "sv48\|sv57\|levels\|PT_LEVELS\|4-level\|5-level\|vpn\[3\]\|vpn\[4\]" cva6_mmu/cva6_mmu.sv cva6_mmu/cva6_ptw.sv
```

**Key result lines:**
```
cva6_mmu/cva6_mmu.sv:127:    logic [CVA6Cfg.PtLevels-2:0][HYP_EXT:0] is_page;
cva6_mmu/cva6_mmu.sv:404:      if (CVA6Cfg.PtLevels == 3 && itlb_is_page[CVA6Cfg.PtLevels-2]) begin
cva6_mmu/cva6_ptw.sv:118:  logic [CVA6Cfg.PtLevels-2:0] misaligned_page;
cva6_mmu/cva6_ptw.sv:120:  logic [HYP_EXT:0][CVA6Cfg.PtLevels-2:0] ptw_lvl_n, ptw_lvl_q;
```

**PtLevels hardcoding (build_config_pkg.sv):**
```bash
cat include/build_config_pkg.sv | head -80
```

**Key line:**
```
int unsigned PtLevels = (CVA6Cfg.XLEN == 64) ? 3 : 2;   // line 29
```

**Verdict:** UNVERIFIED — design is fully parametric on `PtLevels` but it is hardcoded to 3 for 64-bit. Sv48 requires adding `Sv48En` flag and changing line 29 in `build_config_pkg.sv`.

---

## 6. pte_t struct — full definition

**Command:**
```bash
sed -n '300,340p' include/riscv_pkg.sv
```

**Result:**
```systemverilog
typedef struct packed {
  logic [9:0] reserved;   // bits 63:54 — MT[1:0] subsumed here for Svpbmt
  logic [44-1:0] ppn;
  logic [1:0] rsw;
  logic d;
  logic a;
  logic g;
  logic u;
  logic x;
  logic w;
  logic r;
  logic v;
} pte_t;                  // lines 305–318
```

---

## 7. envcfg_rv_t struct — full definition

**Command:**
```bash
sed -n '125,145p' include/riscv_pkg.sv
```

**Result:**
```systemverilog
typedef struct packed {
  logic        stce;       // bit 63 — not implemented, Sstc extension
  logic        pbmte;      // bit 62 — NOT IMPLEMENTED, requires Svpbmt  ← line 133
  logic [61:8] wpri1;
  logic        cbze;       // bit 7  — LIVE, Zicboz
  logic        cbcfe;      // bit 6  — LIVE, Zicbom
  logic [1:0]  cbie;       // bits 5:4 — LIVE, Zicbom
  logic [2:0]  wpri0;
  logic        fiom;       // bit 0
} envcfg_rv_t;             // lines 130–140
```

---

## 8. csr_regfile.sv — menvcfg write block

**Command:**
```bash
sed -n '1690,1715p' csr_regfile.sv
```

**Result:**
```systemverilog
riscv::CSR_MENVCFG: begin
  if (CVA6Cfg.RVU) begin
    fiom_d = csr_wdata[0];
  end
  if (CVA6Cfg.RVZiCbom) begin
    unique case (csr_wdata[5:4])
      2'b00:   mcbie_d = riscv::CBIE_ILLEGAL;
      2'b01:   mcbie_d = riscv::CBIE_FLUSH;
      2'b11:   mcbie_d = riscv::CBIE_INVAL;
      default: mcbie_d = riscv::CBIE_RSVD;
    endcase
    mcbcfe_d = csr_wdata[6];
  end
  // NOTE: NO pbmte case exists here — writes silently dropped
end
```

**Verdict:** `pbmte` bit is declared in struct but write is silently dropped. Adding `if (CVA6Cfg.SvpbmtEn) begin mpbmte_d = csr_wdata[62]; end` here mirrors the existing Zicbom pattern exactly.

---

## 9. csr_regfile.sv — Zicbom register declarations (pattern to copy)

**Command:**
```bash
grep -n "mcbcfe\|mcbie\|fiom" csr_regfile.sv | head -20
```

**Result:**
```
132:    output riscv::cbie_t mcbie_o,
138:    output logic mcbcfe_o,
269:  logic fiom_d, fiom_q;
321:  riscv::cbie_t mcbie_q, mcbie_d, scbie_q, scbie_d, hcbie_q, hcbie_d;
322:  logic mcbcfe_q, mcbcfe_d, scbcfe_q, scbcfe_d, hcbcfe_q, hcbcfe_d;
533:            csr_rdata = '0 | fiom_q;
636:            csr_rdata[6]   = mcbcfe_q;
1061:    fiom_d     = fiom_q;
1099:    mcbie_d                = mcbie_q;
1103:    mcbcfe_d               = mcbcfe_q;
1389:            fiom_d = csr_wdata[0];
1697:            fiom_d = csr_wdata[0];
1701:              2'b00:   mcbie_d = riscv::CBIE_ILLEGAL;
```

---

## 10. decoder.sv — sfence pattern (template for Svinval)

**Command:**
```bash
grep -n "sfence\|SFENCE\|FENCE\|fu_op\|FU_OP" core/decoder.sv | head -50
```

**Key result lines:**
```
214:              // check if the RD and RS1 fields are zero, may be reset for the SFENCE.VMA instruction
296:                // SFENCE.VMA
299:                    // check privilege level, SFENCE.VMA can only be executed in M/S mode
307:                    instruction_o.op = ariane_pkg::SFENCE_VMA;
308:                    // check TVM flag and intercept SFENCE.VMA call if necessary
315:                      instruction_o.op = ariane_pkg::HFENCE_VVMA;
322:                      instruction_o.op = ariane_pkg::HFENCE_GVMA;
466:            // FENCE
468:            3'b000: instruction_o.op = ariane_pkg::FENCE;
469:            // FENCE.I
470:            3'b001: instruction_o.op = ariane_pkg::FENCE_I;
```

---

## 11. ariane_pkg.sv — fu_op enum (Svinval new entries go here)

**Command:**
```bash
grep -n "typedef enum\|SFENCE\|sfence\|TLB\|tlb\|pte_cva6\|typedef.*pte" include/ariane_pkg.sv | head -50
```

**Key result lines:**
```
274:  typedef enum logic [7:0] {  // basic ALU op
311:    SFENCE_VMA,
650:  typedef enum logic [3:0] {
677:  } tlb_update_sv32_t;
```

**Note:** `SFENCE_VMA` is at line 311 inside `fu_op` enum declared as `logic [7:0]`. Add `SINVAL_VMA`, `SFENCE_W_INVAL`, `SFENCE_INVAL_IR` immediately after.

---

## 12. config_pkg.sv — SvnapotEn and PtLevels (pattern for new flags)

**Command:**
```bash
grep -n "SvnapotEn\|SvpbmtEn\|RVH\|PtLevels\|XLEN\|SV\b" include/config_pkg.sv | head -50
```

**Key result lines:**
```
64:    int unsigned                 XLEN;
78:    bit                          RVH;
265:    bit          SvnapotEn;
351:    bit SvnapotEn;
354:    int unsigned PtLevels;
424:    vm_mode_t MODE_SV;
425:    int unsigned SV;
```

---

## 13. load_store_unit.sv — MMU output interface (no MT signal currently)

**Command:**
```bash
grep -n "cacheable\|cache_req\|lsu_paddr\|dtlb\|pmp\|mmu" load_store_unit.sv | head -50
```

**Key result lines:**
```
220:  logic [CVA6Cfg.VLEN-1:0] mmu_vaddr, cva6_mmu_vaddr, acc_mmu_vaddr;
221:  logic [CVA6Cfg.PLEN-1:0] mmu_paddr, cva6_mmu_paddr, acc_mmu_paddr, lsu_paddr;
229:  logic dtlb_hit, cva6_dtlb_hit, acc_dtlb_hit;
287:        .lsu_dtlb_hit_o(dtlb_hit),
288:        .lsu_dtlb_ppn_o(dtlb_ppn),
291:        .lsu_paddr_o    (lsu_paddr),
```

**Verdict:** Only `lsu_paddr_o` at line 291. No cacheability or memory-type signal. `lsu_pbmt_o[1:0]` needs to be added alongside `lsu_paddr_o`.

---

## 14. tlb_update_cva6_t — definition location

**Command:**
```bash
grep -rn "tlb_update_cva6_t\|is_napot_64k" ../
```

**Key result:**
```
../core/cva6_mmu/cva6_mmu.sv:124:  localparam type tlb_update_cva6_t = struct packed {
../core/cva6_mmu/cva6_mmu.sv:126:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
../core/cva6_mmu/cva6_ptw.sv:28:    parameter type tlb_update_cva6_t = logic,
../core/cva6_mmu/cva6_shared_tlb.sv:26:    parameter type tlb_update_cva6_t = logic,
../core/cva6_mmu/cva6_tlb.sv:29:    parameter type tlb_update_cva6_t = logic,
```

**Critical finding:** `tlb_update_cva6_t` is a `localparam type` defined inside `cva6_mmu.sv` line 124 — NOT in any shared package. Adding `logic [1:0] pbmt` there is a single change that automatically propagates to `cva6_ptw`, `cva6_tlb`, and `cva6_shared_tlb` because they receive it as a parameter type.

---

## 15. OpenPiton integration boundary

**Command:**
```bash
grep -n "mmu\|tlb\|sfence\|cacheable" ../corev_apu/openpiton/ariane_verilog_wrap.sv | head -40
```

**Result:**
```
35:  // cacheable regions
```

**Note:** OpenPiton already has a "cacheable regions" concept via address-range checking. Svpbmt extends this with per-page granularity. The `wt_l15_adapter.sv` is the cache-to-NoC boundary where MT enforcement must be present for the OpenPiton path.

---

