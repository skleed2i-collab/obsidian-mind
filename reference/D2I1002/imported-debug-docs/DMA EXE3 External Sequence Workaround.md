---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: DMA EXE3 External Sequence Workaround"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/dma_exe3_external_sequence_workaround.md
imported_at: 2026-07-03 10:19:28 KST
---

# DMA/EXE3 Hang 회피를 위한 외부 Sequence 방안

작성 시각: 2026-06-04 20:04:34 KST
작업 위치: `/home/tim/pjt_work/D2I1002/top`
대상: RTL 수정 없이 외부 register/program sequence로 DMA/EXE3 hang 가능성을 회피하는 방안

---

## 1. 전제

이 문서는 RTL 수정 없이 외부 sequence만으로 DMA/EXE3 hang 가능성을 낮추는 방법을 정리한다.

수정하지 않는 항목:

```text
RTL FSM 수정 없음
timeout/retry/abort logic 추가 없음
AXI master/slave protocol logic 수정 없음
EXE3 내부 done 조건 수정 없음
```

사용 가능한 회피 수단:

```text
register write/read sequence
reset sequence
clock enable sequence
power domain enable sequence
address/length 설정 순서
DMA/EXE3 start 전 사전 readback check
start 후 monitor polling
hang 감지 시 외부 reset/re-init sequence
```

핵심 방향:

```text
DMA/EXE3가 시작된 뒤에는 내부 FSM이 특정 AXI response를 영구 대기할 수 있다.
따라서 회피는 start 이후가 아니라 start 이전에 AXI response가 반드시 돌아올 조건을 만들어 놓는 것이다.
```

---

## 2. 왜 외부 sequence 회피가 가능한가

현재 hang의 핵심 원인은 RTL 계산 오류라기보다, DMA가 접근하는 외부 조건이 준비되지 않은 상태에서 transaction을 시작할 때 발생한다.

대표 wait-chain:

```text
EXE3 start
  -> DMA start
  -> AXI transaction 발행
  -> target/source가 response 미반환
  -> o_dma_end 미발생
  -> o_exe3_dma_end 미발생
  -> i_exe3_end 미발생
  -> EXE____3___ACT 고정
```

따라서 외부에서 다음 조건을 보장하면 hang 가능성을 크게 줄일 수 있다.

```text
1. AXI source/destination address가 유효하다.
2. source/destination slave clock이 살아 있다.
3. source/destination slave reset이 release되어 있다.
4. source/destination power domain이 ON이다.
5. interconnect/firewall/security 설정이 완료되어 있다.
6. target slave가 DECERR/SLVERR 포함 response를 반환할 수 있다.
7. DMA length/alignment/WSTRB 조합이 target에서 허용된다.
8. EXE3/DMA clock domain이 start pulse와 done pulse를 받을 수 있는 상태다.
```

즉 외부 sequence 회피란, DMA를 시작하기 전에 위 조건들을 register readback 또는 dummy access로 검증하는 것이다.

---

## 3. 권장 전체 sequence

아래 순서를 기본 bring-up / production sequence로 사용한다.

```text
SEQ0  Global/functional reset assert
SEQ1  Power domain enable
SEQ2  Clock source select 고정
SEQ3  Functional clock enable
SEQ4  Low-power clock enable
SEQ5  Reset release
SEQ6  Clock/reset status polling
SEQ7  Interconnect/firewall/security 설정
SEQ8  Source/Destination slave ready polling
SEQ9  Dummy read/write probe
SEQ10 DMA register 설정
SEQ11 DMA 설정 readback 검증
SEQ12 EXE3 start
SEQ13 DMA/EXE3 monitor polling
SEQ14 정상 완료 또는 hang 감지 시 recovery sequence
```

---

## 4. 상세 sequence

### SEQ0. Reset assert

권장:

```text
functional reset assert
DMA/EXE3 관련 reset assert
AXI fabric 관련 reset assert 가능하면 같이 assert
```

기술적 이유:

```text
이전 실패 transaction으로 FSM, FIFO, outstanding counter, response pending 상태가 남아 있을 수 있다.
start 전 reset을 통해 DMA master/slave FSM과 FIFO pointer를 초기 상태로 되돌린다.
```

회피 효과:

