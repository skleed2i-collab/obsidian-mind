---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: DMA Hang IC Analysis"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/dma_hang_ic_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# 실제 IC에서 DMA/EXE3 Hang 발생 가능 시나리오

## 1. 요약

현재 RTL 구조상 `timeout`, `retry`, `abort` 경로가 없습니다.

따라서 특정 AXI handshake 또는 EXE 완료 신호가 오지 않으면 FSM이 해당 상태에서 영구 대기할 수 있습니다.

가장 가능성 높은 hang 경로는 다음입니다.

```text
EXE3 실행 시작
  -> EXE3가 DMA 요청 발생
  -> dma_core_mst가 AXI master transaction 수행
  -> AXI write/read response 중 하나가 오지 않음
  -> dma_core_mst의 o_dma_end 미발생
  -> dma_core_slv의 o_exe3_dma_end 미발생
  -> EXE3가 i_exe3_end 미발생
  -> dma_core_slv가 EXE____3___ACT 상태에서 고정
```

---

## 2. 완료 신호 흐름

EXE3 실행 중 완료 경로는 대략 다음과 같습니다.

```text
dma_core_slv:
  r_exe_fsm == EXE____3___ACT

EXE3 내부:
  i_exe3_dma_srt / i_exe3_dma_zero_srt / i_exe3_dma_user_srt 발생

DMA slave:
  o_dma__srt / o_dma_zero_srt / o_dma_user_srt 발생

DMA master:
  AXI read/write transaction 수행
  o_dma_end 발생

DMA slave:
  o_exe3_dma_end 발생

EXE3 내부:
  i_exe3_end 발생

DMA slave:
  r_exe_fsm -> EXE____3__DONE
  o_exe3_done 발생
```

`dma_core_slv.v` 기준 EXE3 완료 조건은 다음입니다.

```verilog
EXE____3___ACT : w_exe_nxt = (i_exe3_end) ? EXE____3__DONE : EXE____3___ACT;
```

즉 `i_exe3_end`가 올라오지 않으면 `EXE____3___ACT`에서 멈춥니다.

---

## 3. 가장 가능성 높은 원인: AXI B response 미발생

`dma_core_mst.v`에서 `o_dma_end`는 write data FSM이 DONE 상태가 되어야 발생합니다.

```verilog
assign o_dma_end = r_wdat_fsm==MST_WDAT___________DONE;
```

write FSM은 write response 상태에서 `axi4_mst_dma.bvalid`를 기다립니다.

대표 상태:

```text
MST_WDAT0_BURST_BRE_SRT
MST_WDAT0_SINGL_BRE_SRT
MST_WDAT1_BURST_BRE____
MST_WDAT1_SINGL_BRE____
MST_WDAT0_BURST_BRE____
MST_WDAT0_SINGL_BRE____
MST_WDAT_DIRECT_BRE____
```

이 상태들은 `bvalid`가 와야 다음으로 진행합니다.

따라서 실제 IC에서 다음 상황이면 hang이 발생할 수 있습니다.

```text
AW handshake 완료
W handshake 완료
WLAST 전송 완료
BVALID 미발생
  -> r_wdat_fsm이 *_BRE_* 상태에 고정
  -> o_dma_end 미발생
  -> o_exe3_dma_end 미발생
  -> i_exe3_end 미발생
  -> EXE3 hang
```

### 실제 IC 원인 후보

- destination address가 unmapped 영역
- destination slave clock이 꺼져 있음
- destination slave reset이 아직 풀리지 않음
- destination power domain이 off 상태
- interconnect route table 또는 address decode 오류
- security/firewall 권한 문제
- AXI default slave가 없거나 DECERR 응답을 만들지 않음
- AXI slave가 WLAST 수신 후 BVALID를 내지 않는 버그
- narrow write 또는 WSTRB 조합을 slave가 처리하지 못함
- interconnect 내부 deadlock

AXI protocol상 error 상황이어도 `BVALID`와 `BRESP`로 응답해야 합니다.
하지만 실제 SoC에서 address map, clock, reset, power, firewall 문제가 있으면 response가 아예 돌아오지 않아 master FSM이 영구 대기할 수 있습니다.

---

## 4. Source read side hang 가능성

DMA copy의 경우 source read data가 FIFO에 들어와야 destination write가 진행됩니다.

read address FSM은 `arready`를 기다립니다.

```text
MST_RADR0_________ADR
MST_RADR1_________ADR
```

read data FSM은 `rvalid`, `rlast`, `rready` 조건을 기다립니다.

```verilog
assign w_rdat_end = axi4_mst_dma.rvalid && axi4_mst_dma.rlast && r_axi4_mst_rready_ && w_rdat_cnt_end;
```

