---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top Debug README"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/README_logic_scan_top_debug.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top debug documents index

This folder contains the current debug package for logic_scan_top SYS_RST_VROM / VROM 9E / DMA wait investigation.

## Documents

1. logic_scan_top_fsm_debug_plan.md
   - Main investigation plan.
   - Contains FSM decode, wait-chain, required waveform signals, and step-by-step debug flow.

2. logic_scan_top_possible_cause_analysis.md
   - Broad possible-cause analysis.
   - Covers OSC on-time delay, clock jitter, deglitch, reset/clock gate sequencing, pad skew, VROM/DMA/AXI handshake candidates.

3. logic_scan_top_top10_candidates_reset_800us_1p3ms.md
   - Refined top-10 candidate list for the condition:
     - OSC about 60MHz
     - issue occurs only when reset-low duration is 800us ~ 1.3ms.

4. logic_scan_top_rt_readback_checklist.md
   - RT readable ro_* register checklist.
   - Lists which external reads are most useful to narrow the issue without waveform access.

5. logic_scan_top_reset_low_followup_execution_plan.md
   - Follow-up execution plan for reset-low 800us~1.3ms only.
   - Defines short-good / window-fail / long-good comparison matrix, required waveform groups, RT readback order, and decision tree.

6. logic_scan_top_reset_low_io_osc_discharge_hypothesis.md
   - Refined hypothesis after clarifying clk_sys and clk_axi_sys/clk_axi_sys_lp share the same root.
   - Treats reset-low dependency as IO pad discharge plus OSC analog discharge/restore timing, especially IO_DIS_EXCLK_Y and OSC first-clock interaction, not same-root clock skew.

7. init_vcd_timing_summary.txt
   - Parsed summary of init.vcd.
   - Shows TO_RESX_SPROUT and *_Y initial/transition timing in the reduced VCD.

8. logic_scan_top_vrom_read_clk_vout_s1_domain_finding.md
   - RTL trace result of the actual VROM 9E read path.
   - Key finding: the read target (otp_vrom_slv_top_new) runs on clk_vout_s1 (dat-path branch),
     crossed from clk_axi_sys via an async DW_axi_a2x bridge.
   - Proves clk_vout_s1 was alive up to the SLPOUT write (BRESP_MODE=1), narrowing the hang to
     dat-clock death/glitch after the write, or a one-shot pulse-loss deadlock inside the slave.
   - MEASURED: D0h PCNT13 read all-zero -> shown to be an RT mirror register (rm_d0_54/55,
     command-write-only, default 0), NOT live slave status -> result inconclusive by construction.
     Live status exists only at OTP slave reg 0x017 (AXI 0x4401_005C) through the same
     possibly-dead domain (observability gap, ECO candidate).
   - Replacement no-waveform discriminator: in-band experiments E1 (rm_clk_dat_div_man_on /
     rm_clk_dat_div manual switch) and E2 (rm_dat_exclk_sel toggle) during the hang.
     Hang released by E1/E2 -> dat-clock stop (path identified); not released -> slave-internal
     pulse-loss deadlock.
   - VERIFIED: TO_OSC_CLK is wired raw to the clock root with no toggle stage
     (logic_top.sv:4276 direct; deglitch full-rate path gates the raw clock).
     Deglitch protection covers only the initial ~72-cycle window; raw runts during
     operation propagate to both clk_sys and clk_dat roots unprotected.
     Waveform plan updated: capture TO_OSC_CLK raw continuously from reset release
     past the hang point (not just the first 50us).
   - MEASURED then RETRACTED: E1/E2 showed no FSM-state change, which first looked
     like "dat clock stopped is excluded -> deadlock confirmed". A follow-up read of
     D2h PCNT11 also showed NO change, proving the E1 register WRITE never committed.
   - ROOT OF THE ASYMMETRY (RTL-confirmed): RT register WRITE commit is on clk_rt
     (rt_d2.v:511; wrapper:410,440) while READ latch is on clk_axi_sys_lp
     (rt_d2.v:332; wrapper:424,455). Reads work (clk_axi_sys_lp always-on); writes
     do not commit (clk_rt not running in this hang). clk_rt is a switched clock
     (clk_axi_sys <-> MIPI byte clk via rt_slv handshake, rt_slv.v:400-438).
   - CONSEQUENCE: all host-write in-band experiments (E1/E2/E3) are no-ops here and
     unreliable; the prior "deadlock confirmed" conclusion is WITHDRAWN; dat-clock
     liveness is again undetermined. In-band observe/control is effectively absent
     in this state -> debug-observability ECO needed.
   - Next: waveform only. Top probes add clk_rt vs clk_axi_sys_lp, rt_slv
     o_clk_rt_axi_sys_on / w_hold_req_bus / r_mipi_fsm, alongside the P0-SLV/BRIDGE/DAT
     groups and continuous TO_OSC_CLK raw capture.
   - RESOLVED (user): channel = MIPI; after reset the host sends no boot DATA
     (autonomous POR -> VROM default load -> hang); readbacks and the E1/E2 write
     attempts were all poked over MIPI. MIPI reads work (clk_axi_sys_lp), MIPI writes
     do not commit (clk_rt, which switches to the MIPI byte clock via the rt_slv
     handshake on traffic). The write-commit failure is a clk_rt-domain issue,
     SEPARATE from the original clk_vout_s1 VROM-read bug — do not conflate.
   - BOTTOM LINE: the decisive flops (slave r_cur_rfsm, vrom_ctrl r_cur_v1fsm, a2x
     bridge) are not in the RT read mux, so silicon MIPI reads cannot reach them and
     in-band debug is exhausted. Resolving the original bug now requires one of:
     (A) emulation/FPGA repro with full waveform (hard: analog/PVT dependent),
     (B) silicon internal probe (FIB/test chip), or
     (C) observability/recovery ECO — move OTP/VROM slave to clk_axi_sys
     ("//need to fix"), add read-handshake timeout+recovery, route slave live FSM to
     an always-on hang-safe readback. Recommended near-term path: review the C ECOs
     with the design team since further measurement is impractical.