```text
이전 hang 이후 남은 partial AXI transaction 또는 내부 busy 상태를 제거한다.
```

---

### SEQ1. Power domain enable

권장:

```text
source slave power ON
destination slave power ON
interconnect/fabric power ON
DMA/EXE3 power ON
```

가능하면 power-good/status bit를 polling한다.

```text
while power_good != 1:
  wait
```

기술적 이유:

```text
AXI target power가 꺼져 있으면 AW/AR handshake 또는 B/R response가 돌아오지 않을 수 있다.
특히 BVALID/RVALID 미발생 hang은 power/reset/clock 미준비 상태에서 자주 발생한다.
```

회피 효과:

```text
NO_BVALID, NO_RVALID, NO_AWREADY, NO_ARREADY 계열 hang을 start 전에 차단한다.
```

---

### SEQ2. Clock source select 고정

권장:

```text
OSC/EXCLK select
scan/test mode select
clock mux select
low-power clock mux select
```

이 값들이 reset release 직후 흔들리지 않도록 먼저 고정한다.

기술적 이유:

```text
clock select pad 또는 mux select가 reset release 근처에서 흔들리면 DMA/EXE3 또는 AXI clock domain이 일부 cycle만 동작하거나 start/done pulse를 놓칠 수 있다.
```

회피 효과:

```text
EXE3가 o_exe3_dma_end pulse를 놓치는 문제, clk_axi_sys/clk_axi_sys_lp 간 초기 sequencing 문제를 줄인다.
```

---

### SEQ3. Functional clock enable

권장:

```text
clk_axi_sys enable
EXE3 clock enable
DMA master/slave clock enable
source/destination slave clock enable
interconnect clock enable
```

clock enable 후 충분한 stabilization wait를 둔다.

예:

```text
clock_enable = 1
wait 10~100 us 또는 설계 spec 기준 안정화 시간
```

기술적 이유:

```text
DMA가 AXI transaction을 발행해도 상대 slave clock이 멈춰 있으면 ready/valid response가 생성되지 않는다.
```

회피 효과:

```text
AWREADY/ARREADY/BVALID/RVALID 미발생을 방지한다.
```

---

### SEQ4. Low-power clock enable

권장:

```text
clk_axi_sys_lp enable
LP clock gate disable 또는 force-on
RT/register readback clock enable
```

기술적 이유:

```text
dma_core_slv.v에서는 일부 AXI slave response register가 clk_axi_sys_lp에서 구동되는 것으로 확인되었다.
clk_axi_sys는 살아 있어도 clk_axi_sys_lp가 멈춰 있으면 register access response 또는 내부 control path가 정상적으로 진행되지 않을 수 있다.
```

회피 효과:

```text
register write/read sequence 누락, start bit 미반영, status polling 불능, slave response 미발생 가능성을 줄인다.
```

---

### SEQ5. Reset release 순서

권장 reset release 순서:

```text
1. clock source/mux stable
2. fabric/interconnect reset release
3. source/destination slave reset release
4. DMA/EXE3 reset release
5. software-visible register block reset release
```

또는 SoC reset architecture에 맞춰 다음 원칙을 지킨다.

```text
transaction을 받을 target/fabric을 먼저 준비한다.
transaction을 발생시키는 master는 나중에 release한다.
```

기술적 이유:

```text
master가 먼저 살아나서 AXI transaction을 발행하면 target/fabric이 아직 reset 상태라 response가 사라질 수 있다.
```

회피 효과:

```text
start 직후 AWREADY/ARREADY/BVALID/RVALID가 오지 않는 상황을 방지한다.
```

---

### SEQ6. Clock/reset status polling

DMA/EXE3 start 전에 다음 status를 polling한다.

```text
clock stable == 1
clock gate open == 1
reset released == 1
power good == 1
PLL/OSC ready == 1, 해당 시
LP clock alive == 1
```

기술적 이유:

```text
register write가 가능하다는 것만으로 모든 clock domain이 살아 있다고 볼 수 없다.
특히 RT read path가 clk_axi_sys_lp로 동작하더라도 EXE3/DMA 내부 functional clock이 아직 안정화되지 않았을 수 있다.
```

회피 효과:

