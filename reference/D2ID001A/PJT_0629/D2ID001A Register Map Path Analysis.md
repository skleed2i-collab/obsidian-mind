---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: D2ID001A Register Map Path Analysis"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/register_map_path_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# Register Map / Register Access Path Analysis

## 1. 목적

이 문서는 현재 RTL에서 register map과 register access path를 정리한다.

중점:

- MIPI command / AXI register access가 `rt`로 들어오는 경로
- `rt_slv`와 `rt_dec`의 역할
- command별 `cmd_xx_reg_if` interface 분배 구조
- subsystem에서 register를 받는 방식
- memory map(`mmap.v`)과 bus slave selection 구조
- testbench register access package의 위치

## 2. 분석 근거 파일

확인한 주요 파일:

```text
source/rtl/top/logic_top.sv
source/rtl/top/mmap.v
source/rtl/rt/rt.v
source/rtl/rt/rt_slv.v
source/rtl/rt/rt_dec.sv
source/rtl/rt/rt.f
source/rtl/rt/cmd_reg_if/cmd_reg_if.f
source/rtl/rt/cmd_reg_if/cmd_*_reg_if.sv
source/rtl/peri/peri.v
source/rtl/peri/sys/sys_fsm_nvm_reset.v
tbench/package/reg_access/reg_access_pkg.sv
```

## 3. Register access 전체 그림

현재 register path는 크게 두 갈래가 있다.

```text
1) MIPI command path
   MIPI DSI / host stimulus
     → rt_slv MIPI FIFO/write path
     → rt_dec
     → rt command modules
     → cmd_xx_reg_if
     → subsystem

2) AXI system bus path
   AXI master / sys_mst / MCU / DMA 등
     → bus_top
     → axi4_slv_sys_rt
     → rt_slv AXI slave path
     → rt_dec
     → rt command modules
     → cmd_xx_reg_if
     → subsystem
```

두 path 모두 `logic_top.u_rt`를 통해 command/register table로 들어가는 구조다.

## 4. rt top interface

`source/rtl/rt/rt.v`는 register table top이다.

주요 port:

```text
cmd_48_reg_if.reg_out cmd_48
cmd_b1_reg_if.reg_out cmd_b1
...
cmd_ff_reg_if.reg_out cmd_ff

axi4_if.slave axi4_slv_sys_rt

MIPI write inputs:
  i_mipi_cmd
  i_mipi_wdata0..3
  i_mipi_wdata0_en..3_en
  i_mipi_wdata0_pcnt..3_pcnt

clock/reset:
  clk_mipi_rt
  clk_rt_mipi_fifo
  clk_axi_sys
  clk_axi_sys_lp
  clk_rt
  rstb_dly/rstb_mbist/rstb_sof/rstb_bus
```

역할:

- MIPI command와 AXI register access를 받아 내부 register table로 전달한다.
- command별 register interface(`cmd_xx_reg_if`)를 밖으로 제공한다.
- SRAM clock gate, RT clock gate, read-only monitor ports도 함께 다룬다.

## 5. rt_slv 역할

`source/rtl/rt/rt_slv.v`는 slave front-end다.

핵심 기능:

```text
MIPI async FIFO
AXI4 slave write/read FSM
rt_*_cmd.v write interface 생성
rt_*_cmd.v read interface 생성
read FIFO / response handling
RT/MIPI/AXI clock gate on signal 생성
```

주요 localparam:

```text
MIPI__IDLE / OPEN_REQ / OPEN / CLOSE_REQ
SLV__WRIT_* state
SLV__READ_* state
RFIFO_WRIT_* state
CLKGATE_* state
```

MIPI path:

```text
i_mipi_cmd + i_mipi_wdata*_pcnt/data/en
  → async_fifo u_mipi_cmd_fifo
  → clk_rt_mipi_fifo domain pop
  → o_slv_wcmd/o_slv_wdat*/o_slv_wden*
```

AXI path:

```text
axi4_slv_sys_rt
  → AW/W/B write FSM
  → AR/R read FSM
  → o_slv_wcmd/o_slv_rcmd/o_slv_*den
```

디버그 포인트:

- MIPI write가 안 들어오면 `w_mipi_wden`, FIFO empty/full/overflow, `clk_mipi_rt`, `clk_rt_mipi_fifo`, `rstb_bus`를 본다.
- AXI write/read가 안 되면 `axi4_slv_sys_rt` handshake, `clk_axi_sys`, `rstb_bus`, write/read FSM state를 본다.
- read가 hang이면 read FIFO full/empty, `i_slv_rhold`, downstream `o_slv_rden`/`i_slv_rdat*`를 본다.

