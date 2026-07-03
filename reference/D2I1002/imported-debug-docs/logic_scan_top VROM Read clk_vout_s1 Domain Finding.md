---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top VROM Read clk_vout_s1 Domain Finding"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_vrom_read_clk_vout_s1_domain_finding.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top VROM 9E read path clock-domain finding (RTL trace result)

Goal:
- VROM 9E DMA read가 실제로 어느 clock/bus 경로를 타는지 RTL에서 끝까지 추적한 결과를 정리한다.
- 기존 가설 문서(logic_scan_top_reset_low_io_osc_discharge_hypothesis.md)의 "clk_sys와 clk_axi_sys는 같은 root" 정리를 유지하면서, 지금까지 문서에 없던 **세 번째 clock branch(clk_vout_s1 = dat-path)** 가 read completion을 지배한다는 사실을 추가한다.
- waveform 없이 RT readback만으로 hang 지점을 즉시 분류할 수 있는 새 readback 항목과 decode 표를 제공한다.

## 1. Key finding: VROM read target is on clk_vout_s1 (dat-path), not clk_axi_sys

RTL trace (full chain):

```text
sys_fsm_nvm_vrom (clk_sys)
  o_vrom1_default_load_dma_srt (pulse, clk_sys)
    -> dma.v u_dma0_vrom_cdc pulse_sync (clk_sys -> clk_axi_sys)        [dma.v:117]
    -> dma_core_slv EXE3 (clk_axi_sys / clk_axi_sys_lp)
    -> dma_core_mst AR read, src = MMAP_SYS_VROM1 = {MMAP_SYS_OTP,8'h02} [mmap.v:31]
    -> amba_bus sys decoder -> sys_s8
    -> u_axi4_bridge_sys_to_vout_s1 (DW_axi_a2x)                        [amba_bus.v:1925]
         clk_pp = clk_axi_sys  /  clk_sp = clk_axi_vout_s1 (= clk_vout_s1)
         A2X_CLK_MODE  = 2  (async crossing)
         A2X_BRESP_MODE= 1  (non-bufferable: bresp는 secondary 실제 완료 후 반환)
    -> axi4_vout_s1 interconnect (clk_vout_s1)
    -> u_otp_vrom_slv_top_new .clk_axi( clk_vout_s1 )   // "need to fix" [peri.v:807]
         axi4_slv_rd_new (clk_vout_s1)  -> vrom_ctrl (clk_vout_s1) -> ROM macros (CK=clk_vout_s1)
```

clk_vout_s1 generation (sys_clk.v):

```text
i_osc_dat
  -> u_oscdat_overshoot_protect (deglitch, clk_sys와 공유 root)          [sys_clk.v:666]
  -> u_mux_oscdat_scanbyp -> w_clk_m_oscdat
  -> u_mux_dat_exclk : S = w_dat_exclk_sel && ~i_scan_mode               [sys_clk.v:693]
       w_dat_exclk_sel = (~i_dis_exclk) ^ rm_dat_exclk_sel               [sys_clk.v:327]
       (sys 쪽 w_sys_exclk_sel과 *별개의* mux/별개의 XOR — IO_DIS_EXCLK_Y에 독립적으로 민감)
  -> w_clk_m_dat
  -> u_cdc_clk_dat_reset_mask (i_clk_mask_rstb sync, *별개의* sync FF)   [sys_clk.v:698]
  -> u_icg_clk_dat (EN = !w_dat_reset_mask)                              [sys_clk.v:701]
  -> w_clk_dat_div__1 -> divider chain
  -> u_mux_dat_pre (sys_clk_mux4, glitchless)                            [sys_clk.v:755]
       sel = i_scan_mode ? 0 : rm_clk_dat_div_man_on ? rm_clk_dat_div
                                                     : i_fq_ofc_div  (AOFC osc_cont_top 출력!)
  -> w_clk_dat -> clk_vout_s1                                            [sys_clk.v:903]
```

Implications:

1. 기존 문서의 P0 그룹(clk_sys, clk_axi_sys, clk_axi_sys_lp)만으로는 이 hang을 설명/관측할 수 없다.
   read completion은 **dat-path branch**가 지배한다.
2. dat-path는 sys-path와 같은 OSC root/deglitch를 공유하지만, 다음이 **독립**이다:
   - EXCLK mux (u_mux_dat_exclk vs u_mux_sys_exclk) — IO_DIS_EXCLK_Y glitch가 두 mux에 다르게 영향 가능
   - reset-mask sync FF (w_dat_reset_mask vs w_sys_reset_mask)
   - divider mux select: **i_fq_ofc_div는 AOFC(osc_cont_top)가 boot 중 바꿀 수 있음** —
     o_fq_sel_change가 osc deglitch mask를 재시작시키는 경로도 존재 [sys_clk_osc_overshoot_protect.v:79-91]