따라서 다음 상황이면 read side에서 hang이 발생할 수 있습니다.

```text
ARVALID 발생했지만 ARREADY 미발생
  -> r_radr_fsm이 *_ADR 상태에 고정

AR handshake 완료 후 RVALID 미발생
  -> r_rdat_fsm이 read data 상태에 고정

RVALID는 오지만 RLAST 미발생 또는 beat count mismatch
  -> w_rdat_end 미발생
  -> read FSM 미완료
```

### 실제 IC 원인 후보

- source address가 unmapped 영역
- source slave clock/reset/power 문제
- source slave가 read response를 내지 않음
- interconnect read path deadlock
- RLAST 누락
- ARLEN과 실제 RLAST 위치 불일치
- firewall/security block에 의해 read transaction 차단

---

## 5. FIFO empty로 인한 write 진행 불가

write address FSM에는 FIFO data를 기다리는 상태가 있습니다.

```verilog
MST_WADR________READY : w_wadr_nxt = (|w_fifo_lp) ? MST_WADR0_________ADR : MST_WADR________READY;
```

즉 일반 DMA copy에서 source read data가 FIFO에 들어오지 않으면 write address조차 진행하지 않습니다.

이 경우는 destination 문제가 아니라 source read 또는 FIFO push 쪽 문제일 가능성이 큽니다.

판정:

```text
r_wadr_fsm == MST_WADR________READY
w_fifo_lp == 0
  -> source read data가 FIFO에 안 들어오는 중
```

---

## 6. Write data channel hang 가능성

write data FSM이 data transfer 상태에서 멈출 수도 있습니다.

대표 상태:

```text
MST_WDAT0_BURST_____SRT
MST_WDAT0_SINGL_____SRT
MST_WDAT1_BURST________
MST_WDAT1_SINGL________
MST_WDAT0_BURST________
MST_WDAT0_SINGL________
```

가능한 원인:

- `WREADY`가 올라오지 않음
- FIFO empty로 `WVALID`를 만들 수 없음
- `WLAST` 조건이 맞지 않음
- write beat count가 기대값과 다름
- last beat의 WSTRB 조합이 slave에서 허용되지 않음

판정:

```text
WVALID = 1, WREADY = 0 지속
  -> destination slave write data backpressure 또는 hang

WVALID = 0, FIFO empty
  -> source read/FIFO 문제

WVALID/WREADY는 진행되지만 WLAST가 안 나옴
  -> write length/count mismatch 의심
```

---

## 7. EXE3 내부 완료 조건 누락 가능성

DMA master가 정상적으로 `o_dma_end`를 냈더라도 EXE3 내부가 최종 `i_exe3_end`를 올리지 않으면 `dma_core_slv`는 `EXE____3___ACT`에서 대기합니다.

확인할 신호:

```text
o_exe3_srt
o_exe3_dma_on
i_exe3_dma_srt
i_exe3_dma_zero_srt
i_exe3_dma_user_srt
o_exe3_dma_end
i_exe3_end
```

판정:

```text
o_exe3_dma_end 발생
하지만 i_exe3_end 미발생
  -> DMA 문제가 아니라 EXE3 내부 완료 FSM 문제

반대로 o_exe3_dma_end 미발생
  -> DMA master/AXI transaction 문제
```

### 실제 IC 원인 후보

- EXE3 내부 FSM 완료 조건 누락
- `o_exe3_dma_end` pulse를 EXE3가 놓침
- EXE3 clock/reset domain 문제
- EXE3가 DMA 완료 후 추가 조건을 기다리는데 그 조건이 오지 않음
- start/end pulse CDC 문제
- busy/end handshake 설계 오류

---

## 8. Clock/reset/low-power clock 관련 가능성

`dma_core_slv.v`에서는 일부 AXI slave output이 `clk_axi_sys_lp`로 구동됩니다.

예:

```verilog
r_axi4_slv_awready
r_axi4_slv_wready_
r_axi4_slv_bvalid_
```

반면 slave FSM과 execute FSM은 `clk_axi_sys`를 사용합니다.

두 clock이 완전히 synchronous이고 항상 같이 동작한다는 보장이 없다면 실제 IC에서만 문제가 생길 수 있습니다.

가능한 문제:

- `clk_axi_sys_lp`가 clock-gate되어 slave response가 나가지 않음
- `clk_axi_sys`와 `clk_axi_sys_lp` 사이 CDC 없이 FSM 신호를 사용
- reset release timing 차이
- low-power mode 진입/해제 중 handshake 손실
- scan/test mode에서 lp clock만 정지

