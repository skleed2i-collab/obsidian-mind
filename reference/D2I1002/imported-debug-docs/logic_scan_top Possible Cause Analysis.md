---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top Possible Cause Analysis"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_possible_cause_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top possible-cause analysis

Goal: 현재 logic_scan_top bring-up에서 SYS_RST_VROM / VROM_UCS__9E / DMA read-data wait 상태가 발생한 원인 후보를 clock jitter, OSC on-time delay, pad input skew, reset/clock-gate sequencing, AXI/DMA handshake 관점으로 정리한다.

Scope:
- Top: logic_scan_top
- Main state chain:
  - r_sys_fsm = 5'h04 = SYS_RST_________VROM
  - r_nvm_fsm = 4'h1 = NVM____VROM_LOAD_REQ
  - r_vrom_ctrl_fsm = 3'h3 = VROM_CTL_DMA_LV2
  - r_vrom_lv2_fsm = 8'h9E = VROM_UCS__9E
  - r_vrom_lv3_fsm = 8'h00 = VROM_LV3_IDL
  - r_slv_fsm = 3'h6 = SLV___EXE__ACT
  - r_exe_fsm = 5'h0E = EXE____3___ACT
  - r_radr_fsm = 3'h2 = MST_RADR0________DONE
  - r_rdat_fsm = 3'h1 = MST_RDAT0_BURST_SRT
  - r_wadr_fsm = 4'h1 = MST_WADR________READY
  - r_wdat_fsm = 4'h0 = MST_WDAT___________IDLE

Immediate wait point:
- sys_fsm_nvm_vrom.v: r_vrom_lv2_fsm = VROM_UCS__9E
- next condition: i_vrom1_default_load_dma_end == 1
- downstream DMA state says read address was issued, read data burst completion is still pending.

---

## 1. Clock/reset topology extracted from RTL

### 1.1 Top-level clock/reset pins

logic_scan_top.v:
- TO_RESX_SPROUT: pad reset input
- TO_PORB_VDD: POR reset input
- TO_OSC_CLK: OSC clock input to logic_top/peri/sys_clk
- IP_OSC_BIAS_EN: OSC bias enable output
- IP_OSC_CLK_EN: OSC clock enable output
- IO_EXCLK_Y: external clock input after pad path
- IO_DIS_EXCLK_Y: clock select, comment says 1: OSC, 0: EXCLK

logic_top.sv connects:
- TO_RESX_SPROUT -> peri.i_pad_resx
- TO_PORB_VDD -> peri.i_por_reset
- TO_OSC_CLK -> peri.i_osc_dat
- IO_EXCLK_Y -> peri.i_exclk
- IO_DIS_EXCLK_Y -> peri.i_dis_exclk
- IP_OSC_CLK_EN -> peri.i_osc_clk_en

### 1.2 Reset generation