3. a2x bridge는 async이므로 **secondary(clk_vout_s1)가 죽어 있어도 primary 쪽 AR handshake는 완료**된다.
   → DMA가 RADR0_DONE / RDAT0_BURST_SRT에서 RVALID만 기다리는 관측 증상과 정확히 일치.
4. peri.v:807의 `.clk_axi ( clk_vout_s1 ), //need to fix` 주석 — 설계 의도 확인 필요 (design owner 질문).

## 2. Deduction from the observed FSM snapshot

관측: vrom_ctl_fsm = VROM_CTL_DMA_LV2 (즉, PWR__ON 단계의 SLPOUT write ack까지 받음).

- SLPOUT write 대상도 같은 OTP slave (MMAP_SYS_OTP_CMD) = 같은 bridge/같은 clk_vout_s1 도메인.
- A2X_BRESP_MODE=1 이므로 bresp는 slave에서 write가 **실제 완료된 후** 반환된다.
- sys_mst의 ack은 bvalid 수신 시점이다 [sys_mst.v:144,153].

따라서:

```text
vrom_ctl_fsm이 DMA_LV2에 도달했다
  = SLPOUT write가 clk_vout_s1 도메인에서 실제 완료되었다
  = hang 직전 시점까지 clk_vout_s1은 살아 있었다
```

**"dat clock이 처음부터 안 떴다" 시나리오는 제외된다.** 남는 후보:

- (a) SLPOUT write 완료와 9E read data 구간 사이에 clk_vout_s1이 죽거나 glitch
  - IO_DIS_EXCLK_Y 후속 transient가 dat mux만 흔드는 경우
  - AOFC가 이 시점에 i_fq_ofc_div를 바꾸는 경우 (mux4 switchover 동안 출력 정지)
  - o_fq_sel_change로 deglitch mask 재시작 (~1.2us 동안 osc root 정지 — 단 이건 sys도 같이 죽음)
- (b) clk_vout_s1 glitch/runt가 slave 내부 one-shot pulse를 깨뜨려 **FSM 간 deadlock** 고착
  - 이후 clock이 회복되어도 hang은 영구 지속됨 (관측과 부합)

## 2.5 TO_OSC_CLK raw 경로 확인 — deglitch 보호 범위는 초기 윈도우뿐

RTL 확인 결과, TO_OSC_CLK는 toggle(분주) 회로 없이 raw 그대로 클럭 root가 된다.

```text
logic_top.sv:246   input TO_OSC_CLK                       // create_clock 100Mhz, dat_osc
logic_top.sv:4276  .i_osc_dat( TO_OSC_CLK )               // peri로 직결, 중간 toggle FF 없음
peri.v / sys.v     포트 그대로 통과

sys_clk_osc_overshoot_protect.v:
  deglitch1 (정상 운용) : CLK_GATE(EN=mask[8], ICLK=raw TO_OSC_CLK buf)  <- toggle 없음
  deglitch2 (초기만)    : CLK_GATE(EN=mask 1~7, ICLK=div2 toggle FF)
  o_clk_osc = OR(deglitch1, deglitch2)
```

보호 범위 정리:

```text
보호됨   : i_osc_clk_en 직후 첫 ~8-16 cycle 완전 차단,
           ~72 cycle까지 div2(toggle FF 출력)만 통과
보호 안됨: mask 개방 이후의 모든 raw 품질 저하
           - startup dirty 구간이 고정 mask 길이(~72 cycle)보다 길면 그대로 통과
           - 운용 중(부팅 중반 t_w~t_r 포함) marginal bias로 OSC가 흔들리면
             runt가 clk_sys/clk_dat 양쪽 root에 그대로 전파
           - 재마스킹 트리거는 osc_clk_en deassert / fq_sel_change 뿐
```

[정정] 1~3µs deglitch gate를 반영한 보호 범위 (사용자 확인: OSC OFF→ON 시
약 1~3µs 클럭 게이팅으로 내부 입력 차단):

```text
gate 동작 (sys_clk_osc_overshoot_protect.v):
  - 전체속도 출력은 r_clk_i_osc_glitch_mask[8] 에서 열림.
  - mask는 div8(raw OSC) 로 클럭되는 9-bit shift → 72 raw edge 후 open.
  - 따라서 gate는 '시간'이 아니라 'edge 수'를 센다.
  - reset 트리거: i_osc_clk_en low, 그리고 fq_sel_change (w_fq_sel_change_mask_on).

gate가 막는 것 (문제 아님):
  - 느린 진폭 startup. 진폭<Vih면 edge가 적게 잡혀 div8 counter가 느리게 진행
    → gate가 더 늦게 열림 → '느린 기동'은 자동으로 더 기다려 보호된다.
    => 단순 mechanism A('초기 dirty cycle이 그대로 내부로')는 대부분 배제.

gate가 못 막는 것 (잔존 위험):
  - C: EXCLK mux(u_mux_sys_exclk/u_mux_dat_exclk)는 deglitch '하류'이고
       B입력(w_clk_ex)은 deglitch 안 됨. IO_DIS_EXCLK_Y select 글리치는
       gate와 무관하게 통과. => 최우선 잔존 후보.
  - A': runt/double-edge는 div8 counter를 빨리 진행시켜 gate를 일찍 열거나,
        open 순간 첫 full-rate edge가 marginal일 수 있음.
  - D: AOFC가 부팅 중 주파수 변경 시 fq_sel_change가 mask를 재시작 → gate가
       다시 닫혔다 열림 → open 순간 위험이 t_w~t_r 구간에서 재노출.
```

