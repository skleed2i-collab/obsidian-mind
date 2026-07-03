---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: Claude DMA Debug Work Instruction"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/claude_dma_debug_work_instruction.md
imported_at: 2026-07-03 10:19:28 KST
---

# Claude 지속 Debugging 작업내역 지시서

작성 시각: 2026-06-04 19:59:59 KST
작업 위치: `/home/tim/pjt_work/D2I1002/top`
대상 프로젝트: D2I1002 top RTL / DMA / EXE3 hang debugging

---

## 1. Claude에게 줄 기본 지시

이 문서는 Claude에서 이어서 debugging하기 위한 작업내역 및 지시서이다.

Claude는 아래 내용을 기준으로 RTL을 직접 확인하고, 기존 가설을 검증하며, 가능하면 simulation 재현용 testbench 또는 fault-injection 조건을 구체화해야 한다.

중요 원칙:

1. 추측만 하지 말고 RTL source를 직접 읽고 근거 line을 확인한다.
2. FSM raw value가 있으면 반드시 RTL의 `localparam`/`parameter` state name으로 먼저 decode한다.
3. top FSM에서 child FSM, DMA slave, DMA master, AXI handshake까지 wait-chain을 구성한다.
4. 실제 IC hang 원인과 simulation 재현 조건을 분리해서 정리한다.
5. `timeout`, `retry`, `abort` 경로가 없는지 확인하고, 없으면 어떤 handshake에서 영구 대기하는지 명확히 적는다.
6. 정상 완료 test와 hang 재현 fault test를 모두 설계한다.
7. 결과는 markdown 문서와 필요한 경우 SystemVerilog testbench/fault model 파일로 남긴다.

---

## 2. 현재까지 확인한 파일 및 환경

현재 작업 폴더:

```text
/home/tim/pjt_work/D2I1002/top
```

환경 파일:

```text
environment.env
```

주의:

```text
사용자가 처음 요청한 파일명은 environmet.env였지만 실제 존재 파일은 environment.env이다.
```

RTL filelist:

```text
source/rtl/rtl.f
```

이미 작성된 분석 문서:

```text
dma_hang_ic_analysis.md
```

이 문서에는 현재까지 분석한 DMA/EXE3 hang 가능 시나리오, AXI handshake별 원인, FSM monitor 판정표, ILA 확인 신호, bring-up 우선순위가 포함되어 있다.

Claude는 먼저 아래 파일들을 읽어야 한다.

```bash
pwd
sed -n '1,200p' environment.env
sed -n '1,240p' source/rtl/rtl.f
sed -n '1,260p' dma_hang_ic_analysis.md
```

그 다음 `rtl.f`에 포함된 실제 RTL 중 다음 module/file을 찾아 읽어야 한다.

```text
dma_core_slv.v
dma_core_mst.v
EXE3 관련 module
EXE3 DMA start/end handshake를 만드는 wrapper/top module
AXI master interface 정의 또는 interface module
readback/monitor register mux가 있다면 ro_dma_core_* 신호 연결부
```

검색 예시:

```bash
grep -R "module dma_core_slv" -n source/rtl source/model
 grep -R "module dma_core_mst" -n source/rtl source/model
 grep -R "EXE____3___ACT\|i_exe3_end\|o_exe3_dma_end\|o_dma_end" -n source/rtl source/model
 grep -R "ro_dma_core_mst\|ro_dma_core_slv" -n source/rtl source/model
```

---

## 3. 현재까지의 핵심 분석 요약

현재 RTL 구조상 `timeout`, `retry`, `abort` 경로가 발견되지 않았다.

따라서 특정 AXI handshake 또는 EXE 완료 신호가 오지 않으면 FSM이 해당 상태에서 영구 대기할 가능성이 있다.

가장 유력한 wait-chain은 다음과 같다.

