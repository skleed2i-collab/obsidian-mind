---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top Reset-Low Follow-up Execution Plan"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_reset_low_followup_execution_plan.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top reset-low 800us~1.3ms follow-up execution plan

Goal:
- reset-low 800us~1.3ms에서만 SYS_RST_VROM / VROM_UCS__9E / DMA EXE3 / AXI read-data wait가 발생하는 원인을 다음 측정에서 바로 좁힌다.
- RTL 수정 없이 waveform/RT readback/bring-up sequence 관찰만으로 analog/pad/clock-startup 원인과 DMA/AXI response 원인을 분리한다.

Current known symptom:
- sys_fsm       = SYS_RST_________VROM
- nvm_fsm       = NVM____VROM_LOAD_REQ
- vrom_ctl_fsm  = VROM_CTL_DMA_LV2
- vrom_lv2_fsm  = VROM_UCS__9E
- dma slv/exe   = SLV___EXE__ACT / EXE____3___ACT
- dma mst read  = RADR0_DONE / RDAT0_BURST_SRT
- direct wait   = VROM 9E DMA completion, i_vrom1_default_load_dma_end, is not returning.

Important interpretation:
- TO_RESX_SPROUT low turns OSC off.
- Therefore 800us~1.3ms reset-low duration is not normal synchronous RTL counter time.
- Because clk_sys and clk_axi_sys/clk_axi_sys_lp share the same root, do not explain reset-low dependency as independent clock-root skew.
- Treat reset-low dependency as IO pad discharge/threshold state plus OSC analog discharge/startup state. Reset-mask/gate/divider differences are downstream functional-availability effects, not the primary time-memory source.

Reference RTL evidence:
- source/rtl/peri/sys/sys_clk_osc_overshoot_protect.v
  - r_clk_i_osc_glitch_mask shifts on div8 after i_osc_clk_en.
  - full div1 deglitch output needs r_clk_i_osc_glitch_mask[8].
  - at 60MHz, div8 period is about 133.3ns, so 9 shifts are about 1.2us plus edge alignment.
- source/rtl/peri/sys/sys_reset.v
  - r_clk_mask_rstb_sof_cnt and r_clk_mask_rstb_slp_cnt are 4-bit counters clocked by clk_sys.
  - o_clk_mask_rstb remains 1 until either counter saturates.
  - downstream clock mask cannot clear before clk_sys is alive and the counters progress.
- source/rtl/peri/sys/sys_clk.v
  - clk_sys is produced before reset-mask-gated w_clk_sys_reset_mask.
  - clk_axi_sys is selected from w_clk_sys_div___1/2/4/8 after reset-mask gating.
  - clk_axi_sys_lp is gated by rm_axi_sys_lp_mode_off || i_clk_axi_sys_lp.
  - w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel; IO_DIS_EXCLK_Y skew can transiently select EXCLK.

Execution matrix:

| Case | Reset-low duration | Expected classification | Purpose |
|---|---:|---|---|
| C0 | no reset or previous-good baseline | good/reference | verify setup and readback sequence |
| C1 | short-good, below 800us | good | compare retained analog/pad state |
| C2 | 800us | fail boundary start | catch earliest failing signature |
| C3 | 950us | fail mid-window | primary failing waveform/RT snapshot |
| C4 | 1.1ms | fail mid-window | repeatability and margin |
| C5 | 1.3ms | fail boundary end | catch exit signature |
| C6 | long-good, above 1.3ms | good | compare full discharge/cold-like restart |

Required capture ordering per case:
1. Apply reset-low duration.
2. Capture analog/pad/clock startup from at least 10us before TO_RESX_SPROUT rising edge to VROM/DMA hang or done.
3. If waveform is limited, at minimum capture from TO_RESX_SPROUT rising edge to 50us after first TO_OSC_CLK edge.
4. Run the same RT readback sequence after the snapshot time:
   - FFh PCNT 11, 12, 13, 14, 15
   - D2h PCNT 13, 11
   - FFh PCNT 1
   - C4h PCNT 1, 2, 3
5. Record pass/fail, exact reset-low duration, temperature/voltage/board condition, and whether the case is reproducible.

Priority waveform groups:

