---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: logic_scan_top FSM Debug Plan"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/docs/plans/logic_scan_top_fsm_debug_plan.md
imported_at: 2026-07-03 10:19:28 KST
---

# logic_scan_top FSM/VROM/DMA Debug Plan

> For Hermes: follow systematic-debugging first. Do not propose RTL fixes until the failing boundary is proven by waveform/RTL evidence.

Goal: logic_scan_top bring-up 상태에서 sys_fsm/VROM/DMA FSM이 현재 값에 머무르는 원인을 reset, pad input delay, OSC clock start, VROM DMA handshake 관점으로 분리해 확인한다.

Architecture: top은 logic_scan_top이다. 현재 hang 후보 경로는 sys_fsm -> sys_fsm_nvm -> sys_fsm_nvm_vrom -> dma_core_slv/dma_core_mst 이다. 분석은 먼저 RTL state 값을 symbolic name으로 decode하고, init.vcd 및 추가 waveform에서 reset/input/clock/handshake의 실제 타이밍을 확인한다.

Tech Stack: Verilog/SystemVerilog RTL, rtl.f filelist, VCD/FSDB waveform, Xcelium xmsim/xrun 환경.

---

## 0. 현재까지 확인된 기본 정보

작업 위치:
- environment.env 존재
- environmet.env 는 없음

환경 변수:
- pjt_path = /home/tim/pjt_work/D2I1002/top

filelist:
- source/rtl/rtl.f

Top module:
- logic_scan_top

초기 VCD:
- init.vcd
- timescale 1ps
- scope U_LOGIC_SCAN_TOP

초기 reset/input 조건:
- TO_PORB_VDD = 1 at t=0
- TO_RESX_SPROUT = 0 at t=0
- TO_RESX_SPROUT 0->1 at 999950000 ps = 999950.000 ns
- init.vcd 상에서는 여러 *_Y 입력이 TO_RESX_SPROUT과 같은 timestamp에서 0->1로 보임
- 실제 회로 해석 기준: TO_RESX_SPROUT 0->1 시점에 *_Y 입력은 동시에 안정화되지 않고 수 ns delay를 가지고 각각 1이 됨

OSC 조건:
- TO_RESX_SPROUT = low 이면 OSC off, TO_OSC_CLK 등 전체 OSC-derived clock 없음
- OSC_BIAS_EN 및 OSC_CLK_EN assert 후에도 clock 출력은 즉시가 아니라 약 3~5us 뒤 시작
- 따라서 reset release 직후 수 ns pad settling 구간과 OSC clock start 전 3~5us 구간을 정상 wait 구간으로 분리해야 함

현재 확인된 FSM raw 상태:
- sys_fsm.v: r_sys_fsm = 5'h04
- sys_fsm_nvm.v: r_nvm_fsm = 4'h1
- sys_fsm_nvm_vrom.v: r_vrom_ctrl_fsm = 3'h3
- sys_fsm_nvm_vrom.v: r_vrom_lv2_fsm = 8'h9E
- sys_fsm_nvm_vrom.v: r_vrom_lv3_fsm = 8'h00
- dma_core_slv.v: r_slv_fsm = 4'h6, 실제 RTL reg width는 [2:0]
- dma_core_slv.v: r_exe_fsm = 5'h0E
- dma_core_mst.v: r_pop_fsm[1:0] = 2'h0
- dma_core_mst.v: r_rdat_fsm[2:0] = 3'h1
- dma_core_mst.v: r_radr_fsm[2:0] = 3'h2
- dma_core_mst.v: r_wdat_fsm[3:0] = 4'h0
- dma_core_mst.v: r_wadr_fsm[3:0] = 4'h1
- ro_sys_fsm_nvm_otp_fsm[4:0] = 5'h00
- ro_sys_fsm_nvm_mcu_fsm[4:0] = 5'h10
- ro_sys_fsm_nvm_fls_fsm[3:0] = 4'h0

---

## 1. RTL state decode 결과

### 1.1 sys_fsm.v

File:
- source/rtl/peri/sys/sys_fsm.v

Decode:
- r_sys_fsm = 5'h04 = SYS_RST_________VROM

관련 전이 의미:
- reset sequence 중 VROM default load 단계로 보임
- 이 상태의 done 조건은 하위 NVM/VROM done 경로를 확인해야 함

### 1.2 sys_fsm_nvm.v