이런 문제는 RTL simulation에서는 잘 보이지 않다가 실제 IC에서 hang처럼 보일 수 있습니다.

---

## 9. 실제 IC에서 우선 확인할 monitor FSM

가능하면 아래 monitor 값을 읽어야 합니다.

```text
ro_dma_core_slv_exe_fsm
ro_dma_core_slv_slv_fsm
ro_dma_core_mst_radr_fsm
ro_dma_core_mst_rdat_fsm
ro_dma_core_mst_wadr_fsm
ro_dma_core_mst_wdat_fsm
ro_dma_core_mst_pop_fsm
```

### FSM 상태별 1차 판정

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

## 10. ILA/logic analyzer에서 확인할 AXI 신호

최소 다음 신호를 잡는 것이 좋습니다.

### Write address channel

```text
awvalid
awready
awaddr
awlen
awsize
```

### Write data channel

```text
wvalid
wready
wlast
wstrb
```

### Write response channel

```text
bvalid
bready
bresp
```

### Read address channel

```text
arvalid
arready
araddr
arlen
arsize
```

### Read data channel

```text
rvalid
rready
rlast
rresp
```

---

## 11. AXI waveform 판정표

| 관측 상황 | 의심 원인 |
|---|---|
| `awvalid=1`, `awready=0` 지속 | destination/interconnect가 write address를 accept하지 않음 |
| AW handshake 완료, W 전송 완료, `bready=1`, `bvalid=0` 지속 | destination slave/interconnect가 write response 미발생 |
| `wvalid=1`, `wready=0` 지속 | destination slave write data backpressure/hang |
| `wvalid=0`, FIFO empty | source read 또는 FIFO push 문제 |
| `arvalid=1`, `arready=0` 지속 | source/interconnect가 read address를 accept하지 않음 |
| AR handshake 완료, `rvalid=0` 지속 | source slave read response 미발생 |
| `rvalid=1`은 오지만 `rlast=0` 지속 | RLAST 누락 또는 burst length mismatch |
| `rresp`/`bresp`가 SLVERR/DECERR | address/security/power/reset 문제 가능성 |
| `o_exe3_dma_end` 발생 후 `i_exe3_end` 미발생 | EXE3 내부 FSM/완료 조건 문제 |

---

## 12. 가장 현실적인 bring-up 원인 순위

실제 IC bring-up 기준으로 가능성이 높은 순서는 다음입니다.

1. Destination address가 잘못되어 B response가 안 돌아옴
2. Destination slave clock/reset/power가 준비되지 않음
3. Source address가 잘못되어 read response가 안 돌아옴
4. Interconnect default slave 또는 DECERR response 생성 문제
5. Firewall/security setting 문제
6. EXE3 내부에서 `o_exe3_dma_end` pulse를 놓침
7. EXE3 내부 완료 조건 누락
8. `clk_axi_sys` / `clk_axi_sys_lp` 관련 clock gating 또는 CDC 문제
9. AXI burst length, WLAST, RLAST mismatch
10. narrow write / WSTRB 조합을 target slave가 처리하지 못함

---

## 13. 결론

실제 IC에서 이 구조가 멈추는 가장 대표적인 시나리오는 다음입니다.

```text
1. EXE3가 DMA write를 시작함
2. DMA master가 AW/W transaction을 전송함
3. destination slave 또는 interconnect가 BVALID를 돌려주지 않음
4. dma_core_mst가 *_BRE_* 상태에서 영구 대기함
5. o_dma_end가 발생하지 않음
6. o_exe3_dma_end가 발생하지 않음
7. EXE3가 i_exe3_end를 발생시키지 못함
8. dma_core_slv가 EXE____3___ACT 상태에서 멈춤
```

따라서 칩에서 먼저 볼 것은 다음입니다.

```text
1. ro_dma_core_mst_wdat_fsm 값
2. bvalid/bready 상태
3. awvalid/awready, wvalid/wready/wlast 완료 여부
4. ro_dma_core_mst_radr_fsm/rdat_fsm 값
5. arvalid/arready, rvalid/rready/rlast 상태
6. o_exe3_dma_end와 i_exe3_end 관계
```

`wdat_fsm`이 `*_BRE_*`이면 destination write response 문제를 최우선으로 봐야 합니다.

`radr_fsm` 또는 `rdat_fsm`이 멈춰 있으면 source read path 문제를 먼저 봐야 합니다.

`o_exe3_dma_end`가 나왔는데 `i_exe3_end`가 안 나오면 DMA가 아니라 EXE3 내부 완료 조건 문제입니다.

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