P0. Reset/OSC first-clock group:
- TO_RESX_SPROUT
- TO_PORB_VDD
- IP_OSC_BIAS_EN
- IP_OSC_CLK_EN
- TO_OSC_CLK
- sys_clk_osc_overshoot_protect.r_clk_i_osc_glitch_mask[8:0]
- sys_clk_osc_overshoot_protect.w_clk_osc_deglitch1
- sys_clk_osc_overshoot_protect.w_clk_osc_deglitch2
- sys_clk_osc_overshoot_protect.o_clk_osc
- sys_clk.w_clk_oscdat_deglitch
- sys_clk.w_clk_m_oscsys
- sys_clk.clk_sys

P0. Reset-mask and AXI clock group:
- sys_reset.r_clk_mask_rstb_sof_cnt[3:0]
- sys_reset.r_clk_mask_rstb_slp_cnt[3:0]
- sys_reset.o_clk_mask_rstb
- sys_clk.w_sys_reset_mask
- sys_clk.w_clk_sys_reset_mask
- sys_clk.w_clk_sys_div___1
- sys_clk.clk_axi_sys
- sys_clk.clk_axi_sys_lp
- sys_clk.rm_axi_sys_lp_mode_off
- sys_clk.i_clk_axi_sys_lp

P0. VROM/DMA/AXI stop-point group:
- r_sys_fsm, r_nvm_fsm, r_vrom_ctl_fsm, r_vrom_lv2_fsm, r_vrom_lv3_fsm
- o_vrom1_default_load_dma_srt
- i_vrom1_default_load_dma_end
- i_cdc_exe3_srt, o_exe3_srt, i_exe3_end, o_exe3_done
- r_slv_fsm, r_exe_fsm
- r_radr_fsm, r_rdat_fsm, r_wadr_fsm, r_wdat_fsm, r_pop_fsm
- axi4_mst_dma.arvalid/arready/araddr/arlen
- axi4_mst_dma.rvalid/rready/rlast/rresp
- w_rdat_cnt_end, r_rdat_cnt, w_fifo_push, w_fifo_lp

P1. Pad/mode skew group:
- IO_DIS_EXCLK_Y, IO_EXCLK_Y
- sys_clk.rm_sys_exclk_sel
- sys_clk.w_sys_exclk_sel
- sys_clk.w_clk_ex
- sys_clk.w_clk_m_sys_mux
- IO_PNSW_Y, IO_DBIST_Y, IO_PMODE_Y, IO_TMODE_Y[*], IO_IM_Y[*]
- i_scan_mode / w_scan_mode if available

Pass/fail decision tree:
1. If failing case has no TO_OSC_CLK or abnormal TO_OSC_CLK first cycles compared with short-good/long-good:
   - classify as OSC analog restart / first-clock quality.
2. If TO_OSC_CLK is normal but w_clk_oscdat_deglitch or clk_sys starts late/irregular:
   - classify as OSC deglitch/overshoot-protect startup interaction.
3. If clk_sys starts normally but clk_axi_sys or clk_axi_sys_lp starts late/missing:
   - classify as reset-mask / AXI clock-gate sequencing.
4. If o_vrom1_default_load_dma_srt occurs before clk_axi_sys/clk_axi_sys_lp and reset-mask are stable:
   - classify as DMA start crossing too early / possible pulse loss or partial DMA initialization.
5. If VROM dma_srt and EXE3 start are seen, AR handshakes, but RVALID/RLAST never returns:
   - classify as AXI read target/source response missing, then check source target reset/clock/address decode.
6. If RVALID/RLAST returns but DMA end does not assert:
   - classify as rready/count_end/FIFO read-data completion mismatch.
7. If DMA end asserts but VROM remains 9E:
   - classify as DMA-end return CDC/connection back to VROM.
8. If IO_DIS_EXCLK_Y or scan/test mode differs only in failing window:
   - classify as pad/mode skew or partial-retention startup path.

Immediate next action for lab/simulation owner:
- Capture C1/C3/C6 first: short-good, mid-window-fail, long-good.
- Use the same probe list and same RT readback order.
- Do not start by expanding all scenarios; first prove which of these three comparisons diverges earliest:
  1. TO_OSC_CLK first edge/quality,
  2. clk_sys vs clk_axi_sys/clk_axi_sys_lp first valid edge,
  3. VROM dma_srt vs DMA/AXI readiness,
  4. AXI R channel response.

Expected deliverable after capture:
- A three-column comparison table: short-good / window-fail / long-good.
- Earliest divergent signal and timestamp.
- Final boundary classification from the decision tree above.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
