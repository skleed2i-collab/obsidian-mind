---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top Top10 Candidates Reset 800us 1p3ms"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_top10_candidates_reset_800us_1p3ms.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top top-10 candidates for reset-low 800us~1.3ms only

Assumptions added:
- OSC speed is about 60MHz.
- TO_RESX_SPROUT reset-low duration issue is observed only when low interval is about 800us ~ 1.3ms.
- User-provided analog behavior remains valid:
  - TO_RESX_SPROUT low -> OSC off, TO_OSC_CLK absent.
  - OSC_BIAS_EN / OSC_CLK_EN assertion to actual OSC output has about 3~5us delay.
  - *_Y pad-derived inputs settle with several-ns skew around TO_RESX_SPROUT 0->1.

Useful numbers at 60MHz:
- OSC period: about 16.667ns.
- 3us on-time delay: about 180 OSC cycles.
- 5us on-time delay: about 300 OSC cycles.
- sys_clk_osc_overshoot_protect div8 period: about 133.333ns.
- 9 div8 cycles for full mask progression: about 1.2us.
- 800us reset-low duration: about 48,000 OSC cycles if clock existed, but OSC is off during reset-low.
- 1.3ms reset-low duration: about 78,000 OSC cycles if clock existed, but OSC is off during reset-low.

Important interpretation:
- Because OSC is off while reset is low, the absolute 800us~1.3ms low duration does not provide OSC cycles to internal RTL.
- Therefore the duration dependence is likely caused by analog/pad/retention/power/RC/clock-start state, not by normal synchronous RTL counting during reset-low.
- After reset release, digital state starts from the first valid OSC/clk_sys/clk_axi_sys edges.

---

## Top 10 candidates, filtered for reset-low = 800us~1.3ms only

### 1. OSC analog restart marginality in 800us~1.3ms off interval

Why high priority:
- OSC is off during TO_RESX_SPROUT low.
- Failure only in a specific reset-low duration window strongly suggests analog restart/settling dependency.
- 800us~1.3ms may be long enough for some bias/internal node to partially discharge, but not long enough for a full cold-reset-like condition.

Possible effect:
- IP_OSC_BIAS_EN/IP_OSC_CLK_EN assert normally, but actual TO_OSC_CLK starts late, has first-cycle jitter, or has runt/missing pulses.
- sys_clk_osc_overshoot_protect may suppress or reshape early clocks.
- clk_sys/clk_axi_sys startup alignment changes.

Check:
- IP_OSC_BIAS_EN rise
- IP_OSC_CLK_EN rise
- TO_OSC_CLK first edge
- TO_OSC_CLK first 1000 cycles period/high/low histogram
- clk_sys first edge
- clk_axi_sys first edge

Expected smoking gun:
- Only 800us~1.3ms case shows longer clock-start delay, first-cycle runt pulse, missing early pulses, or abnormal duty/frequency settling.

### 2. OSC deglitch/overshoot protect startup interacting with 60MHz OSC

Why high priority:
- At 60MHz, digital deglitch full-mask progression is about 1.2us, not about 720ns as estimated for 100MHz.
- This adds on top of analog 3~5us clock output delay.

Possible effect:
- TO_OSC_CLK starts, but w_clk_oscdat_deglitch / clk_sys output is still suppressed or partially divided for about 1.2us plus edge alignment.
- If early VROM/DMA request timing assumes earlier clk_axi_sys availability, a cross-domain pulse can be missed.

Check:
- TO_OSC_CLK
- i_osc_clk_en
- i_fq_sel_change
- r_clk_i_osc_glitch_mask
- w_clk_oscdat_deglitch
- clk_sys

Smoking gun:
- TO_OSC_CLK is present, but deglitch output or clk_sys starts materially later only in 800us~1.3ms cases.

### 3. clk_sys alive before clk_axi_sys/clk_axi_sys_lp, causing DMA start pulse loss

Why high priority:
- Current state is VROM waiting for DMA end.
- VROM control is clk_sys side; DMA/AXI side is clk_axi_sys/clk_axi_sys_lp.
- Reset/clock mask can make clk_axi_sys start later than clk_sys.

Possible effect:
- o_vrom1_default_load_dma_srt is generated while clk_axi_sys or clk_axi_sys_lp is still not active/stable.
- DMA side misses the start pulse or only partially initializes.

Check:
- clk_sys first edge
- clk_axi_sys first edge
- clk_axi_sys_lp first edge
- rstb_bus
- o_vrom1_default_load_dma_srt
- i_cdc_exe3_srt
- o_exe3_srt

Smoking gun:
- In failing 800us~1.3ms window, VROM dma_srt occurs before clk_axi_sys/clk_axi_sys_lp is alive or before reset-mask completed.

### 4. IO_DIS_EXCLK_Y pad skew causing transient EXCLK selection at reset release

Why high priority:
- User reported *_Y inputs do not settle simultaneously; each has ns-scale delay.
- sys_clk uses i_dis_exclk/IO_DIS_EXCLK_Y for OSC vs EXCLK selection.
- If IO_DIS_EXCLK_Y is temporarily 0, EXCLK path may be selected.

Possible effect:
- clk_sys mux selects EXCLK when EXCLK is absent/X, then switches to OSC after IO_DIS_EXCLK_Y settles.
- This can create no-clock, X-clock, or glitchy first clock behavior.

Check:
- TO_RESX_SPROUT
- IO_DIS_EXCLK_Y
- IO_EXCLK_Y
- rm_sys_exclk_sel
- w_sys_exclk_sel
- w_clk_ex
- w_clk_m_oscsys
- w_clk_m_sys_mux
- clk_sys

