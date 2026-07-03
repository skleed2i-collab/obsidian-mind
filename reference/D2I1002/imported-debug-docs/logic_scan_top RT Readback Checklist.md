---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top RT Readback Checklist"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_rt_readback_checklist.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top RT readback checklist

Goal: 현재 u_rt로 들어가는 ro_* 신호 중 외부 read로 확인 가능한 값을 골라, waveform 없이도 reset/clock/VROM/DMA wait-chain을 좁힌다.

Assumption:
- u_rt로 입력되는 ro_*는 외부에서 RT command read로 확인 가능하다.
- 아래 mapping은 RTL 기준으로 확인했다.
- Read pipeline은 clk_axi_sys_lp에서 동작한다. 따라서 아래 read가 정상 응답하는 것 자체가 최소한 AXI/RT read path와 clk_axi_sys_lp가 어느 정도 살아있음을 의미한다.

Relevant RTL:
- source/rtl/top/logic_top.sv: u_rt instantiation, ro_* connections
- source/rtl/rt/rt_global/rt_ff.v: FF TEST command readback mux
- source/rtl/rt/rt_global/rt_d2.v: D2 system command readback mux
- source/rtl/rt/rt_global/rt_c4.v: C4 OSC_FUSE command readback mux

Notation:
- CMD = read command byte, for example FFh, D2h, C4h.
- PCNT = read data packet/count index used by i_rdat_pcnt. RTL uses r_en_pcnt[3:0] and sometimes r_en_pcnt[4].
- Byte0..Byte3 correspond to rdat0..rdat3.

---

## 1. Highest-priority reads

### 1.1 FFh, PCNT=11: SYS/NVM main state