```text
clock/reset race로 인한 one-shot start pulse miss, done pulse miss, AXI response miss를 줄인다.
```

---

### SEQ7. Interconnect/firewall/security 설정

DMA start 전에 다음을 설정한다.

```text
DMA master ID 권한 허용
source address range read 허용
destination address range write 허용
secure/non-secure attribute 일치
privileged/user attribute 일치
cache/buffer/protection attribute 필요 시 설정
interconnect route/address decode 설정 완료
```

기술적 이유:

```text
firewall/security에서 transaction이 차단되면 정상적으로 DECERR/SLVERR가 돌아와야 하지만, 실제 IC bring-up 환경에서는 response가 사라지거나 interconnect 내부에서 대기하는 경우가 있다.
```

회피 효과:

```text
address decode/security 문제로 인한 NO_BVALID/NO_RVALID hang 가능성을 줄인다.
```

---

### SEQ8. Source/Destination slave ready polling

DMA source와 destination이 software-visible status를 제공하면 start 전에 반드시 확인한다.

예:

```text
source_ready == 1
destination_ready == 1
source_not_busy == 1
destination_not_busy == 1
memory_init_done == 1
rom/sram/eeprom/flash ready == 1
```

기술적 이유:

```text
DMA는 source에서 read data를 받아 FIFO에 넣고 destination으로 write한다.
source가 ready가 아니면 RVALID/RLAST가 오지 않고, destination이 ready가 아니면 AWREADY/WREADY/BVALID가 오지 않을 수 있다.
```

회피 효과:

```text
read side hang과 write side hang을 start 전에 줄인다.
```

---

### SEQ9. Dummy access probe

가장 중요한 회피 sequence이다.

DMA start 전에 CPU/RT/다른 master로 동일 target에 dummy access를 수행한다.

#### 9.1 Source dummy read

```text
read source_start_address
read source_end_address 또는 source_start_address + length - beat
check read response 정상
```

기술적 이유:

```text
source address가 decode되고, source slave가 RVALID/RLAST/RRESP를 반환할 수 있는지 DMA 시작 전에 확인한다.
```

회피 효과:

```text
DMA가 AR을 낸 뒤 RVALID/RLAST를 영구 대기하는 상황을 사전에 잡는다.
```

#### 9.2 Destination dummy write/readback

가능하면 scratch 영역에 write 후 readback한다.

```text
write destination_probe_address = test_pattern
read  destination_probe_address
compare test_pattern
```

readback이 불가능한 write-only target이면 최소 write response status를 확인한다.

기술적 이유:

```text
destination이 AW/W를 accept하고 BVALID/BRESP를 반환할 수 있는지 확인한다.
```

회피 효과:

```text
가장 유력한 NO_BVALID hang을 DMA start 전에 검출한다.
```

주의:

```text
dummy write는 side-effect 없는 scratch 주소에서만 수행한다.
FIFO, command register, clear-on-write register 등에는 적용하지 않는다.
```

---

### SEQ10. DMA register 설정

DMA 관련 register 설정 순서 권장:

```text
1. DMA/EXE3 status clear
2. pending interrupt clear
3. source address 설정
4. destination address 설정
5. transfer length 설정
6. burst/size/alignment 설정
7. mode 설정: copy/zero/user 등
8. interrupt/mask 설정, 필요 시
9. start bit는 가장 마지막에 write
```

기술적 이유:

```text
start bit가 먼저 들어가면 source/destination/length/mode가 이전 값 또는 partial update 값으로 사용될 수 있다.
잘못된 주소/길이로 transaction이 나가면 response 미반환 hang이 발생할 수 있다.
```

회피 효과:

```text
잘못된 address/length/mode 조합으로 인한 AXI hang을 방지한다.
```

---

### SEQ11. DMA 설정 readback 검증

start 전에 설정값을 readback한다.

```text
readback source address == programmed source address
readback destination address == programmed destination address
readback length == programmed length
readback mode == programmed mode
readback start/busy == idle
```

기술적 이유:

```text
register write path가 clk_axi_sys_lp 또는 bridge timing 문제로 누락되면 DMA가 의도하지 않은 주소로 동작할 수 있다.
```

회피 효과:

```text
register write 누락 또는 stale configuration으로 인한 잘못된 AXI transaction을 방지한다.
```