File:
- source/rtl/peri/sys/sys_fsm_nvm.v

Decode:
- r_nvm_fsm = 4'h1 = NVM____VROM_LOAD_REQ

중요 전이:
- NVM____VROM_LOAD_REQ -> NVM____VROM_LOAD_DON 조건: w_vrom_don == 1
- w_vrom_don은 sys_fsm_nvm_vrom.o_vrom_don에서 옴

즉 현재 NVM은 VROM done을 기다리는 중이다.

### 1.3 sys_fsm_nvm_vrom.v

File:
- source/rtl/peri/sys/sys_fsm_nvm_vrom.v

Decode:
- r_vrom_ctrl_fsm = 3'h3 = VROM_CTL_DMA_LV2
- r_vrom_lv2_fsm = 8'h9E = VROM_UCS__9E
- r_vrom_lv3_fsm = 8'h00 = VROM_LV3_IDL

중요 전이:
- VROM_CTL_DMA_LV2 -> VROM_CTL_DMA_LV3 조건: w_vrom_lv2_don & i_vrom_req
- r_vrom_lv2_fsm VROM_UCS__9E -> VROM_LV2__48 조건: i_vrom1_default_load_dma_end == 1
- w_vrom_lv2_don 조건: r_vrom_lv2_fsm == VROM_LV2__FC && i_vrom1_default_load_dma_end

즉 현재 VROM은 LV2 첫 DMA command인 9E transfer의 완료 pulse를 기다리는 중이다.

### 1.4 dma_core_slv.v

File:
- source/rtl/peri/dma/dma_core_slv.v

Decode:
- r_slv_fsm = 3'h6 = SLV___EXE__ACT
- r_exe_fsm = 5'h0E = EXE____3___ACT

중요 전이:
- EXE____3___ACT -> EXE____3__DONE 조건: i_exe3_end == 1
- SLV___EXE__ACT -> SLV___EXE_DONE 조건 중 하나: r_exe_fsm == EXE____3__DONE

즉 DMA slave는 exe3 동작이 끝나기를 기다리는 중이다.

### 1.5 dma_core_mst.v

File:
- source/rtl/peri/dma/dma_core_mst.v

Decode:
- r_pop_fsm = 2'h0 = POP______IDLE
- r_rdat_fsm = 3'h1 = MST_RDAT0_BURST_SRT
- r_radr_fsm = 3'h2 = MST_RADR0________DONE
- r_wdat_fsm = 4'h0 = MST_WDAT___________IDLE
- r_wadr_fsm = 4'h1 = MST_WADR________READY

중요 전이:
- MST_RADR0________DONE -> IDLE 조건: w_rdat0_end == 1
- MST_RDAT0_BURST_SRT 유지 조건: w_rdat_end가 아직 0
- w_rdat_end = axi4_mst_dma.rvalid && axi4_mst_dma.rlast && r_axi4_mst_rready_ && w_rdat_cnt_end
- MST_WADR________READY -> MST_WADR0_________ADR 조건: |w_fifo_lp != 0
- r_wdat_fsm IDLE은 아직 write data transfer가 시작되지 않았음을 의미

즉 DMA master는 read address는 이미 한 번 issue 완료 상태이고, read data burst 완료를 기다리며, write side는 FIFO에 데이터가 쌓이기를 기다리는 상태로 해석된다.

---

## 2. 현재 가장 가능성이 높은 대기 지점

현재 raw 상태를 RTL 전이 조건으로 연결하면 다음 chain이다.

1. sys_fsm = SYS_RST_VROM
2. nvm_fsm = NVM_VROM_LOAD_REQ
3. vrom_ctrl_fsm = VROM_CTL_DMA_LV2
4. vrom_lv2_fsm = VROM_UCS__9E
5. lv2 9E는 i_vrom1_default_load_dma_end를 기다림
6. dma_core_slv는 EXE3 active에서 i_exe3_end를 기다림
7. dma_core_mst는 read data burst completion을 기다림

따라서 1차 hypothesis:
- VROM 9E DMA가 시작되었고, read address는 issue되었지만 AXI read data channel completion이 오지 않아 DMA end가 발생하지 않는다.
- 그 결과 i_vrom1_default_load_dma_end가 발생하지 않고, VROM LV2 FSM이 8'h9E에 머물며, sys_fsm은 SYS_RST_VROM에 머문다.

