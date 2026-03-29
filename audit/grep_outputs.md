## CVA6 RVA23 Audit — Grep Outputs
## Repository: cva6 (master branch, March 2026)
## Purpose: Complete evidence base for GSoC 2026 gap analysis
## Working directory notes:
##   [mmu/]   = ~/Downloads/cva6/core/cva6_mmu/
##   [core/]  = ~/Downloads/cva6/core/
##   [root/]  = ~/Downloads/cva6/

---

### Part 1 — Repository Structure

---

#### 1.1 Full repository layout

**Command** `[root/]`
```bash
ls -R
```

**Key directories confirmed:**
```
./core:
  cva6.sv  decoder.sv  id_stage.sv  ex_stage.sv  load_store_unit.sv
  csr_regfile.sv  commit_stage.sv  cva6_rvfi_probes.sv
  cva6_mmu/   cache_subsystem/   include/   frontend/

./core/cva6_mmu:
  cva6_mmu.sv  cva6_ptw.sv  cva6_shared_tlb.sv  cva6_tlb.sv

./core/cache_subsystem:
  std_nbdcache.sv              wt_dcache.sv
  std_cache_subsystem.sv       wt_cache_subsystem.sv
  cva6_hpdcache_wrapper.sv     cva6_hpdcache_subsystem.sv
  cva6_icache.sv               wt_l15_adapter.sv
  miss_handler.sv

./core/include:
  ariane_pkg.sv   build_config_pkg.sv   config_pkg.sv   riscv_pkg.sv
  cv64a6_imafdc_sv39_openpiton_config_pkg.sv
  [many other per-target config_pkg variants]

./corev_apu/openpiton:
  ariane_verilog_wrap.sv   bootrom/   riscv_peripherals.sv

./verif/regress:
  dv-riscv-arch-test.sh   linux.sh   [many other test runners]

./verif/tests/custom/sv32:
  [40+ hand-written MMU assembly tests — model for new Svpbmt/Svinval tests]
```

**Finding:** Three cache backends in `cache_subsystem/`: `wt_dcache.sv` (OpenPiton writethrough path), `std_nbdcache.sv` (standard writeback), `cva6_hpdcache_wrapper.sv` (HPDcache). `wt_l15_adapter.sv` is the cache-to-OpenPiton-NoC boundary. `cva6_rvfi_probes.sv` is the RVFI trace module used for test failure observation.

---

### Part 2 — Extension Status Sweeps (MMU files)

All commands run from `[mmu/]` unless noted.

---

#### 2.1 Svnapot — naturally aligned huge pages

**Command** `[mmu/]`
```bash
grep -in "napot\|svnapot\|contiguous\|NAPOT" cva6_mmu.sv cva6_ptw.sv cva6_tlb.sv cva6_shared_tlb.sv
```

**Result:**
```
cva6_mmu.sv:126:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_ptw.sv:101:  //Field for SVNAPOT
cva6_ptw.sv:102:  logic is_napot_64k;
cva6_ptw.sv:206:    shared_tlb_update_o.is_napot_64k = is_napot_64k;
cva6_ptw.sv:309:    is_napot_64k            = 1'b0;
cva6_ptw.sv:430:          if (!pte.v || (!pte.r && pte.w) || (|pte.reserved && CVA6Cfg.XLEN == 64) || (!CVA6Cfg.SvnapotEn && pte.n) || (CVA6Cfg.SvnapotEn && !(pte.r || pte.x) && pte.n))
cva6_ptw.sv:440:              if (CVA6Cfg.SvnapotEn && pte.n) begin
cva6_ptw.sv:442:                is_napot_64k = (pte.ppn[3:0] == 4'b1000) && (ptw_lvl_q[0] == 2);
cva6_ptw.sv:443:                // Svnapot: Any other encoding with the N-bit set is a reserved format and must cause a page fault
cva6_ptw.sv:445:                if (!is_napot_64k) begin
cva6_tlb.sv:72:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_tlb.sv:98:  logic [TLB_ENTRIES-1:0] napot_tag_match;
cva6_tlb.sv:100:  logic [TLB_ENTRIES-1:0] vpn0_napot_match;
cva6_tlb.sv:168:      assign napot_tag_match[i_gen] = (CVA6Cfg.SvnapotEn && tags_q[i_gen].is_napot_64k) ? ...
cva6_tlb.sv:259:          if (tags_q[i].is_napot_64k && CVA6Cfg.SvnapotEn) begin
cva6_tlb.sv:381:        if (update_i.is_napot_64k && CVA6Cfg.SvnapotEn) begin
cva6_tlb.sv:395:          CVA6Cfg.SvnapotEn ? update_i.is_napot_64k : 1'b0
cva6_shared_tlb.sv:92:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
cva6_shared_tlb.sv:148:  logic [SHARED_TLB_WAYS-1:0] vpn0_napot_match;
cva6_shared_tlb.sv:149:  logic [SHARED_TLB_WAYS-1:0] napot_tag_match;
cva6_shared_tlb.sv:297:          itlb_update_o.is_napot_64k = shared_tlb_update_i.is_napot_64k;
cva6_shared_tlb.sv:308:          dtlb_update_o.is_napot_64k = shared_tlb_update_i.is_napot_64k;
cva6_shared_tlb.sv:416:    (shared_tlb_update_i.is_napot_64k && CVA6Cfg.SvnapotEn)
cva6_shared_tlb.sv:443:  assign shared_tag_wr.is_napot_64k = shared_tlb_update_i.is_napot_64k;
```

