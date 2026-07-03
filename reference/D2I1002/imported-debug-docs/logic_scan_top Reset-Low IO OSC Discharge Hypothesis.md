---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top Reset-Low IO OSC Discharge Hypothesis"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_reset_low_io_osc_discharge_hypothesis.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top reset-low IO/OSC discharge hypothesis

Goal:
- reset-low 800us~1.3ms에서만 fail하고, 고온/저온에서는 재현되지 않는 현상을 설명하기 위한 refined hypothesis를 정리한다.
- clk_sys와 clk_axi_sys/clk_axi_sys_lp가 같은 clock root라는 조건을 반영하여, clock root skew가 아니라 IO pad/OSC analog discharge와 reset release 순간의 clock-source/startup 조합을 1차 원인으로 본다.

## 1. Corrected interpretation

clk_sys와 clk_axi_sys/clk_axi_sys_lp는 같은 clock root에서 파생된다.
따라서 reset-low 길이 의존성을 다음처럼 설명하는 것은 약하다.

Wrong emphasis:
- clk_sys와 clk_axi_sys가 독립 root처럼 상대 timing/skew/drift가 틀어진다.
- reset-low 800us~1.3ms 자체가 digital clock-domain timing 차이를 직접 만든다.

Correct emphasis:
- reset-low 동안 OSC가 off이면 internal synchronous counter는 돌지 않는다.
- reset-low duration을 기억할 수 있는 것은 digital counter가 아니라 analog/pad/IO/OSC/POR/reset-delay node의 discharge 또는 retention 상태다.
- clk_sys/clk_axi_sys 차이는 1차 원인이 아니라, IO/OSC startup 상태가 달라진 뒤 나타나는 2차/3차 sequencing 결과로 봐야 한다.

Refined causal chain:

```text
reset-low duration
  -> IO pad node discharge / threshold crossing state
  -> OSC bias/startup node discharge / retention state
  -> reset release 순간 pad settle, clock source select, OSC first-clock quality 변화
  -> deglitch/mux/reset-mask/gate progression 변화
  -> VROM/DMA/AXI transaction이 source/target readiness와 나쁜 조합으로 시작
  -> VROM_UCS__9E / DMA EXE3 / RDAT0_BURST_SRT wait
```

## 2. Why reset-low length points to discharge/retention

TO_RESX_SPROUT low turns OSC off.
Therefore reset-low 800us~1.3ms is not counted by internal synchronous RTL.
The duration can matter only if some physical/analog state changes with time while reset is held low.

Candidate time-memory elements:
- IO pad capacitance and weak pull network
- pad input buffer / Schmitt / level-shifter internal nodes
- ESD/parasitic capacitance around pad inputs
- clock source select pad path, especially IO_DIS_EXCLK_Y
- scan/test/mode pad paths: IO_PNSW_Y, IO_TMODE_Y, IO_DBIST_Y, IO_PMODE_Y, IO_IM_Y
- OSC bias node
- OSC startup/amplitude/control node
- OSC output buffer threshold crossing
- POR/reset analog delay node
- leakage-dependent floating or weakly-held nodes

Typical reset-window behavior:

```text
short reset-low:
  IO/OSC nodes still retain previous-good state
  -> restart is short-reset-like
  -> pass

mid reset-low, 800us~1.3ms:
  one path partially discharged and another path partially retained
  -> reset release creates the worst combination
  -> fail

long reset-low:
  IO/OSC nodes fully discharged or enter defined cold-start state
  -> restart is predictable
  -> pass
```

This also explains why hot/cold can pass while mid temperature fails:

```text
cold:
  leakage small, nodes remain more retained
  -> 950us behaves short-like
  -> pass

hot:
  leakage large, nodes discharge more completely
  -> 950us behaves long-like
  -> pass

middle temperature:
  one node retained and another discharged, or both are near threshold
  -> worst relative state
  -> fail
```

## 3. Relationship between IO discharge and OSC discharge

