---
date: 2026-06-17
description: "logic_scan_top reset-low 800us~1.3ms fail의 IO/OSC discharge 가설과 RTL 수정 불가 조건의 workaround 정당화 정리"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - reset-low
  - workaround
  - customer-explanation
status: active
quarter: Q2-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_reset_low_io_osc_discharge_hypothesis.md
---

# logic_scan_top Reset-Low Workaround Rationale

Back to [[D2I1002 Reference]].

## 1. 현재까지의 관찰 요약

대상:

```text
Project: D2I1002
Top: logic_scan_top
Main path: SYS / NVM / VROM / DMA / AXI read path
```

관찰된 reset-low 의존성:

```text
reset-low 800us~1.3ms window에서 fail
고온/저온에서는 재현되지 않음
```

현재 digital stop point:

```text
SYS_RST_VROM
  -> NVM_VROM_LOAD_REQ
  -> VROM_CTL_DMA_LV2
  -> VROM_UCS__9E
  -> DMA EXE3 active
  -> DMA master RADR0_DONE / RDAT0_BURST_SRT
  -> AXI read-data completion wait
```

즉 보이는 최종 정지점은 `VROM 9E DMA completion`이 돌아오지 않는 것이고, DMA master read side는 AXI R channel completion을 기다리는 상태로 정리되어 있습니다.

## 2. 중요한 정정: clk_sys와 clk_axi_sys는 같은 root

사용자가 명확히 정정한 내용:

```text
clk_sys와 clk_axi_sys/clk_axi_sys_lp의 clock root는 동일하다.
```

따라서 reset-low 길이 의존성을 다음처럼 설명하면 안 됩니다.

```text
clk_sys와 clk_axi_sys가 독립 root처럼 drift/skew한다.
reset-low 800us~1.3ms 자체가 clock-domain relative timing을 직접 만든다.
```

맞는 표현:

```text
같은 root라도 reset release 이후 gate/mask/divider/LP enable에 의해 functional availability가 달라질 수는 있다.
하지만 reset-low 길이를 기억하는 1차 원인은 clock tree가 아니라 IO/OSC/POR 등 physical node의 discharge/retention 상태다.
```

## 3. reset-low 길이 의존성의 핵심 해석

`TO_RESX_SPROUT` low 동안 OSC가 off입니다. 따라서 reset-low 800us~1.3ms를 내부 synchronous RTL counter가 세고 있다고 보기 어렵습니다.

reset-low duration을 기억할 수 있는 후보는 physical/analog 상태입니다.

예:

- IO pad capacitance / weak pull network
- pad input buffer threshold crossing
- level shifter / Schmitt trigger 내부 node
- `IO_DIS_EXCLK_Y` 등 clock/mode select pad path
- OSC bias/startup/amplitude node
- OSC output buffer threshold crossing
- POR/reset analog delay node
- leakage-dependent weakly held node

정리된 causal chain:

```text
reset-low duration
  -> IO pad discharge / threshold crossing state
  -> OSC bias/startup discharge / retention state
  -> reset release 순간 pad settle, clock source select, OSC first-clock quality 변화
  -> deglitch/mux/reset-mask/gate progression 변화
  -> VROM/DMA/AXI transaction이 bad readiness condition에서 시작
  -> VROM_UCS__9E / DMA EXE3 / RDAT0_BURST_SRT wait
```

## 4. IO discharge와 OSC discharge의 관계

IO discharge와 OSC discharge가 같은 node일 필요는 없습니다. 핵심은 서로 다른 time constant를 가진 두 physical path가 reset-low 시간에 대해 다른 상태가 된다는 점입니다.

개념 모델:

```text
V_IO(T)  ~= V_IO0  * exp(-T / tau_IO)
V_OSC(T) ~= V_OSC0 * exp(-T / tau_OSC)
```

reset-low `T = 800us~1.3ms`에서만 다음 조합이 만들어질 수 있습니다.

```text
IO path near input threshold
  + OSC path near marginal startup state
  -> reset release produces unstable clock-source select and/or poor first OSC cycles
```

short / mid / long 해석:

```text
short reset-low:
  IO/OSC nodes still retained enough
  -> warm-reset-like startup
  -> pass

mid reset-low, 800us~1.3ms:
  one path partially discharged and another partially retained
  -> worst reset-release combination
  -> fail

long reset-low:
  nodes fully discharged or defined cold-start state
  -> repeatable startup
  -> pass
```

온도 해석:

```text
cold:
  lower leakage, nodes remain retained
  -> 950us behaves short-like
  -> pass

hot:
  higher leakage, nodes discharge more completely
  -> 950us behaves long-like
  -> pass

middle temperature:
  one node retained while another partially discharged, or both near threshold
  -> worst combination
  -> fail
```

## 5. 가장 중요한 구체 가설: IO_DIS_EXCLK_Y + OSC restart

RTL 근거 from `sys_clk.v`:

```verilog
assign w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel;
```

설계 comment:

```text
i_dis_exclk = 1 -> OSC selected
i_dis_exclk = 0 -> EXCLK selected
```

가능한 mechanism:

```text
reset-low 800us~1.3ms
  -> IO_DIS_EXCLK_Y pad path가 threshold 근처 또는 late settle
  -> reset release 순간 EXCLK path가 transient하게 선택됨
  -> IO_EXCLK_Y가 absent/invalid이면 clk_sys first window가 missing/glitchy/discontinuous
  -> 이후 OSC path가 정상 선택되어도 startup sequence는 이미 diverge
  -> VROM/DMA/AXI가 fail path로 진입
```