**Verdict:** COMPLETE — 40+ hits. N-bit decode, NAPOT hit logic, VPN masking, PPN patching, flush logic all present. Gated on `CVA6Cfg.SvnapotEn`.

---

#### 2.2 Svpbmt — page-based memory types

**Command** `[mmu/]`
```bash
grep -in "pbmt\|svpbmt\|memory.type\|MT_bits\|menvcfg\|senvcfg" cva6_mmu.sv cva6_ptw.sv cva6_tlb.sv cva6_shared_tlb.sv
```

**Result:** `(no output)`

**Verdict:** COMPLETELY MISSING from all four MMU files.

---

#### 2.3 Svinval — fine-grained TLB invalidation (MMU files)

**Command** `[mmu/]`
```bash
grep -in "svinval\|sinval\|sfence.w\|hinval\|hfence" cva6_mmu.sv cva6_ptw.sv cva6_tlb.sv cva6_shared_tlb.sv
```

**Result (comments only, no implementation):**
```
cva6_tlb.sv:350:          // ("SFENCE.VMA/HFENCE.VVMA x0 x0" case)
cva6_tlb.sv:353:          // ("SFENCE.VMA/HFENCE.VVMA vaddr x0" case)
cva6_tlb.sv:356:          // ("SFENCE.VMA/HFENCE.VVMA vaddr asid" case)
cva6_tlb.sv:359:          // ("SFENCE.VMA/HFENCE.VVMA 0 asid" case)
cva6_tlb.sv:366:          // ("HFENCE.GVMA x0 x0" case)
cva6_tlb.sv:368:          // ("HFENCE.GVMA gpaddr x0" case)
```

---

#### 2.4 Svinval — full repo confirmation

**Command** `[root/]`
```bash
grep -rn "sinval\|SINVAL" ../
```
**Result:** `(no output)`

**Command** `[core/]`
```bash
grep -in "sinval\|svinval\|sfence_w\|sfence_i" ../id_stage.sv ../decoder.sv 2>/dev/null || grep -rn "sinval\|SINVAL" ../
```
**Result:** `(no output from either command)`

**Verdict:** COMPLETELY MISSING — `sinval.vma`, `sfence.w.inval`, `sfence.inval.ir` not decoded anywhere in the repo. Not in decoder, not in id_stage, not in any MMU file.

---

#### 2.5 Svadu / Svade — accessed/dirty bit management

**Command** `[mmu/]`
```bash
grep -in "svade\|svadu\|accessed\|dirty\|pte\.a\|pte\.d\|hw_update\|page.fault" cva6_ptw.sv cva6_mmu.sv
```

**Key result lines:**
```
cva6_ptw.sv:289:  //    Set pte.a to 1, and, if the memory access is a store, set pte.d to 1.
cva6_ptw.sv:487:                if ((!pte.x && (!CVA6Cfg.RVH || ptw_stage_q != G_INTERMED_STAGE)) || !pte.a
cva6_ptw.sv:504:                  (pte.a && ((pte.r && !hlvx_inst_i) || ...))
cva6_ptw.sv:509:                    && (!lsu_is_store_i || (pte.w && pte.d) || ...)
```