관측(sys 도메인 생존)과의 결합 해석:

```text
raw OSC root는 clk_sys와 clk_dat가 공유한다.
운용 중 root runt가 있었다면 sys 도메인도 깨질 수 있었는데 sys는 멀쩡하다.

1. root 공통 글리치 시나리오: 어느 FF가 깨질지는 확률적이므로
   sys는 살아남고 dat 쪽 slave의 1-cycle pulse만 유실됐을 가능성 — 배제 불가
   (영구 deadlock은 단 한 번의 유실로 충분)
2. dat 분기 고유 지점(EXCLK mux select 글리치, mux4 전환) 시나리오:
   root가 깨끗해도 dat 분기만 죽일 수 있는 유일한 지점 — 구조적으로 더 유력

두 경우 모두 4.5절 E1/E2 실험이 그대로 판별한다
(E1/E2로 해소 -> dat 분기 / 미해소 -> pulse 유실 deadlock, root든 dat든).
```

추가 주의: "EXCLK mux 글리치" 경로에서 u_mux_sys_exclk / u_mux_dat_exclk 는
glitchless mux(sys_clk_mux2/4)가 아니라 단순 CLK_MUX 셀이며 [sys_clk.v:387,693],
select는 IO_DIS_EXCLK_Y에서 오는 조합 신호다. deglitch가 깨끗한 클럭을 내보내도
select가 흔들리면 mux 출력에서 글리치가 새로 생성된다 (deglitch 보호 범위 밖, 하류).

## 3. The exact digital hang point inside the OTP/VROM slave

axi4_slv_rd_new.sv read FSM (clk_vout_s1):

```text
SLV_READ__ARADDR : VROM addr && !i_vrm_sleep_stat -> SLV_READ_VRREADY     [axi4_slv_rd_new.sv:189]
SLV_READ_VRREADY : w_rfifo_push 나올 때까지 무한 대기 (rvalid=0)          [axi4_slv_rd_new.sv:195]
  w_rfifo_push <- w_vrm_rd_en <- vrom_ctrl o_vrm_rd_en = (ROM1/2 FSM == OPERATE)
```

vrom_ctrl.sv ROM1 FSM:

```text
DPSTB   --(wakeup_cmd: SLPOUT write decode)--> VREADY
VREADY  --(w_vrm1_rd_requst 1-cycle pulse || react)--> OPERATE   <- 여기서 pulse를 놓치면 영구 deadlock
OPERATE --(i_vrm_hold_ || arlen_ful)--> VREADY
react   = w_rfifo__pop && s_rd_vrdata (1-cycle pulse)                      [axi4_slv_rd_new.sv:371]
hold_   = w_almost_full && !rready (level)                                 [axi4_slv_rd_new.sv:372]
```

Deadlock 성립 조건 (둘 다 1-cycle pulse 의존):

```text
1. r_vrm_rd_requst pulse가 ROM1 FSM에 전달되지 못함 (예: runt clock이 일부 FF만 클럭)
   -> slave는 VRREADY에서 push 대기, vrom_ctrl은 VREADY에서 request 대기 -> 영구 hang
2. burst 중 hold로 VREADY 복귀 후 react pulse 유실
   -> slave는 VRDATA/VRREADY에서 데이터 대기, vrom_ctrl은 VREADY 대기 -> 영구 hang
3. wakeup_cmd 유실 (ROM1=DPSTB 고착)인데 sleep_stat=0 (ROM2가 깨어 있을 때)
   -> VRREADY 진입 후 request가 DPSTB 상태 FSM에 무시됨 -> 영구 hang
```

정상 clock에서는 모두 same-domain 1-cycle pulse라 유실되지 않는다.
**reset-low window에서 dat-path 첫 clock 품질이 나쁘면(runt/missing edge) 단 한 번의 pulse 유실로 고착**되고,
이후 clock이 정상화되어도 회복되지 않는다 — hot/cold에서 재현되지 않고 특정 reset-low 폭에서만 나오는 현상과 부합한다.

## 4. RT readback 정정 (측정 결과 반영): D0h byte13은 live status가 아니다

**측정 결과**: fail 상태에서 D0h PCNT13 = 0x00,0x00,0x00,0x00 (4 byte 모두).