---

### SEQ12. EXE3 start

start는 모든 조건 확인 후 마지막에 수행한다.

권장:

```text
write start bit
immediately read status
confirm busy/start accepted
```

기술적 이유:

```text
start pulse가 clock/reset/enable 조건 때문에 놓이면 EXE3가 시작하지 않거나, 반대로 시작했지만 status tracking이 어긋날 수 있다.
```

회피 효과:

```text
start acceptance 여부를 초기에 확인하여 silent failure와 hang을 구분한다.
```

---

### SEQ13. Monitor polling

start 후 다음 status를 polling한다.

```text
o_exe3_done 또는 done status
busy status
error/status bit
ro_dma_core_slv_exe_fsm
ro_dma_core_mst_wdat_fsm
ro_dma_core_mst_radr_fsm
ro_dma_core_mst_rdat_fsm
ro_dma_core_mst_wadr_fsm
ro_dma_core_mst_pop_fsm
```

polling timeout은 외부 software에서 둔다.

```text
if done within timeout:
  PASS
else:
  read monitor FSM
  classify hang state
  run recovery sequence
```

기술적 이유:

```text
RTL 내부에는 timeout이 없으므로, timeout 판단은 외부 sequence가 담당해야 한다.
```

회피 효과:

```text
무한 대기를 system-level deadlock으로 방치하지 않고 외부에서 복구할 수 있다.
```

---

### SEQ14. Hang 감지 시 recovery sequence

RTL 내부 abort가 없다면 외부 recovery는 reset 기반으로 해야 한다.

권장 recovery:

```text
1. 추가 start 금지
2. monitor FSM/status dump
3. AXI/fabric error status dump
4. DMA/EXE3 local reset assert
5. 충분한 wait
6. DMA/EXE3 local reset release
7. status clear
8. config 재설정
9. dummy access probe 재수행
10. 필요 시 더 안전한 address/length로 재시도
```

만약 local reset으로 복구되지 않으면:

```text
fabric reset 또는 target slave reset 포함
power-cycle 필요 여부 판단
```

기술적 이유:

```text
AXI transaction이 이미 outstanding 상태로 남아 있으면 단순 register clear만으로는 FSM이 빠져나오지 못할 수 있다.
RTL abort 경로가 없다면 FSM 초기화를 위해 reset이 필요하다.
```

회피 효과:

```text
hang 상태를 장시간 유지하지 않고 외부 supervisor가 복구 경로로 전환할 수 있다.
```

---

## 5. 최소 권장 sequence 요약

시간이 없으면 최소한 아래만 적용한다.

```text
1. Power/clock/reset ready polling
2. Interconnect/firewall 설정 완료
3. Source dummy read
4. Destination dummy write/readback 또는 write response 확인
5. DMA config write
6. DMA config readback verify
7. EXE3 start
8. External timeout polling
9. Timeout 시 monitor dump 후 DMA/EXE3 reset
```

이 최소 sequence의 핵심은 다음이다.

```text
DMA가 AXI transaction을 처음 발생시키기 전에,
동일 source/destination이 정상 response를 반환하는지 다른 master로 먼저 검증한다.
```

---

## 6. 왜 이게 회피 방안인지 기술적 설명

### 6.1 NO_BVALID 회피

문제:

```text
DMA write가 AW/W를 완료했지만 BVALID가 오지 않으면 r_wdat_fsm이 *_BRE_*에서 멈춘다.
```

회피 sequence:

```text
destination power/clock/reset ready 확인
destination dummy write 수행
write response 정상 확인
그 후 DMA start
```

왜 회피인가:

```text
DMA보다 먼저 동일 destination path에 write transaction을 보내 BVALID/BRESP가 정상 반환되는지 확인한다.
B response가 없는 target이면 DMA를 시작하지 않으므로 r_wdat_fsm이 *_BRE_*로 들어가 영구 대기하는 상황을 방지한다.
```

---

### 6.2 NO_RVALID/NO_RLAST 회피

문제:

```text
DMA read가 AR을 완료했지만 RVALID/RLAST가 오지 않으면 read FSM이 완료되지 않는다.
```

회피 sequence:

```text
source power/clock/reset ready 확인
source dummy read 수행
read response 정상 확인
가능하면 start/end address 모두 probe
그 후 DMA start
```