## 6. rt_dec 역할

`source/rtl/rt/rt_dec.sv`는 command decoder다.

입력:

```text
i_slv_wcmd / i_slv_wdat*_pcnt / i_slv_wdat* / i_slv_wden*
i_slv_rcmd / i_slv_rdat_pcnt / i_slv_rden*
i_slv_wbus
```

출력:

```text
o_lv1_* : level-1 command path
o_lv2_* : level-2 command path
o_lv3_* : level-3 command path
o_slv_rdat* / o_slv_rden
```

Command decode group:

```text
LV1 command examples:
  00-46, 4D-59, 5E, 60, 61, 70, 71, 80-88, 9D, 9E, A1, A2, AA, AF, DA-DC

LV2 command examples:
  BA, CC, E4, E5, EC, C5, C1, 48, B5, C7, ED, F5, E0, D9,
  F4, F9, BD, BE, F2, C2, C3, F6, 62, 63, 64, 66, 67, 68,
  69, 7A, 7B, 7C, 7D, 6A, 6B, 92-9C, BF, F8, E6-E9,
  C6, D6-D8, F0, F1, F7, FC

OTP command:
  B1, D0

LV3 command examples:
  C4, EB, FB, EF, F3, D2, D3, BC, D4, FD, FE, FF
```

중요:

- command가 어느 level/group에 들어가는지에 따라 downstream wrapper가 달라진다.
- `o_dec_wbus`는 write가 bus-origin인지 여부를 register side로 전달하는 latch 성격이다.
- read hold는 C6/GPO/SYS SRAM read FIFO almost-full 등으로 발생할 수 있다.

## 7. rt.f command group 구조

`source/rtl/rt/rt.f`는 command RTL compile list다.

주요 group:

```text
mipi       : E4, E5, EC
regflag    : A0-A4
global0    : 48, B1, B5, BA, BC
global1    : C1, C4, C5, C7, CC
global2    : D0, D2, D4, D6, D7, D8, D9
global3    : E0, EB, ED, EF
global4    : F3, F4, F5, F8, F9, FB, FD, FE, FF
vout       : C2, F6
vin        : BD, BE, E6, E7, E8, E9, F2
vpr_sdc0   : 62, 63, 64, 66, 67, 68, 69, 6A, 6B
vpr_sdc1   : 7A, 7B, 7C, 7D, BF
vpr_sdc2   : 92, 93, 94, 95, 96
vpr_sdc3   : 97, 98, 99, 9A, 9B, 9C
```

각 group은 wrapper + command module 파일로 구성된다.

예:

```text
rt_global2/rt_rt_global2_wrapper.v
rt_global2/rt_d0.v
rt_global2/rt_d2.v
...
```

## 8. cmd_reg_if 구조

`source/rtl/rt/cmd_reg_if/cmd_reg_if.f`는 command별 SV interface list다.

예:

```text
cmd_ba_reg_if.sv
cmd_cc_reg_if.sv
cmd_e4_reg_if.sv
...
cmd_ff_reg_if.sv
```

이 interface들은 `logic_top` 내부에서 instance화되어 register table과 subsystem 사이를 연결한다.

개념적 흐름:

```text
rt command module writes internal fields
  → cmd_xx_reg_if.reg_out from rt
  → u_cmd_xx_reg_if interface instance
  → subsystem receives cmd_xx_reg_if.reg_in
```

예시:

```text
logic_top.u_rt .cmd_d0(u_cmd_d0_reg_if.reg_out)
logic_top.u_peri .cmd_d0(u_cmd_d0_reg_if.reg_in)
logic_top.u_pwrctl .cmd_d9(u_cmd_d9_reg_if.reg_in)
```

## 9. logic_top에서 register fanout

`logic_top.sv`는 register interface를 여러 subsystem으로 fanout한다.

대표 연결:

```text
u_rt
  → 모든 cmd_xx_reg_if.reg_out 생성

u_peri
  ← cmd_48, cmd_62, cmd_9b, cmd_b1, cmd_c1, cmd_d0, cmd_d2,
     cmd_d9, cmd_eb, cmd_fb, cmd_ff 등

u_pwrctl
  ← cmd_b5, cmd_d9, cmd_ef, cmd_f3, cmd_f4, cmd_f6, cmd_f9,
     cmd_e0, cmd_d2 등

u_vin / u_vout / u_vpr_sdc / u_mipi_dsi_top / u_misc 등
  ← 각 block 관련 cmd interface
```