**정정**: 이 readback은 slave의 live FSM 상태가 아니라 **RT 쪽 미러 레지스터를 되읽는 것**이다.

```text
rt_d0.v:626  rm_d0_54 <= r_wdat2   (command WRITE로만 갱신, reset default 0)
rt_d0.v:702  rm_rom1_rd_st = rm_d0_54[5:4]   <- 레지스터 필드, 슬레이브와 미연결
rt_d0.v:420  D0h read byte13은 이 레지스터 필드를 그대로 되읽음
```

즉 **all-zero는 hang 여부와 무관하게 항상 나오는 기본값**이며, 이 측정은 가설을
확인도 반박도 하지 못한다 (inconclusive). 이전 버전 문서의 decode 표는 무효.

실제 live status의 유일한 위치:

```text
otp_reg_table.v:825  reg offset 12'h017 (조합 mux, 레지스터 아님):
  [31]=wr_bus_err [30]=rd_bus_err [29]=mcrc_err [28]=tcrc_err
  [27:24]=otp_bus_rd_st [23:22]=rom1_rd_st [21:20]=rom2_rd_st
  [18:16]=otp_bus_wr_st [13:8]=otp_cur_st [7:0]=otp_rt_pdout
  -> AXI address 0x4401_005C (= {MMAP_SYS_OTP,8'h01} + 0x17*4)
```

그러나 이 read는 **죽어 있을 가능성이 있는 clk_vout_s1 도메인을 통과**하므로:
- hang 중에 이 주소를 읽으면 read를 발행한 master가 그대로 같이 hang된다 (사용 금지).
- monitor pin mux에도 dat-path clock / vout_s1 신호는 없음 (clk_osctest_sys는 sys-path만).

**Observability gap (design owner 전달 항목)**: OTP/VROM slave의 live 상태를
hang-safe하게 읽을 수 있는 경로가 칩에 존재하지 않는다. i_ro_* 신호를 CDC sync 후
rt_d0 read mux에 직접 배선하는 ECO를 권고 (rm_d0_54/55 미러는 이름만 status).

State encodings (waveform/향후 ECO 시 참고):

```text
rom1/2_rd_st : 00=DPSTB(sleep)  01=VREADY  10=OPERATE                      [vrom_ctrl.sv:54]
otp_bus_rd_st: 0000=IDLE 0001=ARADDR 0010=RESERV 0011=RRREADY 0100=ORREADY
               0101=VRREADY 0110=RRDATA 0111=ORDATA 1000=VRDATA
               1001=RLAST 1010=RESDATA                                     [axi4_slv_rd_new.sv:82]
```

## 4.5 In-band 판별 실험: hang 상태에서 dat clock 강제 전환

RT는 hang 중에도 살아 있으므로 (D0h readback 동작 확인됨), 다음 레지스터로
**clk_vout_s1을 강제로 흔들어 hang 해소 여부를 보는 실험**이 waveform 없는 최선의 판별이다.

사용 레지스터 (모두 RT-writable 확인됨):

```text
rm_clk_dat_div_man_on = rm_d2_46 영역, D2h command       [rt_d2.v:613-, peri.v:555]
rm_clk_dat_div        = rm_d2_46[7:6]                     [rt_d2.v:613]
rm_dat_exclk_sel      = FFh command 레지스터              [peri.v:546]
```

실험 순서 (fail/hang 상태에서):

```text
E1. rm_clk_dat_div_man_on=1, rm_clk_dat_div=01 (div2) 써서 dat mux를 수동 전환
    -> 전환 자체가 mux4 (sys_clk_mux4) 를 거치므로, i_fq_ofc_div 쪽 입력이 죽어
       출력이 멈춘 경우라면 살아 있는 divider로 갈아타며 clk_vout_s1이 재기동된다.
    -> a2x bridge에 9E read AR이 보관되어 있으므로 clock이 살아나면
       DMA가 그대로 진행 재개 -> sys_fsm이 풀리는지 FFh PCNT로 확인.
E2. (E1로 안 풀리면) rm_dat_exclk_sel 토글 (0->1 또는 1->0)
    -> dat exclk mux가 부재한 EXCLK에 물려 죽은 경우 OSC 쪽으로 강제 복귀.
    주의: rm_sys_exclk_sel은 건드리지 말 것 (clk_sys까지 죽일 수 있음).
E3. 둘 다 무효면 dat clock은 살아 있다고 판단
    -> slave 내부 pulse-loss deadlock (3절 mechanism 1~3) 쪽으로 분류.

해석:
  E1로 해소  -> divider mux / i_fq_ofc_div (AOFC) 경로 정지 확정
  E2로 해소  -> IO_DIS_EXCLK / dat exclk mux 경로 확정 (기존 가설 문서의 1순위와 직결)
  미해소     -> clk은 살아 있고 slave 내부 deadlock -> 회복 불가 (reset 필요),
               waveform으로 P0-SLV 그룹 확인 필요
```