**Verdict:** PARTIAL (Svade only) — PTW checks A/D bits and faults if unset (lines 487–509). Comment at line 289 explicitly describes the missing write-back logic. No hardware write-back path exists.

---

#### 2.6 Sv48 / Sv57 — page table depth

**Command** `[mmu/]`
```bash
grep -in "sv48\|sv57\|levels\|PT_LEVELS\|4-level\|5-level\|vpn\[3\]\|vpn\[4\]" cva6_mmu.sv cva6_ptw.sv
```

**Key result lines:**
```
cva6_mmu.sv:127:    logic [CVA6Cfg.PtLevels-2:0][HYP_EXT:0] is_page;
cva6_mmu.sv:404:      if (CVA6Cfg.PtLevels == 3 && itlb_is_page[CVA6Cfg.PtLevels-2]) begin
cva6_ptw.sv:118:  logic [CVA6Cfg.PtLevels-2:0] misaligned_page;
cva6_ptw.sv:120:  logic [HYP_EXT:0][CVA6Cfg.PtLevels-2:0] ptw_lvl_n, ptw_lvl_q;
```

**Verdict:** Design is fully parametric on `PtLevels`. The hardcoding is in `build_config_pkg.sv:29` — see section 4.2.

---

### Part 3 — Struct and Type Definitions

---

#### 3.1 pte_t struct — full definition

**Command** `[core/]`
```bash
sed -n '300,340p' include/riscv_pkg.sv
```

**Result:**
```systemverilog
  localparam OpcodeC2Sdsp = 3'b111;
  // ----------------------
  // Virtual Memory
  // ----------------------
  // memory management, pte for sv39
  typedef struct packed {
    logic [9:0] reserved;
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
  } pte_t;               // lines 305–318
  // memory management, pte for sv32
  typedef struct packed {
    logic [22-1:0] ppn;
    logic [1:0] rsw;
    logic d;
    logic a;
    logic g;
    logic u;
    logic x;
    logic w;
    logic r;
    logic v;
  } pte_sv32_t;
```

**Finding:** `pte_t` has `logic [9:0] reserved` as its first field. The Svpbmt MT bits (63:62) are buried here and currently trigger a page fault at `ptw.sv:430`. Change: split into `logic [1:0] mt` + `logic [7:0] reserved`. `pte_sv32_t` is unaffected.

---

#### 3.2 pte_t typedef search across full repo

**Command** `[core/]`
```bash
grep -rn "typedef.*pte\|struct.*pte\|pte_t\|pte_sv" ../
```

**Key results:**
```
../include/ariane_pkg.sv:676:    riscv::pte_sv32_t content;
../include/ariane_pkg.sv:838:                                              input riscv::pte_t pte);
../include/ariane_pkg.sv:855:                                            input riscv::pte_t pte);
../include/riscv_pkg.sv:318:  } pte_t;
../include/riscv_pkg.sv:332:  } pte_sv32_t;
```

**Finding:** `pte_t` defined only in `riscv_pkg.sv:318`. Changing it there is sufficient.

---

#### 3.3 envcfg_rv_t struct — full definition

**Command** `[core/]`
```bash
sed -n '125,145p' include/riscv_pkg.sv
```

**Result:**
```systemverilog
    logic mie;  // machine interrupts enable
    logic wpri1;  // writes preserved reads ignored
    logic sie;  // supervisor interrupts enable
    logic wpri0;  // writes preserved reads ignored
  } mstatus_rv_t;
  typedef struct packed {
    logic        stce;   // not implemented - requires Sctc extension
    logic        pbmte;  // not implemented - requires Svpbmt extension   ← line 133
    logic [61:8] wpri1;  // writes preserved reads ignored
    logic        cbze;   // not implemented - requires Zicboz extension
    logic        cbcfe;  // not implemented - requires Zicbom extension
    logic [1:0]  cbie;   // not implemented - requires Zicbom extension
    logic [2:0]  wpri0;  // writes preserved reads ignored
    logic        fiom;   // fence of I/O implies memory
  } envcfg_rv_t;         // lines 130–140
```

**Finding:** `pbmte` is at bit 62, declared but completely unwired. The "not implemented" comments on `cbze`/`cbcfe`/`cbie` are misleading — those ARE live in `csr_regfile.sv`. `pbmte` is the only field still not connected.