IO discharge and OSC discharge may not share the same physical node.
The risk is their relative discharge and restore timing.

Conceptually:

```text
V_IO(T)  ~= V_IO0  * exp(-T / tau_IO)
V_OSC(T) ~= V_OSC0 * exp(-T / tau_OSC)
```

If tau_IO and tau_OSC differ, a specific reset-low T can create a bad combination:

```text
IO path near input threshold
  + OSC path near marginal startup state
  -> reset release produces unstable clock-source select and/or poor first OSC cycles
```

The fail is not necessarily caused by IO alone or OSC alone.
The most likely mechanism is the relative timing between:
- IO pad settle to a stable digital value
- OSC enable and first valid OSC edge
- clock mux select stability
- deglitch output / clk_sys first valid edge

## 4. Highest-priority concrete hypothesis: IO_DIS_EXCLK_Y + OSC restart

Important RTL relation from sys_clk.v:

```verilog
assign w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel;
```

Design comment:

```text
i_dis_exclk = 1 -> OSC selected
i_dis_exclk = 0 -> EXCLK selected
```

If reset-low 800us~1.3ms leaves IO_DIS_EXCLK_Y near a marginal discharge/threshold state, then reset release can produce:

```text
IO_DIS_EXCLK_Y settles late or glitches low
  -> w_sys_exclk_sel temporarily selects EXCLK
  -> IO_EXCLK_Y may be absent/unstable
  -> clk_sys first window is missing, X/glitchy, or discontinuous
  -> IO_DIS_EXCLK_Y later settles to OSC select
  -> OSC path and clock mux recover, but initial state/progression is already different
  -> VROM/DMA/AXI sequence enters fail state
```

This directly ties reset-low duration to the observed failure without requiring independent clk_sys/clk_axi_sys roots.

Smoking gun:
- Fail case only: w_sys_exclk_sel selects EXCLK during reset release.
- EXCLK is absent or not valid.
- clk_sys first valid behavior differs only in fail case.
- Hot/cold 950us pass cases do not show this transient.

## 5. Other IO discharge hypotheses

Mode/test pad transient can also explain reset-low sensitivity.

Signals:
- IO_PNSW_Y
- IO_TMODE_Y[*]
- IO_DBIST_Y
- IO_PMODE_Y
- IO_IM_Y[*]
- i_scan_mode / w_scan_mode if available
- reset mux select signals

Possible fail mechanism:

```text
reset-low 800us~1.3ms
  -> mode/test pad path near threshold
  -> scan/test mode transient at reset release
  -> reset mux or clock mux temporarily selects scan/test path
  -> normal reset/clock release sequence differs
  -> VROM/DMA startup state diverges
```

## 6. OSC discharge-only hypothesis

Even if IO pads are normal, OSC can create the reset-low window by itself.

Possible behavior:

```text
short reset-low:
  OSC bias node retained enough
  -> fast clean restart
  -> pass

mid reset-low:
  OSC bias node partially discharged to worst bias point
  -> delayed startup, runt pulse, missing pulse, duty distortion, jitter
  -> deglitch/divider/mask progression differs
  -> fail

long reset-low:
  OSC fully cold-starts from defined condition
  -> predictable startup
  -> pass
```

Check:
- IP_OSC_BIAS_EN
- IP_OSC_CLK_EN
- TO_OSC_CLK analog waveform if available
- TO_OSC_CLK digital first edge
- first 100~1000 cycles period/high/low histogram
- runt/missing pulses
- sys_clk_osc_overshoot_protect.r_clk_i_osc_glitch_mask[8:0]
- w_clk_oscdat_deglitch
- clk_sys

## 7. Updated priority order

The previous clk_sys vs clk_axi_sys availability discussion should be treated as a downstream effect, not the reset-low root cause.

Updated priority:

1. IO pad discharge / threshold crossing, especially IO_DIS_EXCLK_Y.
2. OSC bias/startup discharge and first-clock quality.
3. Relative IO settle vs OSC first-clock timing.
4. Clock mux/deglitch/reset-mask progression caused by 1~3.
5. VROM/DMA/AXI transaction starting before source/target path is ready.
6. AXI R response missing as immediate digital stop point.