추가 보조 확인: E1/E2 이전에 D2h PCNT 11 read로 rm_clk_dat_div / rm_clk_dat_div_man_on
현재값 기록 [rt_d2.v:362,379].

## 4.6 [RETRACTED] E1/E2 결과의 초기 해석 — 무효

아래 4.6 본문의 "E1/E2 무효 -> dat 클럭 정지 배제 -> deadlock 확정" 결론은
4.7절의 추가 측정으로 **철회한다**. E1/E2는 host register WRITE에 의존하는데,
4.7에서 그 write 자체가 hang 중에 커밋되지 않음(write-commit 클럭 도메인 문제)을
확인했다. 따라서 E1/E2는 no-op였고, dat 클럭 생사/​deadlock 에 대해 아무것도
증명하지 못한다. 아래는 기록 보존용.

---

**측정 결과**: E1(divider 수동 전환), E2(dat exclk select 토글) 모두 수행 —
FSM state readback 값 변화 없음 (hang 미해소).

**(철회된) 해석**:

```text
E1+E2 수행 후의 dat-path 상태는 구성상 known-good 이다:
  - E2: u_mux_dat_exclk 는 조합 select의 단순 CLK_MUX -> 클럭 없이도 전환됨.
        EXCLK 오선택으로 w_clk_m_dat가 죽어 있었다면 OSC 경로로 강제 복귀됨.
  - w_clk_m_dat 부활 시 dat reset-mask sync FF는 i_clk_mask_rstb=0 을 2 cycle 내
    재샘플 -> gate 재개방 (래치 고착 구조 아님).
  - E1: divider mux select 를 정적 값으로 고정 (AOFC 의존 제거).
  - clk_vout_s1 = w_clk_dat 직결 (별도 gate 없음).

그럼에도 DMA가 재개되지 않음
  => "dat 클럭이 지금 죽어 있어서 멈춤" 분기(decision tree 4.5)는 배제.
  => clk_vout_s1 도메인 내부의 일회성 상태 손상(one-shot state corruption)
     deadlock 으로 분류 확정 (decision tree 4.6).
     위치 후보: a2x bridge SP-side read 기구(async FIFO pointer 등)
              / slave read FSM <-> vrom_ctrl pulse 핸드셰이크 (3절 mechanism 1~3)
  => 원인 사건은 지속적 클럭 정지가 아니라 t_w~t_r 구간의 일회성 transient
     (raw OSC runt / IO_DIS_EXCLK transient / AOFC 전환 글리치)로 압축.
```

보조 확인 권장: E1 레지스터 write가 실제 반영됐는지 D2h PCNT 11 readback으로 확인
(미확인 상태라면 결론의 전제 검증용).

**E3 (후속 in-band): DBI bus-write probe — write 경로 생존 확인**

근거 RTL:
- host(DBI) 명령으로 임의 sys-bus 주소에 write하는 경로가 존재한다:
  sys_exi_dbi (DBI_ADR/DBI_WDAT 시퀀스 -> o_mst_dbi_wreq/addr/wdat) -> sys_mst
  MST_WRIT_DBI_ADR/LST/BRE [sys_exi_dbi.v:239, sys_mst.v:146-148].
- sys_mst write FSM은 현재 IDLE (SLPOUT ack 이후 요청 없음) -> hang 중에도 사용 가능.
- 단, DMA direct write 경로는 사용 불가: wadr FSM이 9E 전송에 묶여
  MST_WADR_READY 등 non-IDLE 상태 [dma_core_mst.v:455-460]. (C4h로 wadr_fsm 확인 가능)

절차 (hang 상태에서):

```text
E3-1. DBI bus-write -> 무해한 RT 레지스터(주소 A)에 V1 기록
      -> RT readback으로 V1 확인 = DBI bus-write 경로 자체의 동작 baseline
E3-2. DBI bus-write -> 0x4401_0010 (VROM1 SLPOUT cmd 재전송, 무해/멱등:
      이미 깬 ROM에는 무시되고, wakeup 유실 case라도 부작용 없음)
E3-3. DBI bus-write -> RT 레지스터(주소 A)에 V2 기록 -> RT readback

판정:
  V2 확인됨   -> E3-2 write가 bvalid까지 완료됨
              -> bridge AW/W + vout_s1 interconnect + slave write FSM + clk_vout_s1
                 모두 생존 확정
              -> 고착 위치는 read 경로 내부(slave read FSM <-> vrom_ctrl)로 최종 압축
  V2 미확인   -> E3-2가 bvalid 대기 hang = sys_mst write FSM 고착
              -> bridge SP side 또는 도메인 광역 문제로 분류
              -> 이후 DBI bus-access 경로는 chip reset까지 사용 불가 (수용 가능:
                 칩은 어차피 reset 필요 상태)
```