주의:
- 이것은 아직 root cause가 아니라 현재 RTL state로부터 도출한 wait-chain hypothesis이다.
- 실제 root cause는 clock/reset/AXI response/VROM source path 중 어느 boundary에서 끊겼는지 waveform으로 확인해야 한다.

---

## 3. 진행 Task

### Task 1: reset/input/OSC 타이밍 확인

Objective:
- FSM이 멈춘 것이 실제 hang인지, reset release / pad input settle / OSC clock start 전 정상 wait인지 구분한다.

Files:
- Read: init.vcd
- Inspect waveform/FSDB: TO_RESX_SPROUT, *_Y, OSC_BIAS_EN, OSC_CLK_EN, TO_OSC_CLK, clk_sys, clk_axi_sys, rstb_bus, rstb_osc_wakeup, rstb_sof

Steps:
1. init.vcd에서 TO_RESX_SPROUT 및 *_Y transition time을 표로 정리한다.
2. 실제 FSDB/VCD에서 OSC_BIAS_EN/OSC_CLK_EN assert time을 확인한다.
3. 실제 TO_OSC_CLK first toggle time을 확인한다.
4. clk_sys / clk_axi_sys가 언제부터 toggle하는지 확인한다.
5. FSM snapshot time이 clock start 후 충분히 지난 시점인지 확인한다.

Pass criteria:
- FSM snapshot이 clock 없는 구간이면 정상 wait로 분류
- clock start 후에도 멈춰 있으면 Task 2로 진행

### Task 2: VROM 9E DMA request 확인

Objective:
- r_vrom_lv2_fsm=8'h9E에서 DMA start pulse와 DMA parameter가 정상 생성되었는지 확인한다.

Signals:
- r_vrom_ctl_fsm
- r_vrom_lv2_fsm
- o_vrom1_default_load_dma_srt
- o_vrom1_default_load_dma_src_adr
- o_vrom1_default_load_dma_des_adr
- o_vrom1_default_load_dma_trn_siz
- i_vrom1_default_load_dma_end

Expected:
- VROM_CTL_DMA_LV2 진입 시 r_vrom_lv2_fsm이 VROM_UCS__9E로 이동
- 해당 이동 시 o_vrom1_default_load_dma_srt pulse 발생
- DMA end가 돌아오면 다음 LV2 command VROM_LV2__48로 이동

### Task 3: DMA slave exe3 boundary 확인

Objective:
- dma_core_slv가 EXE3 active에서 무엇을 기다리는지 확인한다.

Signals:
- r_slv_fsm
- r_exe_fsm
- i_cdc_exe3_srt
- w_exe3_ack
- o_exe3_srt
- i_exe3_end
- o_exe3_done

Expected:
- r_exe_fsm=EXE____3___ACT 상태에서는 i_exe3_end가 1이 되어야 DONE으로 이동
- i_exe3_end가 안 오면 exe3 대상 block이 root-cause boundary 후보

### Task 4: DMA master read data boundary 확인

Objective:
- read address issue 이후 read data response가 정상인지 확인한다.

Signals:
- r_radr_fsm
- r_rdat_fsm
- axi4_mst_dma.arvalid
- axi4_mst_dma.arready
- axi4_mst_dma.araddr
- axi4_mst_dma.arlen
- axi4_mst_dma.rvalid
- axi4_mst_dma.rready
- axi4_mst_dma.rlast
- r_rdat_cnt
- w_rdat_cnt_end
- w_rdat_end
- w_rdat0_end

Expected:
- 현재 상태 r_radr_fsm=MST_RADR0_DONE, r_rdat_fsm=MST_RDAT0_BURST_SRT이면 rvalid/rlast/rready/count end 조합이 필요
- rvalid가 안 오면 AXI read target/VROM source response 문제
- rready가 0이면 FIFO full 또는 read data accept path 문제
- rlast/count mismatch면 burst length/count 문제

### Task 5: DMA write/FIFO boundary 확인

Objective:
- read data가 FIFO에 들어오지 않아서 write side가 대기하는지 확인한다.

Signals:
- r_wadr_fsm
- r_wdat_fsm
- w_fifo_lp
- w_fifo_push
- w_fifo_pop
- r_axi4_mst_awvalid
- axi4_mst_dma.awready
- r_axi4_mst_wvalid_
- axi4_mst_dma.wready
- axi4_mst_dma.bvalid