---

#### 3.4 envcfg search in riscv_pkg

**Command** `[core/]`
```bash
grep -n "menvcfg\|senvcfg\|envcfg\|pbmte" include/riscv_pkg.sv
```

**Result:**
```
133:    logic        pbmte;  // not implemented - requires Svpbmt extension
140:  } envcfg_rv_t;
```

**Finding:** Two hits only. No CSR address constants here — those live in `csr_regfile.sv` as `riscv::CSR_MENVCFG`.

---

#### 3.5 SvpbmtEn / SvinvalEn / SvaduEn — flags search across full repo

**Command** `[root/]`
```bash
grep -rn "Svpbmt\|Svinval\|Svadu\|SvpbmtEn\|SvinvalEn\|SvaduEn" ../
```

**Result:**
```
../include/riscv_pkg.sv:133:    logic        pbmte;  // not implemented - requires Svpbmt extension
```

**Finding:** One hit in the entire repo. No `SvpbmtEn`, `SvinvalEn`, or `SvaduEn` config flags exist anywhere. All three need adding to `config_pkg.sv` following the `SvnapotEn` pattern at line 351.

---

#### 3.6 tlb_update_cva6_t — definition and propagation (full repo)

**Command** `[root/]`
```bash
grep -rn "tlb_update_cva6_t\|is_napot_64k" ../
```

**Key results (MMU files — docs omitted):**
```
../core/cva6_mmu/cva6_mmu.sv:124:  localparam type tlb_update_cva6_t = struct packed {
../core/cva6_mmu/cva6_mmu.sv:126:    logic is_napot_64k;  // Svnapot: Flag indicating a 64KiB NAPOT page
../core/cva6_mmu/cva6_mmu.sv:153:  tlb_update_cva6_t update_itlb, update_dtlb, update_shared_tlb;
../core/cva6_mmu/cva6_mmu.sv:185:      .tlb_update_cva6_t(tlb_update_cva6_t),
../core/cva6_mmu/cva6_mmu.sv:216:      .tlb_update_cva6_t(tlb_update_cva6_t),
../core/cva6_mmu/cva6_mmu.sv:250:      .tlb_update_cva6_t(tlb_update_cva6_t),
../core/cva6_mmu/cva6_ptw.sv:28:    parameter type tlb_update_cva6_t = logic,
../core/cva6_mmu/cva6_ptw.sv:59:    output tlb_update_cva6_t shared_tlb_update_o,
../core/cva6_mmu/cva6_shared_tlb.sv:26:    parameter type tlb_update_cva6_t = logic,
../core/cva6_mmu/cva6_shared_tlb.sv:60:    output tlb_update_cva6_t itlb_update_o,
../core/cva6_mmu/cva6_shared_tlb.sv:61:    output tlb_update_cva6_t dtlb_update_o,
../core/cva6_mmu/cva6_shared_tlb.sv:74:    input tlb_update_cva6_t shared_tlb_update_i,
../core/cva6_mmu/cva6_tlb.sv:29:    parameter type tlb_update_cva6_t = logic,
../core/cva6_mmu/cva6_tlb.sv:42:    input tlb_update_cva6_t update_i,
```

**Critical finding:** `tlb_update_cva6_t` defined ONCE as `localparam type` at `cva6_mmu.sv:124`. Passed as `parameter type` to all three submodules. Adding `logic [1:0] pbmt` at line 124 propagates everywhere automatically.

---

#### 3.7 tlb_update_cva6_t — confirms NOT in any include/ package

**Command** `[root/]`
```bash
grep -rn "tlb_update_cva6_t\|typedef.*tlb_update" include/
```

**Result:** `(no output)`

**Finding:** `tlb_update_cva6_t` does not exist in any include/ package file. It is purely internal to `cva6_mmu.sv`. There is no shared package definition to maintain — changing line 124 is the only change needed.

---

#### 3.8 pte_cva6_t and tlb_update_cva6_t — confirms NOT in ariane_pkg or cva6.sv

**Command** `[core/]`
```bash
grep -n "pte_cva6\|pte_cva6_t\|typedef.*pte" include/ariane_pkg.sv | head -20
```
**Result:** `(no output)`

**Command** `[core/]`
```bash
grep -n "tlb_update_cva6_t\|pte_cva6_t" cva6.sv | head -20
```
**Result:** `(no output)`