주의: E3에서 read는 절대 사용하지 말 것 (0x4401_xxxx read는 sys_mst read FSM을
영구 고착시킴). write만 사용한다.

> 주의: E3도 host register write에 의존하므로 4.7절 결론(write-commit 불가)에 따라
> 현재 상태에서는 신뢰할 수 없다. 4.7 참조.

## 4.7 [측정] D2h PCNT11 무변화 -> RT register write/read 클럭 분리 발견

**측정 결과**: E1 write 후 D2h PCNT11 readback = 변화 없음 (rm_clk_dat_div 그대로).
즉 E1 write가 레지스터에 커밋되지 않았다.

**RTL 근거 (확정):**

```text
RT register WRITE 커밋 : always @(posedge clk_rt)          [rt_d2.v:511, rm_d2_46]
                         상위도 clk_rt                      [wrapper:410,440 r_lv2_wcmd/wden]
RT register READ 래치  : always @(posedge clk_axi_sys_lp)   [rt_d2.v:332, r_1st_rdat2]
                         상위도 clk_axi_sys_lp               [wrapper:424,455 r_lv2_rcmd]

D2h PCNT11 = {rm_clk_dat_div, ...} 이고 rm_clk_dat_div = rm_d2_46[7:6]
-> PCNT11이 안 바뀜 = rm_d2_46(clk_rt 도메인) write가 커밋 안 됨
```

**핵심 비대칭의 정체:**

```text
읽기 OK / 쓰기 실패는 dat-path와 무관한 별개 원인이다:
  - 읽기 datapath = clk_axi_sys_lp (always-on, sys-path) -> 살아있음
  - 쓰기 커밋     = clk_rt        -> hang 중 미동작
clk_rt 는 switched clock:
  default  : clk_axi_sys gated by i_clk_rt_axi_sys_on(=~w_hold_req_bus, reset시 1)
  MIPI 시  : clk_rt_mipi_fifo gated by i_clk_rt_mipi_on   [sys_clk.v:569-571]
  스위치 핸드셰이크는 rt_slv (w_hold_req_bus / w_hold_ack)  [rt_slv.v:400-438]
설계가 readback을 일부러 always-on clk_axi_sys_lp에 둔 것은
clk_rt 가 unavailable 할 수 있음을 전제한 구조로 보인다.
```

**결론 / 영향:**

```text
1. E1/E2/E3 등 host-write 기반 in-band 실험은 이 hang 상태에서 전부 신뢰 불가.
   write commit 클럭(clk_rt)이 동작하지 않으므로 no-op가 된다.
2. 4.6의 deadlock 확정 결론 철회. dat 클럭 생사는 아직 미확정.
3. 읽기/쓰기 front-end는 동일 DCS 채널이므로(읽기가 데이터 반환 = decode 동작),
   분기점은 write-commit 도메인(clk_rt) 하나로 좁혀진다 -> clk_rt write 도메인 의심.
4. protocol/operator 오류 가능성은 낮다: 정상 bring-up 스크립트가 동일 D2h write를
   사용하며, hang 상태에서만 실패하므로 상태/클럭 의존 문제다.
```

**다음 측정 (write 비의존):**

```text
A. clk_rt 생사 확인은 waveform 필수. 최우선 프로브:
   - clk_rt, clk_axi_sys_lp (둘을 같이 봐서 분리 확인)
   - rt_slv.o_clk_rt_axi_sys_on / o_clk_rt_mipi_on
   - rt_slv.w_hold_req_bus / w_hold_req_mipi / w_hold_ack_bus
   - r_mipi_fsm (rt_slv) — MIPI 핸드셰이크 정지 여부
B. 채널 확인 질문(사용자): FFh/D2h 명령이 어느 물리 채널인가
   (MIPI DSI command mode / DBI / I2C·SPI 디버그).
   - MIPI면 clk_rt가 MIPI byte clock으로 스위치됨 -> 스위치 핸드셰이크가 용의선상.
   - DBI/디버그면 clk_rt = clk_axi_sys(생존 추정) -> write-decode 게이팅 쪽 의심.
C. 강제 제어가 가능한 scan/test 경로 유무(있으면 clk_rt 비의존 forcing 가능).
```

**관측성 종합 (ECO 시급도 상향):**

```text
- D0h PCNT13 status : RT 미러 레지스터 (default 0) -> 무의미 (4절)
- live status reg   : 죽은 도메인 통과, hang-safe 불가 (4절)
- monitor pin       : dat-path/clk_rt 신호 없음
- host register WRITE: clk_rt 도메인 -> hang 중 커밋 불가 (본 절)
=> 이 hang 상태에서 칩의 in-band 관측/제어 수단이 사실상 전무.
   waveform/emulation/scan-dump 외 방법이 없음 -> 디버그 관측성 ECO 필요.
```