따라서 특정 register field debug는 다음 순서로 추적한다.

```text
cmd_xx_reg_if.sv에서 field명 확인
  → rt_xx.v에서 field write/read behavior 확인
  → wrapper에서 group 연결 확인
  → rt.v / logic_top u_rt 연결 확인
  → 해당 subsystem의 cmd_xx.reg_in 사용처 확인
```

## 10. mmap.v memory map 구조

`source/rtl/top/mmap.v`는 bus-level address decode symbol을 정의한다.

상위 slave selection:

```text
MMAP_SYS_RT          = 8'h00
MMAP_SYS_OTP         = 8'h01
MMAP_SYS_FLASH       = 8'h02
MMAP_SYS_FLASH_CTRL  = 8'h03
MMAP_SYS_DMA         = 8'h04
MMAP_SYS_MCU         = 8'h05
MMAP_PMIC            = 8'h06
MMAP_SYS_VROM        = 8'h07
MMAP_REGFLAG_RT      = 8'h20
MMAP_REGFLAG_SRAM    = 8'h21
MMAP_VPR_SDC_SRAM    = 8'h30
MMAP_VPR_SDC_GRAM    = 8'h40
MMAP_S1_RT           = 8'hA0
MMAP_S1_LATCH        = 8'hA2
MMAP_SN_RT           = 8'hB0
MMAP_SN_LATCH        = 8'hB2
MMAP_GPO             = 8'hC0
MMAP_VIN_G2G_LUT     = 8'hF0
MMAP_RESERVED        = 8'hFF
```

의미:

- 32-bit address의 상위 영역 또는 slave id 성격으로 보인다.
- `MMAP_SYS_RT`는 register table slave (`axi4_slv_sys_rt`)로 연결된다.
- VPR SRAM/GRAM, regflag SRAM, VOUT/S1/SN/GPO 등은 별도 bus slave 영역이다.

## 11. OTP / boot 관련 register map

`mmap.v`에는 OTP/MCU boot 관련 address가 정의되어 있다.

예:

```text
MMAP_SYS_OTP_MCU_IM__HEADER0 = {MMAP_SYS_OTP,24'h00_4008}
MMAP_SYS_OTP_MCU_IM__HEADER1 = {MMAP_SYS_OTP,24'h00_400C}
MMAP_SYS_OTP_MCU_IM__HEADER2 = {MMAP_SYS_OTP,24'h00_4010}
MMAP_SYS_OTP_MCU_IM__HEADER3 = {MMAP_SYS_OTP,24'h00_4014}
MMAP_SYS_OTP_MCU_CRC_HEADER0 = {MMAP_SYS_OTP,24'h00_4018}
...
MMAP_SYS_OTP_MCU_CTL_HEADER3 = {MMAP_SYS_OTP,24'h00_4034}
```

`sys_fsm_nvm_reset.v`는 이 symbol을 사용하여 OTP header/default 값을 읽고, MCU IM/CRC/CTL address register에 write한다.

예:

```text
read  MMAP_SYS_OTP_MCU_IM__HEADER3..0
read  MMAP_SYS_MCU_RESET_OTP_LOAD_IM_ADR
write MMAP_SYS_MCU_IM_ADR
read  MMAP_SYS_OTP_MCU_CRC_HEADER3..0
read  MMAP_SYS_MCU_RESET_OTP_IM_CRC0
write MMAP_SYS_MCU_IM_CRC_ADR
```

## 12. Bus top과 RT slave 선택

`logic_top.u_bus_top`는 여러 AXI/APB interface를 연결한다.

관련 interface 후보:

```text
u_axi4_if_sys_rt
u_axi4_if_sys_otp / apb_slv_vout_otp
u_axi4_if_sys_flash
u_axi4_if_sys_vrom
u_axi4_if_sys_dma0/1
u_axi4_if_sys_mcu
u_axi4_if_*_vprsram/vprgram/regflag/vout/gpo
```

RT register 접근은 최종적으로:

```text
bus_top decode MMAP_SYS_RT
  → u_axi4_if_sys_rt
  → logic_top.u_rt.axi4_slv_sys_rt
  → rt_slv
```

## 13. Testbench register access package

`tbench/package/reg_access/reg_access_pkg.sv`는 TB side register utility package다.

포함 내용:

```text
import cmd_map_pkg::*
include reg_item_cl.sv
include reg_list_cl.sv
REG_MAX_WIDTH = 64
val2str()
calc_width()
```

의미:

- TB scenario에서 register item/list를 만들거나 command map 기반 access를 추상화하는 유틸리티다.
- `cmd_map_pkg`와 연계되므로 command map generation 결과와 RTL command table이 일치해야 한다.

## 14. Register debug 절차

특정 register write가 안 먹는 경우:

```text
1. access source 확인
   - MIPI command인지 AXI/sys/MCU access인지 구분

2. rt_slv front-end 확인
   - MIPI: i_mipi_cmd, i_mipi_wdata*, i_mipi_wdata*_en, FIFO pop
   - AXI : AW/W/B or AR/R handshake, r_slv_writ_fsm/r_slv_read_fsm

3. rt_dec command decode 확인
   - command가 LV1/LV2/LV3/OTP group에 포함되는지
   - o_lv*_wden/o_lv*_rden이 발생하는지
   - o_dec_wbus 값이 의도와 맞는지

4. command module 확인
   - source/rtl/rt/<group>/rt_xx.v
   - write pcnt/data index가 맞는지
   - reset default와 write protection/key가 있는지

5. interface fanout 확인
   - cmd_xx_reg_if.sv field명
   - logic_top에서 subsystem으로 연결되었는지
   - subsystem 내부에서 reg_in field를 실제 사용하는지

6. clock/reset 확인
   - clk_rt, clk_axi_sys, clk_axi_sys_lp
   - rstb_bus, rstb_sof, rstb_slp
```

특정 register read가 안 되는 경우:

```text
1. rt_slv read FSM이 read request를 만들었는지
2. rt_dec가 rcmd를 올바른 level/group으로 decode했는지
3. command module이 read data enable을 내는지
4. i_slv_rhold가 걸려 read response가 hold되는지
5. read FIFO full/empty/overflow/underflow가 없는지
6. AXI RVALID/RLAST/RDATA가 정상인지
```

## 15. MIPI vs AXI access 차이

MIPI access:

```text
MIPI PHY/model → mipi_dsi_top → rt MIPI write inputs → async FIFO → rt_dec
```

특징:

- command/data packet count (`pcnt`)가 중요하다.
- MIPI clock domain과 RT FIFO pop domain CDC가 있다.
- read path는 MIPI packet/response 구현 확인이 추가로 필요하다.

AXI access:

```text
MCU/sys master/TB force → bus_top → axi4_slv_sys_rt → rt_slv → rt_dec
```

특징:

- `mmap.v` slave decode가 중요하다.
- address 상위 byte가 `MMAP_SYS_RT = 8'h00` 계열이어야 한다.
- AXI handshake와 length/size 해석이 중요하다.

## 16. Register map 관련 리스크

1. command map Excel/Jinja generated RTL과 TB `cmd_map_pkg`가 mismatch되면 TB access와 RTL field가 어긋날 수 있다.
2. `rt_dec.sv`는 generated file이며 command group decode list가 고정되어 있다. 새 command 추가 시 template/regeneration이 필요하다.
3. command별 `pcnt` offset이 틀리면 write는 발생해도 field가 갱신되지 않는다.
4. bus-origin write 여부(`i_slv_wbus`/`o_dec_wbus`)에 따라 command 동작이 달라질 수 있다.
5. read hold/read FIFO backpressure가 있으면 AXI read hang처럼 보일 수 있다.
6. `mmap.v`는 여러 block의 address symbol을 공유하므로 변경 영향이 sys_fsm, bus, RTL, TB에 동시에 퍼진다.
7. `source/rtl/rt/vpr_sdc/`와 `vpr_sdc0..3/`처럼 legacy/alternate directory가 같이 보이므로 active filelist(`rt.f`) 기준으로 판단해야 한다.

## 17. 결론

Register path는 다음과 같이 요약된다.

```text
MIPI/AXI access
  → logic_top.u_rt
  → rt_slv
  → rt_dec
  → rt_<group>_wrapper
  → rt_xx command module
  → cmd_xx_reg_if
  → logic_top subsystem
```

Memory-map/bus 접근은 다음 상위 decode를 먼저 본다.

```text
mmap.v MMAP_* slave id
  → bus_top selected interface
  → target slave, e.g. u_axi4_if_sys_rt
  → rt or SRAM/VROM/OTP/MCU/DMA target
```

따라서 register debug의 우선순위는 다음이다.

```text
1. access source: MIPI or AXI
2. mmap/slave decode: MMAP_SYS_RT or other target
3. rt_slv handshake/FSM
4. rt_dec command group
5. rt_xx module pcnt/data/default/key
6. cmd_xx_reg_if fanout to subsystem
```

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