왜 회피인가:

```text
DMA source path가 실제로 RVALID/RLAST를 반환할 수 있는지 사전에 확인한다.
source path가 막혀 있으면 DMA를 시작하지 않으므로 read FSM 영구 대기를 방지한다.
```

---

### 6.3 잘못된 address/length 회피

문제:

```text
잘못된 source/destination address 또는 length/alignment 때문에 unmapped 영역이나 미지원 WSTRB 조합으로 transaction이 나갈 수 있다.
```

회피 sequence:

```text
address range check
alignment check
length non-zero check
burst boundary check
WSTRB/size 허용 여부 check
config readback verify
```

왜 회피인가:

```text
AXI target이 처리할 수 없는 transaction을 DMA가 발행하지 않게 막는다.
response가 DECERR/SLVERR로 정상 반환되지 않는 SoC bring-up 상태에서도 hang 가능성을 줄인다.
```

---

### 6.4 clock/reset race 회피

문제:

```text
start 시점에 clk_axi_sys, clk_axi_sys_lp, EXE3 clock 중 일부가 안정화되지 않으면 start/done pulse 또는 AXI handshake가 손실될 수 있다.
```

회피 sequence:

```text
clock source select 고정
clock enable
stabilization wait
reset release
clock/reset status polling
그 후 register write/start
```

왜 회피인가:

```text
start pulse와 done pulse가 전달되는 모든 clock domain이 살아 있는 상태에서만 DMA/EXE3를 시작한다.
따라서 pulse miss와 partial handshake 가능성을 줄인다.
```

---

### 6.5 EXE3 done pulse miss 회피

문제:

```text
o_exe3_dma_end가 1-cycle pulse인데 EXE3가 해당 cycle에 reset/clock-gate/sampling-disable 상태면 i_exe3_end가 발생하지 않을 수 있다.
```

회피 sequence:

```text
EXE3 clock force-on
EXE3 reset release 확인
DMA start 전 EXE3 ready 확인
DMA 진행 중 EXE3 clock gate 금지
DMA 진행 중 low-power mode 진입 금지
```

왜 회피인가:

```text
DMA completion pulse가 발생하는 동안 EXE3가 항상 sampling 가능한 상태가 되므로 done pulse miss 가능성을 줄인다.
```

---

## 7. 외부 sequence pseudo-code

```c
int exe3_dma_run_safe(cfg)
{
    assert_functional_reset();

    enable_power_domain(DMA_PD);
    enable_power_domain(FABRIC_PD);
    enable_power_domain(SRC_PD);
    enable_power_domain(DST_PD);
    wait_power_good_all();

    select_stable_clock_source();
    enable_clock(FABRIC_CLK);
    enable_clock(DMA_CLK);
    enable_clock(EXE3_CLK);
    enable_clock(AXI_SYS_CLK);
    enable_clock(AXI_SYS_LP_CLK);
    delay_us(CLOCK_STABLE_WAIT_US);

    release_reset(FABRIC_RST);
    release_reset(SRC_RST);
    release_reset(DST_RST);
    release_reset(DMA_RST);
    release_reset(EXE3_RST);
    wait_reset_released_all();

    configure_firewall_for_dma(cfg.src, cfg.dst, cfg.len);
    configure_interconnect_route();

    if (!check_addr_range(cfg.src, cfg.len, READ))
        return ERR_BAD_SRC_RANGE;
    if (!check_addr_range(cfg.dst, cfg.len, WRITE))
        return ERR_BAD_DST_RANGE;
    if (!check_alignment(cfg.src, cfg.dst, cfg.len))
        return ERR_BAD_ALIGN;

    if (!dummy_read_probe(cfg.src))
        return ERR_SRC_NOT_RESPONDING;
    if (!dummy_read_probe(cfg.src + cfg.len - BEAT_BYTES))
        return ERR_SRC_END_NOT_RESPONDING;

    if (!dummy_write_probe(cfg.dst))
        return ERR_DST_NOT_RESPONDING;

    clear_dma_status();
    clear_exe3_status();

    write_dma_src(cfg.src);
    write_dma_dst(cfg.dst);
    write_dma_len(cfg.len);
    write_dma_mode(cfg.mode);

    if (!verify_dma_config_readback(cfg))
        return ERR_CFG_READBACK_MISMATCH;

    force_exe3_clock_on();
    block_low_power_entry();

    start_exe3();

    for (int i = 0; i < EXE3_TIMEOUT_COUNT; i++) {
        if (read_exe3_done()) {
            allow_low_power_entry();
            return OK;
        }
        delay_us(POLL_INTERVAL_US);
    }

    dump_dma_exe3_monitor_fsm();
    recover_dma_exe3_by_reset();
    allow_low_power_entry();
    return ERR_EXE3_DMA_TIMEOUT;
}
```