## 4.8 [측정 확정] 채널 = MIPI / 자율 부팅 hang / in-band 고갈

**사용자 확인**:
- reset 이후 host가 보내는 부팅 DATA 없음 -> POR -> VROM default load 자율 진행 중 hang.
- 상태 readback 및 E1/E2 write 시도는 모두 MIPI interface 로 수행.

**정리:**

```text
1. 채널 = MIPI. MIPI read는 동작(clk_axi_sys_lp 래치), MIPI write는 커밋 안 됨(clk_rt).
   둘 다 같은 MIPI front-end/FIFO를 거치므로(read 성공 = decode/FIFO 정상),
   갈리는 지점은 최종 commit 클럭뿐:
     read  commit = clk_axi_sys_lp (always-on)        -> OK
     write commit = clk_rt                             -> 미커밋
   MIPI 트래픽이 오면 rt_slv가 clk_rt를 clk_axi_sys -> clk_rt_mipi_fifo로 스위치
   [rt_slv.v:400-438]. 이 스위치 핸드셰이크가 hang 상태에서 미완료면
   clk_rt가 두 소스 모두에서 끊겨 write commit만 유실될 수 있다.
   (단, 이는 "프로브가 안 먹는 이유"이고, 원래 VROM hang과는 다른 클럭 도메인 문제.)

2. 원래 버그(VROM 9E read가 RVALID 미복귀, clk_vout_s1 도메인)와
   write-commit 실패(clk_rt 도메인)는 서로 다른 도메인. 혼동 금지.
   dat-clock-stop vs slave-deadlock 은 여전히 미확정.

3. 결정적 flop(slave r_cur_rfsm, vrom_ctrl r_cur_v1fsm, a2x bridge)은
   RT read mux에 없음 -> MIPI read로 도달 불가(죽은 도메인 경유 필요).
   => 실리콘 MIPI 만으로는 원래 버그를 확정할 수 없음. in-band 고갈.
```

**남은 길 (셋 중 하나 필수):**

```text
A. Emulation/FPGA 재현 + full waveform
   - 단, reset-low 폭/온도 의존(아날로그·PVT)이라 순수 디지털 재현은 어려울 수 있음.
   - marginal 클럭 조건을 강제 주입할 수 있으면 가능.
B. 실리콘 내부 probe (FIB/테스트칩) 로 clk_rt / clk_vout_s1 / TO_OSC_CLK /
   bridge SP / slave FSM 직접 측정.
C. 관측성/복구 ECO (측정 없이 설계로 위험 제거):
   - OTP/VROM slave 를 clk_axi_sys 로 이관 (peri.v:807 "//need to fix" 해소)
     -> read 경로에서 async bridge + dat-clock 의존 제거 (가장 근본적).
   - slave read 핸드셰이크 / DMA read 에 timeout+recovery 추가.
   - slave live FSM 을 always-on 클럭으로 CDC 후 RT read mux 에 직접 배선
     (hang-safe readback).
```

**권고**: 추가 측정이 어려우므로, 확정된 구조적 위험(async bridge 너머
clk_vout_s1 slave + 1-cycle pulse 핸드셰이크 + "//need to fix")에 대해
위 C의 ECO 후보를 설계 측과 우선 검토하는 것이 현실적 최단 경로.

## 5. New P0 waveform group (dat-path / bridge)

기존 P0 그룹에 추가:

```text
P0-DAT. dat-path clock group:
- sys_clk.w_dat_exclk_sel
- sys_clk.w_clk_m_dat
- sys_clk.w_dat_reset_mask
- sys_clk.w_clk_dat_div__1
- sys_clk.w_clk_dat (= clk_vout_s1)
- sys_clk.rm_clk_dat_div_man_on / rm_clk_dat_div
- osc_cont_top.o_fq_ofc_div (i_fq_ofc_div)   <- boot 중 변화 시점 확인
- osc_cont_top.o_fq_sel_change               <- deglitch mask 재시작 트리거

P0-BRIDGE. a2x secondary side group (amba_bus):
- w_arvalid_vout_s1_m1 / w_arready_vout_s1_m1
- w_rvalid_vout_s1_m1  / w_rlast_vout_s1_m1
- w_awvalid_vout_s1_m1 / w_bvalid_vout_s1_m1 (SLPOUT write 완료 시점 확인)

P0-SLV. OTP/VROM slave internal:
- axi4_slv_rd_new.r_cur_rfsm
- vrom_ctrl.r_cur_v1fsm / r_cur_v2fsm
- axi4_slv_rd_new.w_rfifo_push / w_rfifo__pop / w_rfifo_full
- w_vrm_rd_requst (r_vrm_rd_requst) / o_vrm_rd_en / o_vrm_hold_ / o_vrm_react
```