## Customer report

logic_scan_top_debug_customer_report_20260610.pptx
   - 12-slide customer-facing status report (Korean).
   - Covers: symptom, approach, physical hypothesis, three RTL findings
     (clk_vout_s1 topology / clock-not-started exclusion / slave deadlock mechanism),
     the D0h readback correction, E1/E2 in-band experiments, waveform plan,
     design-owner questions, and summary.

logic_scan_top_analog_wave_scenario_20260616.pptx
   - 4-slide deck DRAWING the analog waveform behavior that triggers the hang.
   - FAIL-case timing (TO_RESX_SPROUT / OSC bias+clk en / TO_OSC_CLK analog with the
     buffer Vih threshold / resulting digital / clk_sys), GOOD-vs-FAIL contrast (same
     signals, only startup quality differs), and the 3 analog candidates (A slow OSC
     startup / amplitude < Vih, B duty-freq distortion, C IO_DIS_EXCLK_Y late settle).
   - Traces are real freeform polylines; analog->digital is computed via a Vih/Vil
     buffer model so missing/runt edges are self-consistent. Conceptual, not measured.
   - rev2: accounts for the 1~3µs OSC deglitch gate (user-confirmed). Because the gate
     is EDGE-counted (72 raw OSC edges via div8), slow-amplitude startup self-protects
     (gate opens later) -> naive mechanism A largely excluded. Residual, gate-bypassing
     candidates: C IO_DIS_EXCLK_Y/EXCLK mux (downstream of deglitch, un-gated, now top),
     A' runt/extra-edge opening the gate early, D fq_sel_change (AOFC) re-masking. See
     finding doc section 2.5 [정정].

logic_scan_top_reset_low_WA_rationale_20260616.pptx
   - 9-slide customer deck explaining WHY changing the reset-low width is a valid W/A.
   - Causal chain (reset-low width -> analog/pad discharge state -> clock-startup
     quality -> one-shot transient in the 800us~1.3ms window -> VROM read hang),
     discharge-curve chart (two time-constants, fail window marked), 3-zone timeline
     (short retain / mid worst / long cold-start), temperature cross-check, and the
     apply-guide + limits (trigger-avoidance only; pair with the clk_axi_sys/ timeout ECO).

## Helper scripts

../tools/vcd_timing_summary.py
   - Parses VCD header/value changes.
   - Reports initial values and first 0->1 transitions around TO_RESX_SPROUT.

## Current conclusion snapshot

Immediate digital wait chain:
- sys_fsm = SYS_RST_________VROM
- nvm_fsm = NVM____VROM_LOAD_REQ
- vrom_ctl_fsm = VROM_CTL_DMA_LV2
- vrom_lv2_fsm = VROM_UCS__9E
- DMA slave = EXE____3___ACT
- DMA master read = RADR0_DONE / RDAT0_BURST_SRT

Immediate symptom:
- VROM 9E DMA completion is not returning.

Reset-low-window interpretation:
- Because TO_RESX_SPROUT low turns OSC off, the 800us~1.3ms dependency is unlikely to be ordinary synchronous RTL counting during reset-low.
- Higher-priority causes are OSC analog restart, first-clock quality, deglitch startup, pad/mode skew, and clk_sys vs clk_axi_sys startup alignment.

RTL topology correction (see document 8):
- The VROM 9E read completion does NOT depend on clk_axi_sys alone; it crosses an async a2x bridge
  into the clk_vout_s1 (dat-path) domain where the OTP/VROM slave lives (peri.v "need to fix").
- vrom_ctl_fsm reaching DMA_LV2 proves clk_vout_s1 was alive at SLPOUT-write time; the remaining
  candidates are a dat-clock glitch between the write and the 9E read, or a permanent pulse-loss
  deadlock between axi4_slv_rd_new (VRREADY) and vrom_ctrl (VREADY).
- D0h read byte13 was measured all-zero in fail, but turned out to read an RT mirror register
  (default 0), not live slave status -> inconclusive. The no-waveform discriminator is now the
  in-band E1/E2 experiment (manual dat-clock divider switch / dat exclk select toggle during hang).

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