---

## 8. 실제 적용 시 주의사항

1. Dummy write는 반드시 side-effect 없는 scratch 영역에만 수행한다.
2. Destination이 write-only이면 readback compare 대신 write response/error status를 확인한다.
3. Source/destination이 FIFO나 command register이면 probe access가 동작을 바꿀 수 있으므로 사용하지 않는다.
4. DMA 진행 중 clock gate 또는 low-power 진입을 금지한다.
5. DMA 진행 중 source/destination reset을 건드리지 않는다.
6. Timeout 발생 후에는 단순 재start보다 monitor dump 후 reset recovery가 안전하다.
7. 같은 address로 반복 실패하면 재시도하지 말고 address map/firewall/power/reset을 먼저 확인한다.
8. Production에서는 timeout 값을 정상 worst-case latency보다 충분히 길게 잡는다.
9. IC bring-up 초기에는 burst length를 작게 시작하고, 정상 확인 후 length를 늘린다.
10. 처음에는 aligned address와 full WSTRB 조건만 사용한다.

---

## 9. Bring-up 단계별 안전 운용법

### Phase 1. 가장 안전한 조건

```text
aligned source address
aligned destination scratch address
small length: 1 beat 또는 4 beat
burst disabled 또는 shortest burst
full WSTRB
clock force-on
low-power disabled
firewall fully open for DMA master
```

목적:

```text
복잡한 burst/unaligned/narrow write 요인을 제거하고 순수 AXI response path만 확인한다.
```

### Phase 2. Length 증가

```text
length를 1 beat -> 4 beat -> 16 beat -> normal length 순서로 증가
각 단계마다 done/status/FSM 확인
```

목적:

```text
RLAST/WLAST/count mismatch 문제를 조기에 발견한다.
```

### Phase 3. 실제 address 적용

```text
scratch address에서 정상 확인 후 실제 source/destination address 적용
적용 전 dummy probe 수행
```

목적:

```text
address map 또는 target-specific response 문제를 분리한다.
```

### Phase 4. Low-power 조건 적용

```text
functional sequence 정상 확인 후 clock gating/low-power를 단계적으로 허용
DMA 진행 중 gate 금지 조건은 유지
```

목적:

```text
clock/reset/power sequencing 문제가 DMA hang으로 보이는 상황을 분리한다.
```

---

## 10. 결론

RTL 수정 없이 가능한 가장 현실적인 회피 방안은 다음이다.

```text
DMA/EXE3 start 전에 source/destination/fabric이 정상 AXI response를 반환할 수 있는지 외부 sequence로 먼저 확인한다.
```

가장 중요한 두 가지는 다음이다.

```text
1. Source dummy read로 RVALID/RLAST path 확인
2. Destination dummy write/readback으로 BVALID/BRESP path 확인
```

이것이 회피 방안인 이유는 명확하다.

```text
현재 hang은 DMA가 response 없는 AXI transaction을 시작한 뒤 내부 timeout 없이 기다리기 때문에 발생한다.
외부 sequence는 response 없는 transaction을 DMA가 시작하지 못하게 사전에 걸러낸다.
따라서 RTL을 수정하지 않고도 EXE3/DMA FSM이 영구 대기 상태로 들어갈 확률을 줄일 수 있다.
```

실패 시에는 다음 정책을 적용한다.

```text
start 전 probe 실패:
  DMA/EXE3 start 금지
  power/clock/reset/firewall/address map 확인

start 후 timeout:
  monitor FSM dump
  DMA/EXE3 local reset recovery
  필요 시 fabric/target reset 또는 power-cycle
```

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