비교 포인트: SLPOUT write의 bvalid_sp 시점(t_w)과 9E read의 arvalid_sp 시점(t_r) 사이에서
clk_vout_s1 파형 품질과 i_fq_ofc_div / w_dat_exclk_sel 변화를 본다.
fail case에서만 t_w~t_r 사이에 dat clock 이벤트(러트/정지/뮤스 전환)가 있으면 mechanism (a)/(b) 확정.

TO_OSC_CLK 캡처 구간 변경 (2.5절 반영):
- 기존 계획(기동 직후 ~50us)으로는 부족하다. raw TO_OSC_CLK가 toggle 회로 없이
  양쪽 클럭 root에 직결되므로, 운용 중 runt도 hang을 만들 수 있다.
- fail case에서 TO_OSC_CLK raw 파형을 reset 해제부터 t_r(hang 시점) 이후까지
  연속 캡처하고, 특히 t_w~t_r 구간의 runt/missing/duty 이상을 1순위로 검사한다.

## 6. Updated decision tree (기존 8단계에 삽입)

기존 follow-up plan의 decision tree 4번과 5번 사이에 추가:

```text
4.5 If o_vrom1_default_load_dma_srt와 EXE3는 정상이고 AR handshake(primary)도 됐지만
    a2x secondary side에 arvalid_sp가 전달되지 않음:
    -> clk_vout_s1 (dat-path) 정지로 분류. w_dat_exclk_sel / w_dat_reset_mask / i_fq_ofc_div 확인.
    -> waveform 불가 시 4.5절 in-band 실험 E1/E2로 동일 분류 가능.
4.6 If arvalid_sp/arready_sp까지 정상인데 slave가 VRREADY/VRDATA에서 고착:
    -> slave 내부 pulse-loss deadlock으로 분류 (in-band 실험 E1/E2 모두 무효인 경우).
       세분은 waveform P0-SLV 그룹 (r_cur_rfsm / r_cur_v1fsm) 으로만 가능.
```

## 7. Questions for design owner

1. peri.v:807 `.clk_axi ( clk_vout_s1 ), //need to fix` — OTP/VROM slave를 clk_vout_s1에 두는 것이
   의도인지, clk_axi_sys로 옮길 예정이었는지.
2. boot 중 (OTP/VROM load 진행 중) AOFC가 i_fq_ofc_div를 변경할 수 있는지,
   변경 가능하다면 VROM DMA와의 순서 보장은 무엇인지.
3. vrom_ctrl의 request/react 1-cycle pulse 핸드셰이크에 timeout/recovery가 없는 것이 의도인지.
   (한 번 유실되면 chip reset 외 회복 수단 없음)
4. rm_dat_exclk_sel / rm_sys_exclk_sel의 reset default가 동일한지 (register map 확인 필요).

## 8. One-line summary

```text
VROM 9E read completion은 clk_axi_sys가 아니라 dat-path 분기(clk_vout_s1)와 async bridge를 거친다.
vrom_ctl_fsm=DMA_LV2 도달 사실로부터 "dat clock 미기동"은 배제된다 (BRESP_MODE=1 확정).
TO_OSC_CLK는 toggle 회로 없이 raw로 root에 직결되므로 deglitch 보호는 기동 초기
윈도우(~72 cycle)뿐이고, 운용 중 raw runt는 양쪽 root에 무방비로 전파된다 —
sys 도메인 생존 관측과 결합하면 (i) root 공통 runt의 확률적 dat측 유실,
(ii) dat 분기 고유 지점(EXCLK mux select / mux4) 글리치 두 시나리오가 남는다.
D0h byte13 readback은 미러 레지스터라 무효(all-zero는 기본값)이고, live status는
hang-safe하게 읽을 수 없다 (observability gap).
[측정] E1/E2 후 D2h PCNT11 무변화 -> host register WRITE가 커밋되지 않음.
RTL 확정: RT register write 커밋은 clk_rt, read 래치는 clk_axi_sys_lp 로 분리됨.
  -> "읽기 OK / 쓰기 실패" 비대칭은 dat-path와 무관하며, clk_rt write 도메인이
     hang 중 미동작인 것이 원인.
  -> E1/E2/E3 등 host-write 기반 in-band 실험은 전부 no-op -> 신뢰 불가.
  -> 직전 turn의 "deadlock 확정"은 철회. dat 클럭 생사는 미확정으로 되돌아감.
in-band 관측/제어 수단이 이 상태에서 사실상 전무(관측성 ECO 필요).
다음 단계는 waveform: clk_rt vs clk_axi_sys_lp 분리 확인 + rt_slv 클럭스위치
(o_clk_rt_axi_sys_on / w_hold_req_bus / r_mipi_fsm) + 기존 P0-SLV/BRIDGE/DAT 그룹,
그리고 TO_OSC_CLK raw 연속 캡처.
사용자 확인 필요: FFh/D2h 명령의 물리 채널(MIPI DSI / DBI / I2C·SPI) — clk_rt 소스를 결정.
```

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