Expected:
- r_wadr_fsm=MST_WADR_READY는 |w_fifo_lp를 기다림
- r_wdat_fsm=IDLE은 write transfer 미시작
- FIFO push가 없으면 read data side 문제 가능성 강화

### Task 6: root cause 판정

Objective:
- wait-chain 중 실제로 끊긴 boundary를 하나로 좁힌다.

Decision table:
- clock 없음: reset/OSC sequencing issue 또는 정상 wait
- clk 있음, VROM dma_srt 없음: VROM request generation issue
- dma_srt 있음, exe3_srt 없음: CDC/slave dispatch issue
- exe3_srt 있음, DMA master arvalid 없음: DMA master start issue
- arvalid/arready 완료, rvalid 없음: AXI read target/VROM source response issue
- rvalid 있음, rready 없음: FIFO/write backpressure issue
- rvalid/rready 있음, rlast 없음: AXI target burst completion issue
- read complete 있음, write side bvalid 없음: AXI write target response issue
- DMA end 있음, VROM end 없음: DMA end CDC/connection issue

---

## 4. 현재 필요한 추가 waveform 신호 목록

Reset/Clock:
- TO_RESX_SPROUT
- TO_PORB_VDD
- OSC_BIAS_EN
- OSC_CLK_EN
- TO_OSC_CLK
- clk_wakeup
- clk_sys
- clk_axi_sys
- clk_axi_sys_lp
- rstb_bus
- rstb_sof
- rstb_osc_wakeup
- rstb_slp

SYS/NVM/VROM:
- r_sys_fsm
- r_nvm_fsm
- r_vrom_ctl_fsm
- r_vrom_lv2_fsm
- r_vrom_lv3_fsm
- i_vrom_req
- w_vrom_don
- i_vrom1_default_load_dma_end
- o_vrom1_default_load_dma_srt
- o_vrom1_default_load_dma_src_adr
- o_vrom1_default_load_dma_des_adr
- o_vrom1_default_load_dma_trn_siz

DMA slave:
- r_slv_fsm
- r_exe_fsm
- i_cdc_exe3_srt
- o_exe3_srt
- i_exe3_end
- o_exe3_done
- o_dma__srt
- i_dma__end

DMA master AXI:
- r_radr_fsm
- r_rdat_fsm
- r_wadr_fsm
- r_wdat_fsm
- r_pop_fsm
- axi4_mst_dma.arvalid
- axi4_mst_dma.arready
- axi4_mst_dma.araddr
- axi4_mst_dma.arlen
- axi4_mst_dma.rvalid
- axi4_mst_dma.rready
- axi4_mst_dma.rlast
- axi4_mst_dma.awvalid
- axi4_mst_dma.awready
- axi4_mst_dma.wvalid
- axi4_mst_dma.wready
- axi4_mst_dma.wlast
- axi4_mst_dma.bvalid
- axi4_mst_dma.bready
- w_fifo_lp
- w_fifo_push
- w_fifo_pop

---

## 5. 다음 실행 명령/방법

1. 현재 문서 기준으로 FSDB/VCD에서 신호를 추가 dump한다.
2. 가능하면 xrun/xmsim에서 위 신호를 probe 추가 후 재시뮬레이션한다.
3. 생성된 waveform에서 다음 순서로 확인한다.
   - reset/clock valid
   - VROM 9E dma_srt
   - DMA slave exe3 start/end
   - DMA master AR handshake
   - DMA master R handshake and RLAST
   - FIFO push/write path
   - DMA end -> VROM end return

---

## 6. 현재 결론

추가 possible-cause 분석 문서:
- docs/plans/logic_scan_top_possible_cause_analysis.md

현재 RTL state decode 기준으로는 sys_fsm 자체가 원인이 아니라, SYS_RST_VROM 단계에서 VROM LV2 첫 command 9E DMA 완료를 기다리는 상태다.

가장 직접적인 wait point는:
- sys_fsm_nvm_vrom.v: r_vrom_lv2_fsm = VROM_UCS__9E, waiting for i_vrom1_default_load_dma_end

그 아래 관측된 DMA 상태는:
- dma_core_slv.v: EXE____3___ACT, waiting for i_exe3_end
- dma_core_mst.v: read side active/done-wait and write side FIFO-wait

따라서 다음 분석의 1순위는 AXI read data channel에서 rvalid/rready/rlast 및 FIFO push가 정상인지 확인하는 것이다.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