```text
EXE3 실행 시작
  -> EXE3가 DMA 요청 발생
  -> dma_core_slv가 DMA master start 발생
  -> dma_core_mst가 AXI master transaction 수행
  -> AXI write/read response 중 하나가 오지 않음
  -> dma_core_mst의 o_dma_end 미발생
  -> dma_core_slv의 o_exe3_dma_end 미발생
  -> EXE3의 i_exe3_end 미발생
  -> dma_core_slv가 EXE____3___ACT 상태에서 고정
```

`dma_core_slv.v`에서 EXE3 완료 조건은 다음 형태로 확인되었다.

```verilog
EXE____3___ACT : w_exe_nxt = (i_exe3_end) ? EXE____3__DONE : EXE____3___ACT;
```

즉 `i_exe3_end`가 오지 않으면 `EXE____3___ACT`에서 빠져나오지 않는다.

`dma_core_mst.v`에서 DMA 완료는 write data FSM 완료에 의존하는 것으로 확인되었다.

```verilog
assign o_dma_end = r_wdat_fsm==MST_WDAT___________DONE;
```

따라서 write response가 오지 않으면 `o_dma_end`가 발생하지 않고, EXE3 완료까지 막힐 수 있다.

---

## 4. 우선 검증해야 할 hang 시나리오

### 4.1 NO_BVALID fault

가장 먼저 simulation으로 재현해야 할 조건이다.

조건:

```text
AWREADY = 1
WREADY  = 1
BVALID  = 0 forever
```

의도:

```text
AW handshake 완료
W handshake 완료
WLAST 완료
BVALID 미발생
```

예상 결과:

```text
r_wdat_fsm == *_BRE_* 상태 고정
o_dma_end == 0
o_exe3_dma_end == 0
i_exe3_end == 0
r_exe_fsm == EXE____3___ACT 고정
```

이 조건은 destination slave/interconnect가 write response를 돌려주지 않는 IC bring-up 상황을 모델링한다.

### 4.2 NO_AWREADY fault

조건:

```text
AWREADY = 0 forever
WREADY  = don't care
BVALID  = 0
```

예상 결과:

```text
AWVALID = 1, AWREADY = 0 지속
r_wadr_fsm == *_ADR 상태 고정
o_dma_end 미발생
EXE3 hang
```

### 4.3 NO_WREADY fault

조건:

```text
AWREADY = 1
WREADY  = 0 forever
BVALID  = 0
```

예상 결과:

```text
AW handshake 완료
WVALID = 1, WREADY = 0 지속
r_wdat_fsm == write data transfer state 고정
o_dma_end 미발생
EXE3 hang
```

### 4.4 NO_ARREADY fault

조건:

```text
ARREADY = 0 forever
RVALID  = 0
```

예상 결과:

```text
ARVALID = 1, ARREADY = 0 지속
r_radr_fsm == *_ADR 상태 고정
FIFO에 source read data 미유입
write side 대기
o_dma_end 미발생
EXE3 hang
```

### 4.5 NO_RVALID fault

조건:

```text
ARREADY = 1
RVALID  = 0 forever
```

예상 결과:

```text
AR handshake 완료
RVALID 미발생
r_rdat_fsm == read data state 고정
FIFO push 없음
o_dma_end 미발생
EXE3 hang
```

### 4.6 NO_RLAST fault

조건:

```text
ARREADY = 1
RVALID  = 1
RREADY  = 1
RLAST   = 0 forever
```

예상 결과:

```text
read beat는 들어오지만 RLAST 없음
w_rdat_end 미발생
r_rdat_fsm 완료 안 됨
DMA end 미발생
```

### 4.7 DMA_END_PULSE_MISS fault

AXI는 정상 완료시키고 EXE3가 DMA 완료 pulse를 놓치는 조건을 만든다.

조건 예:

```text
o_exe3_dma_end pulse 발생 cycle에 EXE3 reset assert
o_exe3_dma_end pulse 발생 cycle에 EXE3 clock gate
또는 EXE3 sampling enable을 0으로 둠
```