CMD FFh, PCNT 11:
- Byte1 = {ro_pof_lat, 2'b0, ro_sys_fsm[4:0]}
- Byte2 = {4'b0, ro_sys_fsm_nvm_fsm[3:0]}
- Byte3 = {ro_sys_fsm_nvm_otp_fsm[4:0], ro_sys_fsm_nvm_vrom_ctl_fsm[2:0]}

Why read:
- Confirms whether top-level wait is still SYS_RST_VROM and NVM_VROM_LOAD_REQ.
- Confirms whether VROM control FSM is still DMA_LV2 or moved.

Expected current failing snapshot:
- Byte1[4:0] = 5'h04 = SYS_RST_________VROM
- Byte2[3:0] = 4'h1 = NVM____VROM_LOAD_REQ
- Byte3[2:0] = 3'h3 = VROM_CTL_DMA_LV2
- Byte3[7:3] = ro_sys_fsm_nvm_otp_fsm, currently expected 5'h00

Interpretation:
- If these values already moved, the original hang snapshot is stale or intermittent.
- If unchanged, VROM/DMA path remains the active wait point.

### 1.2 FFh, PCNT=12: VROM LV2/LV3 and MCU NVM state

CMD FFh, PCNT 12:
- Byte0 = ro_sys_fsm_nvm_vrom_lv2_fsm[7:0]
- Byte1 = ro_sys_fsm_nvm_vrom_lv3_fsm[7:0]
- Byte2 = {3'b0, ro_sys_fsm_nvm_mcu_fsm[4:0]}

Expected current failing snapshot:
- Byte0 = 8'h9E = VROM_UCS__9E
- Byte1 = 8'h00 = VROM_LV3_IDL
- Byte2[4:0] = 5'h10 = observed ro_sys_fsm_nvm_mcu_fsm

Interpretation:
- Byte0 stuck at 9E means first LV2 DMA command did not complete.
- Byte0 moved to 48/62/... means DMA end did occur at least once.
- Byte1 nonzero means LV3 phase started; then original issue is not first LV2 DMA.

### 1.3 FFh, PCNT=13: DMA slave and DMA read-side state

CMD FFh, PCNT 13:
- Byte1 = {ro_sys_fsm_nvm_fls_fsm[3:0], ro_dma_core_slv_slv_fsm[3:0]}
- Byte2 = {3'b0, ro_dma_core_slv_exe_fsm[4:0]}
- Byte3 = {ro_dma_core_mst_pop_fsm[1:0], ro_dma_core_mst_rdat_fsm[2:0], ro_dma_core_mst_radr_fsm[2:0]}

Expected current failing snapshot:
- Byte1[3:0] = 4'h6 = SLV___EXE__ACT
- Byte2[4:0] = 5'h0E = EXE____3___ACT
- Byte3[7:6] = 2'h0 = POP_IDLE
- Byte3[5:3] = 3'h1 = MST_RDAT0_BURST_SRT
- Byte3[2:0] = 3'h2 = MST_RADR0_DONE

Interpretation:
- If Byte2 remains 0E, DMA slave is waiting for i_exe3_end.
- If Byte3 says RADR_DONE/RDAT_BURST_SRT, AXI read data completion is still the closest digital wait point.
- If rdat/radr returned to idle but VROM still waits, suspect DMA end propagation back to VROM.

### 1.4 FFh, PCNT=14: DMA write-side state and pad group

CMD FFh, PCNT 14:
- Byte0 = {ro_dma_core_mst_wdat_fsm[3:0], ro_dma_core_mst_wadr_fsm[3:0]}
- Byte2 = {ro_pad_gpo_mon, ro_pad_pcd, ro_pad_err_fg, ro_pad_te, ro_pad_vlout3_en, ro_pad_pmic_en, ro_pad_elon_swire, ro_pad_avdd_swire}
- Byte3 = {ro_pad_rdx, ro_pad_dcx, ro_pad_scl_wrx, ro_pad_scn_csx, ro_pad_f_io[3:0]}

Expected current failing snapshot:
- Byte0[7:4] = 4'h0 = MST_WDAT_IDLE
- Byte0[3:0] = 4'h1 = MST_WADR_READY

Interpretation:
- WADR_READY + WDAT_IDLE means write side is waiting for FIFO/read data to arrive.
- If write side advanced, then read data did arrive and wait point moved.
- Byte2/Byte3 pad values help compare failing 800us~1.3ms case against normal reset cases.

### 1.5 FFh, PCNT=15: pad/mode/clock select readback

CMD FFh, PCNT 15:
- Byte0 = {ro_pad_dbist, ro_pad_pnsw, ro_pad_dis_exclk, ro_pad_exclk, ro_pad_line_fg, ro_pad_tsp_sync3, ro_pad_tsp_sync2, ro_pad_tsp_sync1}
- Byte1 = {ro_pad_pmode, ro_pad_lane_sel[1:0], ro_pad_lnsw[1:0], ro_pad_tin[2:0]}
- Byte2 = {5'b0, ro_pad_im[2:0]}
- Byte3 = {3'b0, ro_vpr_mx_m2s_overflow, 1'b0, ro_vpr_mx_m2s_level[2:0]}

Why read:
- Directly checks pad-derived values visible to RT.
- Particularly important for reset-low 800us~1.3ms condition because *_Y skew/partial retention is a high-priority candidate.

Important bits:
- Byte0[5] = ro_pad_dis_exclk
  - Expected 1 if OSC path should be selected by i_dis_exclk.
- Byte0[4] = ro_pad_exclk
  - Tells whether EXCLK pad is high/low at read time, not whether it toggled.
- Byte0[6] = ro_pad_pnsw
  - Scan reset related.
- Byte0[7] = ro_pad_dbist
  - Test/DBIST related.
- Byte1[7] = ro_pad_pmode
- Byte1[6:5] = ro_pad_lane_sel[1:0]
- Byte1[4:3] = ro_pad_lnsw[1:0]
- Byte1[2:0] = ro_pad_tin[2:0]
- Byte2[2:0] = ro_pad_im[2:0]
  - IO_IM[1] affects i_pad_otp_skip via ~IO_IM_Y[1].

Interpretation:
- If failing case has different ro_pad_dis_exclk than normal case, clock source selection is a strong suspect.
- If scan/test pads differ in failing window, scan/test transient or partial-retention is likely.
- These are sampled readback values; they do not reveal ns-scale skew directly, but they show final/static pad state after read.

---

## 2. Clock/system status reads

### 2.1 D2h, PCNT=13: system done/osc status flags

CMD D2h, PCNT 13:
- Byte3 = {ro_pwr__on_done, ro_pwr_off_done, 1'b0, ro_otp_load_done, ro_flash_load_done, ro_sleep_out_reg, ro_sleep__in_reg, ro_sys_osc_on}

Important bits:
- Byte3[0] = ro_sys_osc_on
- Byte3[1] = ro_sleep__in_reg
- Byte3[2] = ro_sleep_out_reg
- Byte3[3] = ro_flash_load_done
- Byte3[4] = ro_otp_load_done
- Byte3[6] = ro_pwr_off_done
- Byte3[7] = ro_pwr__on_done

Why read:
- ro_sys_osc_on tells whether sys_fsm believes OSC should be on.
- ro_pwr__on_done / ro_otp_load_done / ro_flash_load_done help identify if power/NVM sequence is blocked before or after VROM/DMA.

Interpretation:
- ro_sys_osc_on = 0 while expecting OSC path: check sys_fsm oscillator enable path.
- ro_sys_osc_on = 1 but TO_OSC_CLK/clk_sys absent in waveform: analog OSC output or clock path issue.

### 2.2 C4h, PCNT=2 and PCNT=3: OSC trim/count readback

CMD C4h, PCNT 2:
- Byte1 = {7'b0, ro_osc1_trim_done}
- Byte2 = ro_osc1_trimed_fctrl[7:0]
- Byte3 = ro_osc1_cnt[23:16]

CMD C4h, PCNT 3:
- Byte0 = ro_osc1_cnt[15:8]
- Byte1 = ro_osc1_cnt[7:0]

Why read:
- ro_osc1_cnt may help compare oscillator measurement/counting behavior between normal and failing reset durations.
- ro_osc1_trim_done / ro_osc1_trimed_fctrl may reveal OSC trim/measurement path status.

Interpretation:
- If ro_osc1_cnt is zero/stale only in failing case, OSC measurement/clock path likely did not run correctly.
- If ro_osc1_cnt differs significantly between normal and 800us~1.3ms reset-low case, first-clock/frequency/trim behavior may be involved.

### 2.3 C4h, PCNT=1: OSC control configuration readback

CMD C4h, PCNT 1:
- Byte0 = {rm_osc_trmf_fast_en, 3'b0, rm_osc_lpf_en, rm_osc_t_coeff[2:0]}
- Byte1 = {rm_oscreg_en, rm_osc_itcref_test_en, rm_oscreg_mon_en, 2'b0, rm_osc_force_en, rm_osc_test_en, rm_osc_trim_start}
- Byte3 = rm_osc_trim_ref[15:8]

Why read:
- Confirms whether OSC force/test/trim modes differ between failing and normal cases.
- Not necessarily the root cause, but useful to avoid chasing a mode/config issue.

### 2.4 D2h, PCNT=11: clock configuration bits

CMD D2h, PCNT 11:
- Byte0 = {rm_mcu_im_chksum_on, rm_mcu_im_chksum_err_cnt[1:0], rm_dbv_change_detect_on, rm_osc_wakeup_mode, rm_clk_pmic_sel, rm_mcu_normal_start_sel, rm_mcu_normal_on}
- Byte1 = {rm_clk_pwr_div[1:0], rm_pon_wait_sel[1:0], rm_vpr_sdc_vbp_clk_ctl_off, rm_axi_vprsram_lp_mode_off, rm_axi_sys_lp_mode_off, rm_mcu_mem_pwr_auto_off}
- Byte2 = {rm_clk_dat_div[1:0], rm_clk_axi_sys_div[1:0], rm_clk_osc_mipi_div[1:0], rm_vpr_sdc_clk_ctl_off, rm_vpr_mx_clk_ctl_off}
- Byte3 = {rm_clk_dat_div_man_on, rm_vin_clk_ctl_off, rm_clk_axi_regflag_sel, rm_axi_regflag_lp_mode_off, rm_vin_clk_off_vfp_cnt[3:0]}

Important bits:
- Byte1[1] = rm_axi_sys_lp_mode_off
  - clk_axi_sys_lp gate condition includes rm_axi_sys_lp_mode_off || i_clk_axi_sys_lp.
- Byte2[5:4] = rm_clk_axi_sys_div[1:0]
  - affects clk_axi_sys divider selection.
- Byte0[3] = rm_osc_wakeup_mode

Why read:
- Confirms the clock/LP mode configuration used in the failing case.
- If clk_axi_sys_lp is suspected, Byte1[1] is especially useful.

### 2.5 FFh, PCNT=1: clock source select config

CMD FFh, PCNT 1:
- Byte0 = {6'b0, rm_sys_exclk_sel, rm_dat_exclk_sel}

Why read:
- sys_clk uses:
  - w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel
- Even if ro_pad_dis_exclk is correct, rm_sys_exclk_sel can invert/select differently.

Interpretation:
- Compare rm_sys_exclk_sel and ro_pad_dis_exclk together.
- If ro_pad_dis_exclk=1 and rm_sys_exclk_sel=0, expected functional comment says OSC path should be selected.
- If rm_sys_exclk_sel differs from expected, EXCLK/OSC source selection candidate rises.

---

## 3. Recommended read sequence for failing vs normal comparison

For each reset-low condition:
- normal short reset
- failing 800us~1.3ms reset
- normal long reset

Read the following in this order after the system reaches the suspect state/time:

1. FFh PCNT 11
   - SYS/NVM/VROM control state
2. FFh PCNT 12
   - VROM LV2/LV3 state
3. FFh PCNT 13
   - DMA slave + read-side master FSM
4. FFh PCNT 14
   - DMA write-side FSM + pad group
5. FFh PCNT 15
   - pad/mode/clock-select readback
6. D2h PCNT 13
   - ro_sys_osc_on and system done flags
7. D2h PCNT 11
   - clk LP/div/mode config
8. FFh PCNT 1
   - rm_sys_exclk_sel / rm_dat_exclk_sel
9. C4h PCNT 1
   - OSC mode/config
10. C4h PCNT 2
   - trim done/fctrl and ro_osc1_cnt[23:16]
11. C4h PCNT 3
   - ro_osc1_cnt[15:0]

If only a few reads are possible, minimum set:
- FFh PCNT 11
- FFh PCNT 12
- FFh PCNT 13
- FFh PCNT 15
- D2h PCNT 13
- C4h PCNT 2/3

---

## 4. Decode table for current expected failing state

If current observed state is reproduced, the readback should show approximately:

FFh PCNT 11:
- Byte1[4:0] = 5'h04
- Byte2[3:0] = 4'h1
- Byte3[2:0] = 3'h3

FFh PCNT 12:
- Byte0 = 8'h9E
- Byte1 = 8'h00
- Byte2[4:0] = 5'h10

FFh PCNT 13:
- Byte1[3:0] = 4'h6
- Byte2[4:0] = 5'h0E
- Byte3[7:6] = 2'h0
- Byte3[5:3] = 3'h1
- Byte3[2:0] = 3'h2

FFh PCNT 14:
- Byte0[7:4] = 4'h0
- Byte0[3:0] = 4'h1

D2h PCNT 13:
- Byte3[0] = ro_sys_osc_on; should be checked against expectation.

C4h PCNT 2/3:
- ro_osc1_cnt should be compared across reset-low durations rather than decoded to a single expected constant.

---

## 5. What readback cannot prove alone

The following cannot be fully determined from current ro_* readbacks:

1. Actual TO_OSC_CLK first-edge timing.
2. Clock jitter/runt/missing pulse on TO_OSC_CLK.
3. Exact clk_sys vs clk_axi_sys first-edge alignment.
4. Whether o_vrom1_default_load_dma_srt pulse occurred before clk_axi_sys became alive.
5. AXI arvalid/arready/rvalid/rready/rlast handshakes.
6. i_vrom1_default_load_dma_end and i_exe3_end pulses if they are too narrow and not latched in readable ro_*.
7. ns-scale pad skew at reset release.

For those, waveform/probe is still required.

---

## 6. Practical interpretation guide

### Case A: FF state still equals current failing snapshot

If FF PCNT 11/12/13/14 match the current failing state exactly:
- Digital wait chain is stable and reproducible.
- Next focus: AXI read response / clk_axi_sys / clk_axi_sys_lp / VROM DMA start.

### Case B: FF VROM LV2 moved beyond 9E

If FF PCNT 12 Byte0 is not 8'h9E:
- First VROM DMA completed at least once.
- Check where it later stopped.
- Compare new LV2 command against DMA state.

### Case C: DMA states idle but VROM still at 9E

If DMA read/write/slave FSMs are idle but VROM LV2 remains 9E:
- Suspect DMA end propagation/CDC/connection back to VROM.

### Case D: ro_pad_dis_exclk or scan/test pads differ by reset-low duration

If FF PCNT 15 differs between normal and 800us~1.3ms failing reset:
- Strong evidence for pad skew/retention/mode path issue.
- Especially ro_pad_dis_exclk, ro_pad_pnsw, ro_pad_dbist, ro_pad_pmode, ro_pad_tin, ro_pad_im.

### Case E: ro_sys_osc_on differs by reset-low duration

If D2 PCNT 13 Byte3[0] differs:
- sys_fsm/OSC request path differs.
- If ro_sys_osc_on=0 in fail, OSC may not be requested by digital logic.
- If ro_sys_osc_on=1 in fail, but clock not present in waveform, analog/clock path is suspect.

### Case F: ro_osc1_cnt differs significantly or remains stale

If C4 PCNT 2/3 ro_osc1_cnt is zero/stale/different only for 800us~1.3ms:
- OSC measurement/clock behavior differs by reset-low window.
- Supports OSC restart/deglitch/startup hypothesis.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