sys_reset.v:
- w_rstb_pad = buffered i_pad_resx
- w_rstb_por = buffered i_por_reset
- w_rstb_pwr = w_rstb_pad & w_rstb_por
- w_rstb_sof = w_rstb_pwr & rc_swreset & i_pcd_ccd_soft_rstb & i_dsi_phy_rstb
- w_rstb_bus = w_rstb_sof & ~i_bus_reset
- rstb_slp = w_rstb_bus & (i_sleep_reset || rm_man_sleep_reset_on==8'h5a)
- rstb_osc_wakeup = w_rstb_pwr & i_osc_wakeup_reset

Clock mask:
- r_clk_mask_rstb_sof_cnt counts on clk_sys after rstb_sof release.
- r_clk_mask_rstb_slp_cnt counts on clk_sys after rstb_slp release.
- o_clk_mask_rstb stays 1 until either 4-bit counter saturates, then becomes 0.

Interpretation:
- clk_sys itself must be present for reset mask counters to progress.
- Several downstream clocks use the reset mask, so there can be an intentional post-reset clock-mask delay.

### 1.3 System clock generation

sys_clk.v:
- i_osc_dat enters sys_clk_osc_overshoot_protect.
- output w_clk_oscdat_deglitch feeds system OSC clock path.
- scan bypass can bypass deglitch only in scan mode.
- w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel
- comment: i_dis_exclk = 1 means OSC, 0 means EXCLK.
- u_mux_sys_exclk selects between OSC path and EXCLK path.
- clk_sys is buffered from w_clk_m_sys.

System clock mask/divider:
- u_cdc_clk_sys_reset_mask samples i_clk_mask_rstb into w_clk_m_sys domain.
- u_icg_clk_sys gates w_clk_m_sys into w_clk_sys_reset_mask when !w_sys_reset_mask.
- w_clk_sys_div___1 starts from w_clk_sys_reset_mask.
- clk_axi_sys is muxed from w_clk_sys_div___1/2/4/8 based on rm_clk_axi_sys_div.

Interpretation:
- sys_fsm runs on clk_sys, which is before the reset-mask divider path.
- DMA and AXI blocks run on clk_axi_sys / clk_axi_sys_lp, which may start later than clk_sys due to reset mask and divider/mux behavior.

### 1.4 OSC enable generation

osc_cont_top.v:
- IP_OSC_BIAS_EN = rstb
- IP_OSC_CLK_EN = rstb & (ro_sys_osc_on | i_dsi_osc_on | i_pad_osc_en | rm_osc_force_en | rm_osc_trim_start | rm_osc_test_en)
- clk_rstb = i_scan_mode ? rstb : rstb & IP_OSC_CLK_EN
- r_clk_off2on is reset while clk_rstb low, then clears on clk_osc_root edge after clock is present.

User-provided analog behavior:
- TO_RESX_SPROUT low -> OSC off, TO_OSC_CLK family clock absent.
- OSC_BIAS_EN / OSC_CLK_EN assert 후 실제 clock output까지 3~5us delay.

### 1.5 OSC deglitch/overshoot protect latency

sys_clk_osc_overshoot_protect.v:
- i_osc_clk -> div1/div2/div4/div8 chain.
- r_clk_i_osc_glitch_mask shifts on div8 clock when i_osc_clk_en is high and no fq_sel_change mask.
- o_clk_osc is produced by OR of:
  - gated div1 when r_clk_i_osc_glitch_mask[8] == 1
  - gated div2 during earlier mask states 1,3,7,15,31,63,127

Implication:
- Even after TO_OSC_CLK starts toggling, deglitch output is intentionally masked/conditioned for several OSC cycles.
- Approximate digital-only added latency if OSC is 100MHz:
  - div8 period approx 80ns
  - 9 shift positions approx up to 720ns plus first-edge alignment
  - some div2 pulses may appear during intermediate mask states, but full div1 output waits until mask[8]
- This is in addition to analog OSC on-time delay of 3~5us.

---

## 2. Ranked possible-cause list

### A. Clock/OSC bring-up related

A1. OSC on-time delay not accounted for
- Description: IP_OSC_BIAS_EN/IP_OSC_CLK_EN assert 후 3~5us 동안 TO_OSC_CLK가 실제로 없음.
- Effect: clk_sys/clk_axi_sys may not toggle, FSM appears stuck but is normal wait.
- Evidence to check:
  - IP_OSC_BIAS_EN rise time
  - IP_OSC_CLK_EN rise time
  - TO_OSC_CLK first toggle time
  - clk_sys first toggle time
  - clk_axi_sys first toggle time
- Pass/fail:
  - If snapshot is before TO_OSC_CLK + deglitch + reset-mask completion, not a real hang.

A2. OSC deglitch/overshoot protect added delay
- Description: sys_clk_osc_overshoot_protect masks OSC output until internal div8 mask progresses.
- Effect: TO_OSC_CLK may toggle but w_clk_oscdat_deglitch / clk_sys may still be delayed or initially divided/gated.
- Evidence:
  - TO_OSC_CLK vs w_clk_oscdat_deglitch
  - r_clk_i_osc_glitch_mask
  - i_osc_clk_en
  - i_fq_sel_change
- Risk level: high for early bring-up timing interpretation; medium for permanent hang.

A3. OSC clock jitter/unstable first cycles
- Description: first 3~5us after enable may include irregular period, runt pulses, duty distortion, or frequency settling.
- Effect:
  - deglitch protection may suppress early clocks.
  - clock mux/divider may see unexpected first edges.
  - counters based on clk_sys may start later than expected.
- Evidence:
  - period histogram of TO_OSC_CLK first 100~1000 cycles
  - minimum high/low pulse width
  - any runt pulse around enable
  - compare TO_OSC_CLK and clk_sys edge counts.
- Note: pure RTL sim normally does not model analog jitter unless the testbench/VCD source explicitly models it.

A4. Clock source selection skew on IO_DIS_EXCLK_Y
- Description: IO_DIS_EXCLK_Y selects OSC vs EXCLK. User said *_Y inputs settle with several-ns skew after TO_RESX_SPROUT.
- Effect:
  - at reset release, i_dis_exclk may temporarily be 0, selecting EXCLK path.
  - if EXCLK absent or x, clk_sys may be absent/x until select changes to OSC.
  - switching mux source around reset release can generate missed/glitchy first system clocks.
- Evidence:
  - IO_DIS_EXCLK_Y transition relative to TO_RESX_SPROUT
  - w_sys_exclk_sel
  - w_clk_m_oscsys, w_clk_ex, w_clk_m_sys_mux, clk_sys
- Risk level: high if EXCLK is not toggling and IO_DIS_EXCLK_Y is delayed.

A5. External clock path selected accidentally
- Description: w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel. If rm_sys_exclk_sel is not default as expected, clock may select EXCLK even when IO_DIS_EXCLK_Y=1.
- Effect: clk_sys may depend on IO_EXCLK_Y rather than OSC.
- Evidence:
  - rm_sys_exclk_sel
  - i_dis_exclk / IO_DIS_EXCLK_Y
  - w_sys_exclk_sel
  - IO_EXCLK_Y toggling 여부

A6. clk_axi_sys delayed relative to clk_sys by reset-mask
- Description: sys_fsm runs on clk_sys, but DMA runs on clk_axi_sys. clk_axi_sys starts from w_clk_sys_reset_mask and may be intentionally delayed by sys_reset counters.
- Effect:
  - sys_fsm/VROM can request DMA before clk_axi_sys/DMA is fully alive, depending on sequencing.
  - If request is a pulse crossing to a not-yet-running domain and no level/CDC hold exists, it can be missed.
- Evidence:
  - clk_sys first toggle
  - rstb_sof, rstb_bus
  - o_clk_mask_rstb / w_clk_mask_rstb
  - w_sys_reset_mask
  - w_clk_sys_reset_mask
  - clk_axi_sys first toggle
  - VROM dma_srt pulse width and whether clk_axi_sys sees it
- Risk level: high because current state crosses clk_sys VROM control and clk_axi_sys DMA control.

A7. clk_axi_sys_lp gate not enabled
- Description: dma_core_slv has logic on clk_axi_sys and clk_axi_sys_lp. clk_axi_sys_lp is gated by rm_axi_sys_lp_mode_off || i_clk_axi_sys_lp.
- Effect:
  - some slave ready/valid registers or address latch using clk_axi_sys_lp may not update.
  - slv/exe FSM on clk_axi_sys can be active while LP-clocked signals are stale.
- Evidence:
  - clk_axi_sys_lp toggling
  - rm_axi_sys_lp_mode_off
  - i_clk_axi_sys_lp
  - r_axi4_slv_awready/wready/bvalid behavior
- Risk level: medium/high for DMA slave anomalies.

A8. Frequency select change masks OSC deglitch output
- Description: i_fq_sel_change causes w_fq_sel_change_mask_on and resets/masks deglitch progression.
- Effect: output clock may be suppressed or restarted during bring-up.
- Evidence:
  - i_fq_sel_change
  - r_fq_sel_change
  - w_deglitch_mask_reset
  - r_clk_i_osc_glitch_mask

A9. Reset deassertion too close to first clock edge
- Description: TO_RESX_SPROUT and derived rstb_pwr release may occur close to first OSC/clk_sys edge after analog on-delay.
- Effect: asynchronous reset release can produce different initial cycles across domains; CDC sync registers may require extra cycles.
- Evidence:
  - rstb_pwr/rstb_bus deassertion time vs first clk_sys/clk_axi_sys edges
  - any X propagation on counters/FSMs
- Risk: medium; normal digital design should tolerate async reset release if synchronizers exist, but clock mux/divider/gate first edges are sensitive.

### B. Pad input skew / reset-sequence related

B1. *_Y input skew after TO_RESX_SPROUT
- Description: init.vcd shows same timestamp, but actual pad path has several-ns skew for *_Y signals.
- Effect:
  - reset/scan/clock select/test-mode inputs can be temporarily wrong.
  - IO_PNSW_Y, IO_DIS_EXCLK_Y, IO_TMODE_Y, IO_IM_Y, IO_TIN_Y can affect reset/scan/clock/test path.
- Evidence:
  - individual *_Y transition times in full waveform, not reduced init.vcd.
- Most important *_Y for this case:
  - IO_DIS_EXCLK_Y: OSC vs EXCLK select
  - IO_EXCLK_Y: external clock data path
  - IO_PNSW_Y: scan reset input
  - IO_TMODE_Y[*], IO_DBIST_Y, IO_PMODE_Y: test/scan mode related
  - IO_IM_Y[1]: i_pad_otp_skip = ~IO_IM_Y[1]
  - IO_PTO_Y: dbi csx reset input

B2. scan/test mode transient assertion
- Description: IO_TMODE_Y, IO_DBIST_Y, IO_PMODE_Y, IO_PNSW_Y transition around reset release.
- Effect:
  - scan mux may briefly select scan reset/clock.
  - reset tree may use i_pad_scan_reset instead of functional reset if scan mode transiently high.
- Evidence:
  - w_scan_mode
  - i_scan_mode
  - rstb_* through scan mux outputs
  - clk_scan selection

B3. OTP skip transient value
- Description: logic_top connects i_pad_otp_skip = ~IO_IM_Y[1]. If IO_IM_Y[1] settles late, skip may be temporarily high.
- Effect:
  - NVM reset flow could branch around OTP path differently. Current state is VROM path, so not the immediate wait point, but can affect later flow.
- Evidence:
  - IO_IM_Y[1]
  - i_pad_otp_skip
  - nvm_fsm branch after VROM done.

### C. VROM/DMA/AXI handshake related

C1. VROM 9E DMA start missed by DMA domain
- Description: VROM generates o_vrom1_default_load_dma_srt in clk_sys domain; DMA slave/master action is in clk_axi_sys domain through CDC/wrappers.
- Effect: VROM waits for DMA end forever if start pulse is missed or not converted to level.
- Evidence:
  - o_vrom1_default_load_dma_srt
  - i_cdc_exe3_srt / o_exe3_srt
  - clk_sys vs clk_axi_sys timing
- Risk: high if clk_axi_sys starts later than clk_sys.

C2. DMA EXE3 target never returns i_exe3_end
- Current state says r_exe_fsm = EXE____3___ACT.
- Exit condition is i_exe3_end == 1.
- Evidence:
  - o_exe3_srt pulse
  - i_exe3_end
  - target block FSM for exe3
- Risk: high; direct wait point in dma_core_slv.

C3. AXI read address issued but read data never returns
- Current state says r_radr_fsm = MST_RADR0_DONE and r_rdat_fsm = MST_RDAT0_BURST_SRT.
- Completion requires rvalid & rlast & rready & count_end.
- Evidence:
  - arvalid/arready/araddr/arlen
  - rvalid/rready/rlast/rresp
  - read target decode for VROM source address
- Risk: high; this matches current DMA master state.

C4. Read target address/decode wrong for CMD_9E VROM source
- Description: VROM 9E uses CMD_9E_VROM_START_ADR and MMAP_SYS_VROM1 upper address.
- Effect: AXI target may not respond, or respond with error/never rvalid.
- Evidence:
  - o_vrom1_default_load_dma_src_adr
  - axi4_mst_dma.araddr
  - interconnect decode target
  - target slave arready/rvalid

C5. rready deasserted due to FIFO/backpressure
- Description: rready depends on FIFO level: r_axi4_mst_rready_ <= (w_fifo_lp <= FIFO_SIZE-8).
- Effect: rvalid may be present but transfer stalls if FIFO full or FIFO level wrong.
- Evidence:
  - w_fifo_lp
  - r_axi4_mst_rready_
  - w_fifo_push / w_fifo_pop
  - r_wadr_fsm / r_wdat_fsm

C6. write side waits for FIFO data
- Current state r_wadr_fsm=MST_WADR_READY waits for |w_fifo_lp.
- If read data never pushes FIFO, write side stays idle/ready.
- Evidence:
  - w_fifo_push never toggles
  - w_fifo_lp stays 0
- This is likely a symptom of read data not arriving, not root cause.

C7. AXI write response missing after read side eventually completes
- Not current first suspect because write data FSM is still IDLE, but check after read issue is resolved.
- Evidence:
  - awvalid/awready
  - wvalid/wready/wlast
  - bvalid/bready

---

## 3. Recommended waveform checks in order

### Step 1: classify snapshot time against clock availability

Measure:
- t_resx_rise = TO_RESX_SPROUT 0->1
- t_bias_en = IP_OSC_BIAS_EN 0->1
- t_clk_en = IP_OSC_CLK_EN 0->1
- t_osc_first = first TO_OSC_CLK toggle
- t_clk_sys_first = first clk_sys toggle
- t_clk_axi_first = first clk_axi_sys toggle
- t_snapshot = time where FSM values were captured

Decision:
- if t_snapshot < t_osc_first: normal clock-off wait
- if t_snapshot < t_clk_sys_first: OSC/deglitch/clock-mux wait
- if t_snapshot < t_clk_axi_first: DMA/AXI domain not alive yet
- if t_snapshot is much later than t_clk_axi_first: real handshake hang likely

### Step 2: check clock quality first 1000 cycles

For TO_OSC_CLK and clk_sys:
- min period
- max period
- average period
- high pulse min/max
- low pulse min/max
- edge count between enable and VROM/DMA start

Red flags:
- missing pulses after enable
- runt pulses
- long off gap after IP_OSC_CLK_EN
- clock source switch from EXCLK to OSC during reset release

### Step 3: check reset-mask and domain alignment

Signals:
- rstb_pwr
- rstb_sof
- rstb_bus
- rstb_slp
- i_clk_mask_rstb / w_clk_mask_rstb if accessible
- w_sys_reset_mask
- w_clk_sys_reset_mask
- clk_sys
- clk_axi_sys
- clk_axi_sys_lp

Red flags:
- sys_fsm changes state before clk_axi_sys is toggling
- VROM dma_srt pulse occurs before clk_axi_sys/clk_axi_sys_lp alive
- rstb_bus deasserted while clk_axi_sys not yet stable

### Step 4: check VROM 9E DMA start and end

Signals:
- r_vrom_ctl_fsm
- r_vrom_lv2_fsm
- o_vrom1_default_load_dma_srt
- o_vrom1_default_load_dma_src_adr
- o_vrom1_default_load_dma_des_adr
- o_vrom1_default_load_dma_trn_siz
- i_vrom1_default_load_dma_end

Red flags:
- dma_srt missing
- dma_srt one clk_sys pulse only and not captured by DMA side
- DMA end missing forever

### Step 5: check DMA/AXI read completion

Signals:
- r_slv_fsm
- r_exe_fsm
- o_exe3_srt
- i_exe3_end
- r_radr_fsm
- r_rdat_fsm
- axi4_mst_dma.arvalid/arready/araddr/arlen
- axi4_mst_dma.rvalid/rready/rlast/rresp
- r_rdat_cnt
- w_rdat_cnt_end
- w_rdat_end
- w_fifo_lp/w_fifo_push

Red flags:
- arvalid with no arready: interconnect/target not accepting address
- ar handshake done, no rvalid: target/read decode issue
- rvalid but no rready: FIFO/backpressure issue
- rvalid/rready but no rlast: burst completion issue
- rlast but w_rdat_cnt_end false: arlen/count mismatch

---

## 4. Current priority list

Priority 1:
- Confirm actual clock timeline: IP_OSC_BIAS_EN, IP_OSC_CLK_EN, TO_OSC_CLK, clk_sys, clk_axi_sys, clk_axi_sys_lp.

Priority 2:
- Confirm whether VROM dma_srt happened before clk_axi_sys/clk_axi_sys_lp became alive.

Priority 3:
- Confirm AXI read channel completion for CMD_9E source address.

Priority 4:
- Confirm pad skew did not transiently select EXCLK or scan/test mode at reset release.

Priority 5:
- If all above are clean, inspect AXI target decode/memory model for VROM1 address response.

---

## 5. Most likely candidates right now

Based on current FSM snapshot alone, without full waveform:

1. AXI read data response missing for VROM 9E DMA
   - strongest match to r_radr_fsm=MST_RADR0_DONE and r_rdat_fsm=MST_RDAT0_BURST_SRT.

2. clk_axi_sys/clk_axi_sys_lp not alive or delayed when DMA start was issued
   - plausible because sys_fsm/VROM are clk_sys side and DMA is clk_axi_sys side.

3. OSC on-time + deglitch + reset-mask delay misinterpreted as hang
   - must be excluded first because TO_RESX low means no OSC, and OSC output starts 3~5us after enable.

4. IO_DIS_EXCLK_Y skew caused transient clock-source misselection
   - plausible because clock source select is pad-derived and *_Y settle is not simultaneous.

5. Clock jitter/unstable first OSC cycles caused deglitch/mux/divider startup delay
   - plausible for timing interpretation; less likely as a permanent hang unless the testbench/analog model produces runt/missing pulses or the deglitch mask is repeatedly reset.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