Smoking gun:
- Only failing window has wrong source selected during first post-reset clock period or source switch near reset release.

### 5. Partial analog/pad state retention from 800us~1.3ms reset low

Why high priority:
- Duration-window dependency is typical of partial discharge/retention rather than pure digital reset.
- Very short reset may retain a stable previous state; very long reset may fully discharge; mid-window can be marginal.

Possible effect:
- TO_RESX_SPROUT release itself is clean, but pad-derived *_Y outputs, POR-related analog, DL_OUT, or OSC control pins have altered rise profiles.
- Reset tree sees inconsistent initial mode/clock/reset inputs.

Check:
- TO_RESX_SPROUT analog/digital edge
- TO_PORB_VDD
- DL_OUT
- all mode-related *_Y first settle times
- compare failing 800us~1.3ms vs non-failing shorter/longer resets

Smoking gun:
- One or more pad/mode signals has a different delay/skew only in the failure window.

### 6. Reset mask/counter startup depends on first valid clk_sys edges

Why relevant:
- sys_reset uses clk_sys to count mask release after rstb_sof/rstb_slp.
- If first clk_sys edges are delayed/irregular, o_clk_mask_rstb timing changes.

Possible effect:
- clk_sys exists, but downstream divided/gated clocks including clk_axi_sys become active later than expected.
- sys_fsm may progress earlier than the AXI domain.

Check:
- rstb_pwr
- rstb_sof
- rstb_slp
- r_clk_mask_rstb_sof_cnt
- r_clk_mask_rstb_slp_cnt
- o_clk_mask_rstb
- w_sys_reset_mask
- w_clk_sys_reset_mask
- clk_axi_sys

Smoking gun:
- In failing window, clk_sys FSM progresses while reset-mask still blocks downstream clock path.

### 7. AXI read target/VROM source does not respond after 9E read address

Why still high:
- Current DMA master state directly shows read address done and read data burst start/wait.
- This is the closest digital symptom.

Possible effect:
- ar handshake completed, but target does not return rvalid/rlast.
- Could be independent digital bug or caused by target reset/clock still inactive after reset-window-specific startup.

Check:
- axi4_mst_dma.arvalid/arready/araddr/arlen
- axi4_mst_dma.rvalid/rready/rlast/rresp
- target decode for MMAP_SYS_VROM1 + CMD_9E_VROM_START_ADR
- target clock/reset

Smoking gun:
- ar handshake is complete, but no rvalid comes from target only in 800us~1.3ms case.

### 8. clk_axi_sys_lp gate is not active when dma_core_slv needs it

Why relevant:
- dma_core_slv mixes clk_axi_sys FSMs with clk_axi_sys_lp ready/response/latch logic.
- clk_axi_sys_lp is gated by rm_axi_sys_lp_mode_off || i_clk_axi_sys_lp.

Possible effect:
- r_exe_fsm is active, but AXI slave/memory-side helper registers do not update correctly.
- DMA start/response path can be partially stuck.

Check:
- clk_axi_sys_lp
- rm_axi_sys_lp_mode_off
- i_clk_axi_sys_lp
- dma_core_slv r_axi4_slv_awready/wready/bvalid

Smoking gun:
- clk_axi_sys toggles but clk_axi_sys_lp is absent or starts late in failing window.

### 9. scan/test mode transient due to *_Y skew

Why relevant:
- IO_PNSW_Y, IO_TMODE_Y, IO_DBIST_Y, IO_PMODE_Y all transition around reset release in init.vcd.
- scan muxes exist in reset/clock paths.

Possible effect:
- Reset/clock path briefly selects scan reset/clock, or test mode alters clock/reset behavior during startup.

Check:
- w_scan_mode / i_scan_mode
- IO_PNSW_Y
- IO_TMODE_Y[*]
- IO_DBIST_Y
- IO_PMODE_Y
- rstb_* after scan mux
- clk_scan and clock mux outputs

Smoking gun:
- scan/test mode becomes active for a short period only in failing reset-low window.

### 10. DMA/FIFO read-data backpressure or burst count mismatch

Why included:
- Current DMA master state waits for w_rdat_end:
  rvalid & rlast & rready & w_rdat_cnt_end.
- If rready is low due to FIFO level, or count/rlast mismatch exists, DMA end never occurs.

Possible effect:
- rvalid may exist but not accepted.
- rlast may occur when w_rdat_cnt_end is false.
- FIFO push/write side may not progress.

Check:
- r_axi4_mst_rready_
- w_fifo_lp
- w_fifo_push
- w_fifo_pop
- r_rdat_cnt
- w_rdat_cnt_end
- axi4_mst_dma.rvalid/rready/rlast

Smoking gun:
- rvalid/rlast occurs but w_rdat_end never asserts due to rready or count_end mismatch.

---

## Shortlist if only analog/reset-window sensitivity is considered

If focusing only on why the problem appears when reset-low is 800us~1.3ms, the top five are:

1. OSC analog restart marginality in that off-time window.
2. Pad/mode signal partial-retention/skew after that reset-low duration.
3. OSC deglitch startup latency/first-cycle behavior at 60MHz.
4. clk_sys vs clk_axi_sys startup alignment causing DMA pulse loss.
5. transient EXCLK/OSC mux misselection through IO_DIS_EXCLK_Y skew.

The AXI read-response issue remains the immediate digital symptom, but the reset-duration selectivity suggests the upstream cause may be clock/reset/pad/analog startup rather than an address-map bug.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