예상 결과:

```text
o_exe3_dma_end는 발생
i_exe3_end는 미발생
r_exe_fsm == EXE____3___ACT 고정
```

---

## 5. Simulation testbench 요구사항

Claude는 가능하면 다음 형태의 directed testbench를 구성하거나, 기존 testbench에 fault mode를 추가한다.

권장 testbench 이름:

```text
verif/tb/tb_exe3_dma_hang.sv
```

권장 AXI fault model 이름:

```text
verif/tb/axi_fault_slave_model.sv
```

fault mode plusarg:

```text
+FAULT=NONE
+FAULT=NO_BVALID
+FAULT=NO_AWREADY
+FAULT=NO_WREADY
+FAULT=NO_ARREADY
+FAULT=NO_RVALID
+FAULT=NO_RLAST
+FAULT=DMA_END_PULSE_MISS
```

정상 test와 hang 재현 test의 pass/fail 기준은 다르게 둔다.

정상 test:

```text
PASS if o_exe3_done == 1 within N cycles
FAIL if timeout before o_exe3_done
```

hang 재현 test:

```text
PASS if o_exe3_done == 0 for N cycles
     and expected FSM state is observed
FAIL if o_exe3_done == 1
FAIL if unexpected FSM state observed
FAIL if X propagation으로 상태 판정 불가
```

NO_BVALID 예시 pass 조건:

```text
o_exe3_done == 0
r_exe_fsm == EXE____3___ACT
r_wdat_fsm in one of *_BRE_* states
bready == 1 또는 response 대기 상태
bvalid == 0
```

---

## 6. AXI fault slave model pseudo-code

Claude는 실제 interface 이름과 signal polarity를 RTL에서 확인한 후 아래 pseudo-code를 맞게 변환한다.

```verilog
localparam FAULT_NONE       = 0;
localparam FAULT_NO_BVALID  = 1;
localparam FAULT_NO_AWREADY = 2;
localparam FAULT_NO_WREADY  = 3;
localparam FAULT_NO_ARREADY = 4;
localparam FAULT_NO_RVALID  = 5;
localparam FAULT_NO_RLAST   = 6;

always_comb begin
  awready = normal_awready;
  wready  = normal_wready;
  bvalid  = normal_bvalid;
  bresp   = 2'b00;

  arready = normal_arready;
  rvalid  = normal_rvalid;
  rlast   = normal_rlast;
  rresp   = 2'b00;

  case (fault_mode)
    FAULT_NO_BVALID: begin
      awready = 1'b1;
      wready  = 1'b1;
      bvalid  = 1'b0;
    end

    FAULT_NO_AWREADY: begin
      awready = 1'b0;
      bvalid  = 1'b0;
    end

    FAULT_NO_WREADY: begin
      awready = 1'b1;
      wready  = 1'b0;
      bvalid  = 1'b0;
    end

    FAULT_NO_ARREADY: begin
      arready = 1'b0;
      rvalid  = 1'b0;
    end

    FAULT_NO_RVALID: begin
      arready = 1'b1;
      rvalid  = 1'b0;
    end

    FAULT_NO_RLAST: begin
      arready = 1'b1;
      rvalid  = normal_rvalid;
      rlast   = 1'b0;
    end
  endcase
end
```

주의:

```text
NO_BVALID는 AXI protocol 정상 동작이 아니다.
하지만 실제 IC에서 clock/reset/power/address decode/firewall 문제로 response가 사라지는 상황을 재현하기 위한 fault model이다.
```

---

## 7. 반드시 waveform에 넣을 신호

Clock/reset:

```text
clk_axi_sys
clk_axi_sys_lp
reset_n 또는 rst_n 계열
EXE3 clock/reset이 별도이면 해당 clock/reset
```

DMA slave / EXE:

```text
r_exe_fsm
r_slv_fsm
o_exe3_srt
o_exe3_done
i_exe3_end
o_exe3_dma_end
i_exe3_dma_srt
i_exe3_dma_zero_srt
i_exe3_dma_user_srt
o_dma__srt
o_dma_zero_srt
o_dma_user_srt
o_dma_end
```

DMA master FSM:

```text
r_radr_fsm
r_rdat_fsm
r_wadr_fsm
r_wdat_fsm
r_pop_fsm
```

AXI write channel:

```text
awvalid
awready
awaddr
awlen
awsize
wvalid
wready
wlast
wstrb
bvalid
bready
bresp
```

AXI read channel:

```text
arvalid
arready
araddr
arlen
arsize
rvalid
rready
rlast
rresp
```

FIFO/backpressure:

```text
w_fifo_lp
FIFO empty/full 관련 signal
push/pop 관련 signal
read data valid 관련 signal
```

---

## 8. FSM 판정표

Claude는 RTL에서 실제 state encoding을 찾아 아래 판정표를 구체화해야 한다.

```text
slv_exe_fsm == EXE____3___ACT
  -> EXE3 완료를 기다리는 중

mst_wdat_fsm == *_BRE_*
  -> BVALID 대기
  -> destination write response 문제 가능성 높음

mst_wdat_fsm == *_SRT 또는 *_BURST_* 또는 *_SINGL_*
  -> WREADY/FIFO/WLAST/count 문제 가능성

mst_wadr_fsm == *_ADR
  -> AWREADY 대기
  -> destination address channel 문제 가능성

mst_wadr_fsm == MST_WADR________READY
  -> FIFO data 대기
  -> source read 또는 FIFO push 문제 가능성

mst_radr_fsm == *_ADR
  -> ARREADY 대기
  -> source address channel 문제 가능성

mst_rdat_fsm == *_BURST_* 또는 *_SINGL_*
  -> RVALID/RLAST/RREADY 대기
  -> source read data response 문제 가능성

mst_pop_fsm == POP_FIFO_EMPT
  -> write side가 FIFO data 부족으로 대기
```

---

## 9. Claude가 수행해야 할 구체 작업

### Step 1. RTL 위치 확인

`source/rtl/rtl.f`에서 DMA/EXE3 관련 RTL 파일 경로를 찾는다.

확인 항목:

```text
dma_core_slv.v 위치
dma_core_mst.v 위치
EXE3 관련 RTL 위치
AXI interface/wrapper 위치
monitor readback 위치
```

### Step 2. state definition 추출

다음 FSM의 state localparam/parameter를 표로 만든다.

```text
r_exe_fsm
r_slv_fsm
r_radr_fsm
r_rdat_fsm
r_wadr_fsm
r_wdat_fsm
r_pop_fsm
```

표 형식:

```text
FSM name | State name | Encoding | Next-state condition | Wait signal
```

### Step 3. wait-chain 검증

아래 wait-chain이 실제 RTL에서 맞는지 line 근거와 함께 확인한다.

```text
r_exe_fsm EXE____3___ACT
  waits for i_exe3_end
    generated by EXE3 internal done condition
      may depend on o_exe3_dma_end
        generated by dma_core_slv from o_dma_end
          generated by dma_core_mst r_wdat_fsm DONE
            waits for AXI BVALID or read/write completion
```

### Step 4. simulation fault mode 작성

최소한 NO_BVALID fault를 만들고, 가능하면 NO_AWREADY/NO_WREADY/NO_ARREADY/NO_RVALID/NO_RLAST까지 확장한다.

우선순위:

```text
1. NO_BVALID
2. NORMAL_DMA
3. NO_ARREADY 또는 NO_RVALID
4. NO_WREADY
5. NO_AWREADY
6. NO_RLAST
7. DMA_END_PULSE_MISS
```

### Step 5. regression command 작성