이 가설의 장점:

- reset-low 길이 의존성을 physical discharge로 설명 가능
- 고온/저온 pass, 중간온도 fail의 non-monotonic behavior와 양립
- same-root clock 조건과 충돌하지 않음
- 최종 AXI read wait를 upstream startup issue의 digital endpoint로 설명 가능

## 6. OSC restart / deglitch 가설

IO가 정상이어도 OSC path만으로 window를 만들 수 있습니다.

```text
reset-low 800us~1.3ms
  -> OSC bias/startup node가 worst partial-discharge state
  -> delayed first edge, runt/missing pulse, duty distortion, unstable first cycles
  -> deglitch/divider/mask progression differs
  -> clk_sys startup behavior differs
  -> downstream VROM/DMA/AXI sequence diverges
```

관련 RTL point:

```text
sys_clk_osc_overshoot_protect.r_clk_i_osc_glitch_mask[8:0]
- div8 clock으로 shift
- full div1 deglitch output은 mask[8] set 이후
- 60MHz 기준 9 div8 shift는 약 1.2us + edge alignment
```

## 7. RTL 수정 불가 조건과 W/A 방향

현재 고객 설명 자료의 전제:

```text
RTL은 수정할 수 없다.
W/A만 가능하다.
고객에게 W/A가 타당한 이유를 설명해야 한다.
```

따라서 제안 방향은 RTL timeout/retry/FSM/CDC/clock/reset logic 변경이 아닙니다.

W/A principle:

```text
Do not operate through the marginal reset-low window.
Guarantee reset-low is either shorter than the failing partial-discharge window or long enough to force a defined cold-start state, then allow enough OSC/pad/clock-settle guard time before normal access/operation.
```

W/A가 타당한 이유:

```text
If the fail mechanism is a partial IO/OSC discharge state, then avoiding that reset-low duration removes the physical precondition for failure.
If a long reset fully discharges/initializes the relevant nodes, startup becomes cold-start-like and repeatable.
If a short reset preserves the previous-good state, startup remains warm-reset-like and repeatable.
The W/A therefore converts an ambiguous partial state into a controlled known state.
```

## 8. 고객 설명용 W/A option

### W/A-1. reset-low 800us~1.3ms 회피

```text
Known bad region인 800us~1.3ms에서 reset-low를 운용하지 않는다.
Warm reset은 fail window보다 짧게, cold/forced reset은 fail window보다 길게 가져간다.
```

정확한 limit은 characterization data 기반으로 잡아야 합니다.

### W/A-2. boot/status timeout 시 long reset recovery

```text
boot가 완료되지 않거나 expected status가 오지 않으면,
계속 진행하지 않고 long-good reset-low로 recovery reset을 수행한다.
```

이미 VROM/DMA/AXI no-completion wait에 들어간 상태에서 RTL self-timeout을 기대하지 않습니다.

### W/A-3. reset release 후 startup guard time 추가

```text
TO_RESX_SPROUT release 후 IO/OSC/clock settle guard time을 둔다.
정상 operation 또는 failure 판정 전에 충분한 startup settle 시간을 준다.
```

단, guard time은 이미 stuck된 transaction을 고치는 것이 아니라, premature access/judgement를 줄이기 위한 보조 W/A입니다.

### W/A-4. 가능하면 RT readback/status check 사용

```text
reset release 이후 RT readback/status를 확인한다.
VROM/DMA wait 또는 clock/mode unexpected 상태이면 long reset recovery를 수행한다.
```

## 9. 고객용 PPT 작업지시서

Claude용 작업지시서가 생성되었습니다.

```text
/home/tim/pjt_work/D2I1002/top/claude_reset_low_io_osc_ppt_work_instruction.md
```

목표 PPT:

```text
/home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx
```

목표 notes:

```text
/home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_reset_low_workaround_customer_explanation_notes.md
```

PPT 핵심 메시지:

```text
Because RTL cannot be modified, the practical mitigation is to avoid the marginal reset-low/startup condition and force a known-good startup state.
```

최종 고객용 one-line:

```text
Avoid the 800us~1.3ms reset-low window and use controlled reset/startup sequencing because RTL cannot be modified and the W/A prevents entry into the marginal IO/OSC startup condition.
```

## 10. 다음 확인 항목

W/A 정당화를 더 강하게 만들기 위한 waveform/check 항목:

IO/mux group:

- `IO_DIS_EXCLK_Y`
- `IO_EXCLK_Y`
- `rm_sys_exclk_sel`
- `w_sys_exclk_sel`
- `w_clk_ex`
- `w_clk_m_oscsys`
- `w_clk_m_sys_mux`
- `clk_sys`

OSC group:

- `IP_OSC_BIAS_EN`
- `IP_OSC_CLK_EN`
- `TO_OSC_CLK`
- `r_clk_i_osc_glitch_mask[8:0]`
- `w_clk_oscdat_deglitch`
- `clk_sys`

Downstream correlation group:

- `o_vrom1_default_load_dma_srt`
- `i_cdc_exe3_srt`
- `o_exe3_srt`
- AXI AR handshake
- `RVALID/RLAST`
- `i_vrom1_default_load_dma_end`

비교 matrix:

```text
A. mid-temp, short-good below 800us
B. mid-temp, 950us fail
C. mid-temp, long-good above 1.3ms
D. cold, 950us pass
E. hot, 950us pass
```

## Related

- [[D2I1002 Reference]]
- [[Index]]
- [[Home]]