**Finding:** Both types are not in `ariane_pkg.sv` or `cva6.sv`. They are parameter types passed at instantiation from per-target config packages. The top-level core does not need to be touched.

---

### Part 4 — Include File Investigations

---

#### 4.1 config_pkg.sv — SvnapotEn, PtLevels, XLEN flags

**Command** `[core/]`
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

**Finding:** `SvnapotEn` appears at both line 265 (user config struct) and line 351 (built config struct). New flags `SvpbmtEn`, `SvinvalEn`, `Sv48En` need to be added in both locations after `SvnapotEn`.

---

#### 4.2 build_config_pkg.sv — PtLevels hardcoding and SvnapotEn propagation pattern

**Command** `[core/]`
```bash
cat include/build_config_pkg.sv | head -80
```

**Key section:**
```systemverilog
package build_config_pkg;

  function automatic config_pkg::cva6_cfg_t build_config(config_pkg::cva6_user_cfg_t CVA6Cfg);
    bit IS_XLEN32 = (CVA6Cfg.XLEN == 32) ? 1'b1 : 1'b0;
    bit IS_XLEN64 = (CVA6Cfg.XLEN == 32) ? 1'b0 : 1'b1;
    ...
    // MMU
    int unsigned VpnLen = (CVA6Cfg.XLEN == 64) ? (CVA6Cfg.RVH ? 29 : 27) : 20;
    int unsigned PtLevels = (CVA6Cfg.XLEN == 64) ? 3 : 2;   // ← LINE 29 — HARDCODED

    config_pkg::cva6_cfg_t cfg;
    cfg.XLEN = CVA6Cfg.XLEN;
    ...
    cfg.RVH = CVA6Cfg.RVH;
    ...
    cfg.SvnapotEn = CVA6Cfg.SvnapotEn;   // ← pattern to replicate for SvpbmtEn
```

**Finding:** `PtLevels` hardcoded at line 29. Change to: `(CVA6Cfg.XLEN == 32) ? 2 : CVA6Cfg.Sv48En ? 4 : 3`. `cfg.SvnapotEn = CVA6Cfg.SvnapotEn` is the exact pattern to replicate.

---

#### 4.3 build_config_pkg.sv — tlb_update and pte_cva6 search (confirms types not here)

**Command** `[core/]`
```bash
grep -n "tlb_update\|pte_cva6" include/build_config_pkg.sv | head -30
```

**Result:** `(no output)`

**Finding:** Confirms `tlb_update_cva6_t` and `pte_cva6_t` are not in `build_config_pkg.sv`.

---

#### 4.4 ariane_pkg.sv — fu_op enum and tlb_update_sv32_t

**Command** `[core/]`
```bash
grep -n "typedef enum\|SFENCE\|sfence\|TLB\|tlb\|pte_cva6\|typedef.*pte" include/ariane_pkg.sv | head -50
```

**Key result lines:**
```
274:  typedef enum logic [7:0] {  // basic ALU op
311:    SFENCE_VMA,
650:  typedef enum logic [3:0] {
677:  } tlb_update_sv32_t;
679:  typedef enum logic [1:0] {
```

**Finding:** `fu_op` enum at line 274 as `logic [7:0]` — 256 opcodes, room for Svinval entries. `SFENCE_VMA` at line 311. `tlb_update_sv32_t` closes at line 677 — the Sv32 struct (the Sv39/64 version is the `localparam type` in `cva6_mmu.sv`).

---

### Part 5 — CSR Register File

---

#### 5.1 menvcfg / senvcfg full search

**Command** `[core/]`
```bash
grep -n "menvcfg\|senvcfg\|pbmte\|ENVCFG\|envcfg" csr_regfile.sv | head -40
```

**Result:**
```
320:  // CBO enable flags from menvcfg/senvcfg/henvcfg
531:        riscv::CSR_SENVCFG: begin
576:        riscv::CSR_HENVCFG: begin
630:        riscv::CSR_MENVCFG: begin
643:        riscv::CSR_MENVCFGH: begin
1387:        riscv::CSR_SENVCFG: begin
1514:        riscv::CSR_HENVCFG: begin
1695:        riscv::CSR_MENVCFG: begin
1709:        riscv::CSR_MENVCFGH: begin
```