xrun 또는 vcs 기준으로 실행 command를 작성한다.

예시 형태:

```bash
xrun -64bit -sv -f source/rtl/rtl.f \
  verif/tb/axi_fault_slave_model.sv \
  verif/tb/tb_exe3_dma_hang.sv \
  +FAULT=NO_BVALID \
  +access+rwc \
  -l logs/tb_exe3_dma_hang_no_bvalid.log
```

또는 VCS:

```bash
vcs -full64 -sverilog -f source/rtl/rtl.f \
  verif/tb/axi_fault_slave_model.sv \
  verif/tb/tb_exe3_dma_hang.sv \
  -debug_access+all \
  -l logs/compile_tb_exe3_dma_hang.log
./simv +FAULT=NO_BVALID -l logs/run_tb_exe3_dma_hang_no_bvalid.log
```

실제 compile option은 `environment.env`, 기존 `bin/compile.py`, 기존 regression script를 보고 맞춘다.

### Step 6. 결과 문서 작성

최종적으로 다음 문서를 생성 또는 갱신한다.

```text
dma_hang_ic_analysis.md
claude_dma_debug_work_instruction.md
```

가능하면 추가로 생성:

```text
docs/plans/dma_exe3_hang_sim_reproduction_plan.md
verif/tb/axi_fault_slave_model.sv
verif/tb/tb_exe3_dma_hang.sv
```

---

## 10. 확인해야 할 실제 IC bring-up readback 항목

가능하면 RT/register readback으로 다음 값을 확보한다.

```text
ro_dma_core_slv_exe_fsm
ro_dma_core_slv_slv_fsm
ro_dma_core_mst_radr_fsm
ro_dma_core_mst_rdat_fsm
ro_dma_core_mst_wadr_fsm
ro_dma_core_mst_wdat_fsm
ro_dma_core_mst_pop_fsm
```

같이 확보하면 좋은 값:

```text
DMA source address
DMA destination address
DMA transfer length
EXE3 start/done status
AXI error/status register가 있다면 해당 값
clock/reset/power domain status
firewall/security setting
```

판정 우선순위:

```text
wdat_fsm이 *_BRE_*이면 BVALID 미발생/destination write response 문제를 최우선으로 본다.
radr_fsm 또는 rdat_fsm이 멈춰 있으면 source read path 문제를 먼저 본다.
o_exe3_dma_end가 나왔는데 i_exe3_end가 안 나오면 DMA가 아니라 EXE3 내부 완료 조건 또는 pulse miss 문제다.
```

---

## 11. 예상 산출물

Claude가 이어서 작업한 뒤 남겨야 할 산출물:

```text
1. RTL 근거 line이 포함된 DMA/EXE3 wait-chain 분석
2. FSM state encoding decode 표
3. AXI fault simulation 조건표
4. NO_BVALID 재현 testbench 또는 기존 TB 수정 patch
5. xrun/vcs 실행 command
6. 정상 DMA와 fault DMA의 pass/fail 기준
7. waveform probe list
8. 실제 IC readback 값이 들어왔을 때 해석하는 decision table
```

---

## 12. 가장 중요한 결론

현재 가장 유력하고 먼저 재현해야 할 조건은 다음이다.

```text
NO_BVALID fault:
  AWREADY = 1
  WREADY  = 1
  BVALID  = 0 forever
```

이 조건에서 다음이 관측되면 simulation 재현 성공이다.

```text
r_wdat_fsm == *_BRE_* 상태 고정
o_dma_end == 0
o_exe3_dma_end == 0
i_exe3_end == 0
r_exe_fsm == EXE____3___ACT 고정
o_exe3_done == 0
```

이 결과는 실제 IC에서 destination slave/interconnect가 write response를 반환하지 않을 때 DMA/EXE3가 timeout 없이 영구 hang될 수 있음을 보여주는 직접적인 재현 근거가 된다.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