## 8. Measurement plan

Minimum comparison cases:

```text
A. mid-temp, short-good reset-low below 800us
B. mid-temp, 950us fail reset-low
C. mid-temp, long-good reset-low above 1.3ms
D. cold, 950us pass
E. hot, 950us pass
```

Priority waveform signals:

IO/mux group:
- TO_RESX_SPROUT
- IO_DIS_EXCLK_Y
- IO_EXCLK_Y
- rm_sys_exclk_sel
- w_sys_exclk_sel
- w_clk_ex
- w_clk_m_oscsys
- w_clk_m_sys_mux
- clk_sys

Mode/test group:
- IO_PNSW_Y
- IO_TMODE_Y[*]
- IO_DBIST_Y
- IO_PMODE_Y
- IO_IM_Y[*]
- i_scan_mode / w_scan_mode
- reset mux outputs if available

OSC group:
- IP_OSC_BIAS_EN
- IP_OSC_CLK_EN
- TO_OSC_CLK
- r_clk_i_osc_glitch_div_2/4/8
- r_clk_i_osc_glitch_mask[8:0]
- w_clk_osc_deglitch1
- w_clk_osc_deglitch2
- w_clk_oscdat_deglitch
- clk_sys

Downstream correlation group:
- o_clk_mask_rstb
- w_sys_reset_mask
- w_clk_sys_reset_mask
- clk_axi_sys
- clk_axi_sys_lp
- o_vrom1_default_load_dma_srt
- i_cdc_exe3_srt
- o_exe3_srt
- AR handshake
- RVALID/RLAST

## 9. Key timestamps and deltas

Use TO_RESX_SPROUT rising as t0.

```text
t0  = TO_RESX_SPROUT rising
t1  = IO_DIS_EXCLK_Y stable to OSC-select value
t2  = w_sys_exclk_sel stable to OSC path
t3  = IP_OSC_CLK_EN rising
t4  = TO_OSC_CLK first valid edge
t5  = TO_OSC_CLK first stable N cycles
t6  = w_clk_oscdat_deglitch first valid edge
t7  = clk_sys first valid edge
t8  = o_clk_mask_rstb deassert
t9  = clk_axi_sys first valid edge
t10 = clk_axi_sys_lp first valid edge
t11 = o_vrom1_default_load_dma_srt
t12 = AXI AR handshake
t13 = first RVALID / RLAST
t14 = i_vrom1_default_load_dma_end
```

Most useful deltas:

```text
Delta_IO       = t1 - t0
Delta_MUX      = t2 - t0
Delta_OSC      = t4 - t3
Delta_STABLE   = t5 - t4
Delta_DEGLITCH = t6 - t4
Delta_SYS      = t7 - max(t1, t4)
Delta_DMA      = t11 - t7
Delta_R        = t13 - t12
```

Fail evidence is strongest if mid-temp 950us fail differs from both hot/cold 950us pass in Delta_IO, Delta_MUX, Delta_OSC, or Delta_DEGLITCH before the DMA/AXI state diverges.

## 10. Final working conclusion

Reset-low window fail should not be primarily explained as clk_sys vs clk_axi_sys root timing.
They share a root.

The better working model is:

```text
reset-low 800us~1.3ms
  -> IO pad path and OSC analog path discharge with different time constants
  -> reset release happens when one or both paths are near marginal threshold/startup condition
  -> IO_DIS_EXCLK_Y / scan-test pads and OSC first-clock behavior create a bad startup combination
  -> clock mux/deglitch/reset-mask progression differs from short-good/long-good/hot-good/cold-good
  -> VROM 9E DMA transaction reaches AXI read-data wait and completion never returns
```

One-line summary:

```text
The reset-low dependency is most likely an IO/OSC discharge and relative restore-timing problem, not a same-root clk_sys/clk_axi_sys skew problem.
```

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