**Finding:** `CSR_MENVCFG` handled at read (line 630) and write (line 1695). `CSR_SENVCFG` at lines 531 and 1387. `pbmte` must be added at both read and write locations for both menvcfg and senvcfg.

---

#### 5.2 menvcfg write block — full content

**Command** `[core/]`
```bash
sed -n '1690,1715p' csr_regfile.sv
```

**Result:**
```systemverilog
          end else begin
            mask = '0;
          end
          mip_d = (mip_q & ~mask) | (csr_wdata & mask);
        end
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
        end
        riscv::CSR_MENVCFGH: begin
          if (!CVA6Cfg.RVU || CVA6Cfg.XLEN != 32) update_access_exception = 1'b1;
        end
```

**Finding:** No `pbmte` case exists. Writes to bit 62 are silently dropped. Fix: add `if (CVA6Cfg.SvpbmtEn) begin mpbmte_d = csr_wdata[62]; end` inside this case, mirroring `mcbcfe_d = csr_wdata[6]`.

---

#### 5.3 Zicbom register declarations — pattern to copy for pbmte

**Command** `[core/]`
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

**Finding:** Full pattern for pbmte: output port (132/138), d/q registers (321–322), reset default (1099/1103), CSR read (636), CSR write (1700). Mirror for `mpbmte_d/q` and `spbmte_d/q` at bit 62.

---

### Part 6 — Decoder and Pipeline

---

#### 6.1 decoder.sv — sfence and FENCE decode pattern

**Command** `[core/]`
```bash
grep -n "sfence\|SFENCE\|FENCE\|fu_op\|FU_OP" decoder.sv | head -50
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

**Finding:** `SFENCE.VMA` decode at lines 296–307. Privilege check pattern (lines 299–307) is the direct template for Svinval's three new instructions. `HFENCE_VVMA` and `HFENCE_GVMA` at lines 315–331 show the hypervisor variant.

---

### Part 7 — Load/Store Unit and Cache Interface

---

#### 7.1 load_store_unit.sv — MMU output interface

**Command** `[core/]`
```bash
grep -n "cacheable\|cache_req\|lsu_paddr\|dtlb\|pmp\|mmu" load_store_unit.sv | head -50
```

**Key result lines:**
```
20:    parameter type dcache_req_i_t = logic,
21:    parameter type dcache_req_o_t = logic,
93:    input  acc_mmu_req_t  acc_mmu_req_i,
94:    output acc_mmu_resp_t acc_mmu_resp_o,
148:    output logic          dtlb_miss_o,
151:    input  dcache_req_o_t [2:0] dcache_req_ports_i,
153:    output dcache_req_i_t [2:0] dcache_req_ports_o,
220:  logic [CVA6Cfg.VLEN-1:0] mmu_vaddr, cva6_mmu_vaddr, acc_mmu_vaddr;
221:  logic [CVA6Cfg.PLEN-1:0] mmu_paddr, cva6_mmu_paddr, acc_mmu_paddr, lsu_paddr;
229:  logic dtlb_hit, cva6_dtlb_hit, acc_dtlb_hit;
257:  if (CVA6Cfg.MmuPresent) begin : gen_mmu
260:    cva6_mmu #(
267:        .dcache_req_i_t(dcache_req_i_t),
268:        .dcache_req_o_t(dcache_req_o_t),
287:        .lsu_dtlb_hit_o(dtlb_hit),
288:        .lsu_dtlb_ppn_o(dtlb_ppn),
290:        .lsu_valid_o    (pmp_translation_valid),
291:        .lsu_paddr_o    (lsu_paddr),
292:        .lsu_exception_o(pmp_exception),
```

**Finding:** Only `lsu_paddr_o` at line 291 — no cacheability or memory-type signal. `lsu_pbmt_o[1:0]` needs to be declared alongside `lsu_paddr` at line 221 and connected at the `cva6_mmu` instance at line 291.

---

### Part 8 — MMU Module Headers and Internal Structure

---

#### 8.1 cva6_ptw.sv — module header

**Command** `[mmu/]`
```bash
head -30 cva6_ptw.sv
```

**Result:**
```
// Copyright (c) 2018 ETH Zurich and University of Bologna.
// Copyright (c) 2021 Thales.
// Copyright (c) 2022 Bruno Sá and Zero-Day Labs.
// Copyright (c) 2024 PlanV Technology
// SPDX-License-Identifier: Apache-2.0 WITH SHL-2.1
//
// Author: Angela Gonzalez, PlanV Technology
// Date: 26/02/2024
// Description: Hardware-PTW for CVA6 supporting sv32, sv39 and sv39x4.

module cva6_ptw
  import ariane_pkg::*;
#(
    parameter config_pkg::cva6_cfg_t CVA6Cfg = config_pkg::cva6_cfg_empty,
    parameter type pte_cva6_t = logic,
    parameter type tlb_update_cva6_t = logic,
    parameter type dcache_req_i_t = logic,
    parameter type dcache_req_o_t = logic,
```

---

#### 8.2 cva6_tlb.sv — module header

**Command** `[mmu/]`
```bash
head -30 cva6_tlb.sv
```

**Result:**
```
// Author: Angela Gonzalez PlanV Technology
// Date: 26/02/2024
// Description: Translation Lookaside Buffer, parameterizable to Sv32 or Sv39,
//              or sv39x4 fully set-associative

module cva6_tlb
  import ariane_pkg::*;
#(
    parameter config_pkg::cva6_cfg_t CVA6Cfg = config_pkg::cva6_cfg_empty,
    parameter type pte_cva6_t = logic,
    parameter type tlb_update_cva6_t = logic,
    parameter int unsigned TLB_ENTRIES = 4,
```

**Finding (8.1 + 8.2):** Both modules receive `tlb_update_cva6_t` as a `parameter type`. The `localparam type` defined in `cva6_mmu.sv:124` is passed at instantiation — not imported from a package. One struct change propagates to all submodules.

---

#### 8.3 cva6_ptw.sv — reserved bit fault and pte field access

**Command** `[mmu/]`
```bash
grep -in "pte\.\|rsvd\|reserved\|bits\[" cva6_ptw.sv | head -60
```

**Key result lines:**
```
279:  // 3. If pte.v = 0, or if pte.r = 0 and pte.w = 1, or if any bits or encodings
280:  //    that are reserved for future standard use are set within pte, stop and raise
282:  // 4. Otherwise, the PTE is valid. If pte.r = 1 or pte.x = 1, go to step 5.
285:  //    a = pte.ppn × PAGESIZE and go to step 2.
287:  //    is allowed by the pte.r, pte.w, and pte.x bits. If not, stop and
289:  //    Set pte.a to 1, and, if the memory access is a store, set pte.d to 1.
424:          if (pte.g && ...) global_mapping_n = 1'b1;
429:          // If pte.v = 0, or if pte.r = 0 and pte.w = 1, or if pte.reserved !=0 in sv39 and sv39x4
430:          if (!pte.v || (!pte.r && pte.w) || (|pte.reserved && CVA6Cfg.XLEN == 64) || (!CVA6Cfg.SvnapotEn && pte.n) || ...)
438:            if (pte.r || pte.x) begin
440:              if (CVA6Cfg.SvnapotEn && pte.n) begin
442:                is_napot_64k = (pte.ppn[3:0] == 4'b1000) && (ptw_lvl_q[0] == 2);
487:                if ((!pte.x && ...) || !pte.a
504:                  (pte.a && ((pte.r && !hlvx_inst_i) || ...))
509:                    && (!lsu_is_store_i || (pte.w && pte.d) || ...)
580:                if (CVA6Cfg.RVH && (pte.a || pte.d || pte.u)) begin
```

**Critical finding — line 430:** `(|pte.reserved && CVA6Cfg.XLEN == 64)` must be relaxed for Svpbmt. When `SvpbmtEn` and `menvcfg.pbmte` are set, the MT bits must be excluded from this check. The fix mirrors how `SvnapotEn && pte.n` is handled on the same line.

---

### Part 9 — OpenPiton Integration

---

#### 9.1 ariane_verilog_wrap.sv — OpenPiton boundary

**Command** `[root/]`
```bash
grep -n "mmu\|tlb\|sfence\|cacheable" ../corev_apu/openpiton/ariane_verilog_wrap.sv | head -40
```

**Result:**
```
35:  // cacheable regions
```

**Finding:** OpenPiton has a "cacheable regions" concept via address-range checking. Svpbmt extends this to per-page granularity. `wt_l15_adapter.sv` is the cache-to-P-Mesh-NoC boundary where MT enforcement must also be present for the OpenPiton path.

---
