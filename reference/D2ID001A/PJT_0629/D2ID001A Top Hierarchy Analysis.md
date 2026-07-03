---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: D2ID001A Top Hierarchy Analysis"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/top_hierarchy_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# Top Hierarchy Analysis

## 1. 목적

이 문서는 현재 RTL/TB 프로젝트의 simulation 기준 top hierarchy를 정리한다.

중점:

- TB top에서 DUT까지의 계층
- chip/model wrapper와 digital RTL top의 경계
- logic_scan_top / logic_top의 역할 분리
- 주요 subsystem instance 위치
- compile.py에서 사용하는 hierarchy define과 실제 구조의 대응

## 2. 분석 근거 파일

확인한 주요 파일:

```text
tbench/tb_top.sv
source/model/D2ID001A_TOP.v
source/model/D2ID001A_DIGITAL_TOP.v
source/rtl/top/logic_scan_top.v
source/rtl/top/logic_top.sv
source/rtl/rtl.f
source/model/model.f
doc/compile_flow_analysis.md
```

## 3. Simulation 기준 최상위 구조

`tbench/tb_top.sv`가 testbench top이다.

주요 instance:

```text
tb_top
  ├─ U_HOST        : host stimulus / scenario / MIPI model control
  ├─ chip          : D2ID001A_TOP, main DUT
  ├─ U_FLASH_MODEL : external flash model
  ├─ U_PMIC..4     : PMIC models
  └─ chip_sub 등   : DUAL_CHIP define 시 추가 DUT/host/model
```

근거:

- `tbench/tb_top.sv` line 24: `module tb_top();`
- line 168: `host U_HOST(...)`
- line 236: `D2ID001A_TOP chip(...)`
- line 321: `flash_model U_FLASH_MODEL(...)`
- line 345 이후: `PMIC U_PMIC`, `U_PMIC_2`, `U_PMIC_3`, `U_PMIC_4`
- line 393 이후: `DUAL_CHIP` 조건부로 `U_HOST_SUB`, `chip_sub`, sub flash/PMIC

## 4. DUT wrapper 계층

chip-level wrapper 계층은 다음과 같다.

```text
tb_top.chip
  D2ID001A_TOP
    ├─ D2ID001A_DIGITAL_TOP   : digital top wrapper
    ├─ MIPI_DPHY_UMC40        : MIPI PHY model
    ├─ OSC140M_VDDI_V0        : OSC model
    ├─ OSC140M_VDDI_V0_DSC    : DSC용 OSC model
    ├─ analog_delay_model     : soft reset delay 관련 analog delay model
    └─ pad / analog / OTP / temp / supply 연결 model glue
```

`D2ID001A_TOP.v`는 mixed-signal simulation wrapper 성격이 강하다.

특징:

- `SNPS` 여부에 따라 `snps_msv_nettype_pkg` 또는 `cds_rnm_pkg`를 import한다.
- MIPI pad는 `mipi_realnet` nettype을 사용한다.
- power rail은 `supply1/supply0`와 real/wreal 성격 net으로 모델링된다.
- digital top과 analog/model block 사이의 신호를 매우 넓은 wire bundle로 연결한다.

## 5. Digital top wrapper 계층

`source/model/D2ID001A_DIGITAL_TOP.v`는 digital wrapper다.

핵심 instance:

```text
D2ID001A_DIGITAL_TOP
  └─ U_LOGIC_SCAN_TOP : logic_scan_top
```

근거:

- `D2ID001A_DIGITAL_TOP.v` line 15: `module D2ID001A_DIGITAL_TOP`
- line 1061 부근: `logic_scan_top #(...) U_LOGIC_SCAN_TOP (...)`

역할:

- `D2ID001A_TOP`과 `logic_scan_top` 사이에서 analog/pad/OTP/MIPI/OSC/temp/GPO/source/gamma 신호를 연결한다.
- OTP macro 관련 wire, analog control output, MIPI PHY control, GPO/source/gamma 관련 포트를 scan wrapper로 전달한다.

## 6. Scan wrapper 계층

`source/rtl/top/logic_scan_top.v`는 scan/test wrapper다.

핵심 instance:

```text
D2ID001A_DIGITAL_TOP.U_LOGIC_SCAN_TOP
  └─ U_LOGIC_TOP : logic_top
```

근거:

- `logic_scan_top.v` line 15: `module logic_scan_top`
- file 내부에 `logic_top #(...) U_LOGIC_TOP (...)` instance 존재

역할:

- scan/test mode 관련 muxing과 top-level signal forwarding을 담당한다.
- 실질적인 functional digital subsystem은 `logic_top` 아래에 있다.
- compile.py의 define도 이 구조를 기준으로 잡혀 있다.

## 7. Functional logic top 계층

`source/rtl/top/logic_top.sv`가 주요 functional RTL 통합 top이다.

핵심 instance:

```text
logic_top
  ├─ u_rt             : register table / command decode / RT slave
  ├─ u_bus_top        : AXI/APB interconnect
  ├─ u_peri           : sys/peri/MCU/DMA/OTP/VROM/flash slave
  ├─ u_mipi_dsi_top   : MIPI DSI receiver/control path
  ├─ u_vgen           : video generator / timing source
  ├─ u_vin            : video input pipeline
  ├─ u_vpr_sdc        : VPR/SDC pipeline and SRAM/GRAM interfaces
  ├─ u_vout           : source/GPO/output driver path
  ├─ u_misc           : misc, DCC, TE, checksum, pad control support
  ├─ u_pwrctl         : power/LDO sequencing and analog control outputs
  ├─ u_tempctl        : temperature sensor control
  ├─ u_errfg          : error flag aggregation
  ├─ u_osc_cont_top   : oscillator control
  └─ u_irq            : IRQ aggregation toward MCU
```

근거:

- `logic_top.sv` 내 instance 검색 결과:
  - `rt u_rt (`
  - `bus_top u_bus_top (`
  - `peri u_peri (`
  - `mipi_dsi_top u_mipi_dsi_top (`
  - `vgen u_vgen (`
  - `vin u_vin (`
  - `vpr_sdc u_vpr_sdc (`
  - `vout u_vout (`
  - `misc u_misc (`
  - `pwrctl #(...) u_pwrctl (`
  - `tempctl u_tempctl (`
  - `errfg u_errfg(`
  - `osc_cont_top u_osc_cont_top (`
  - `irq u_irq (`

## 8. Compile.py hierarchy define 대응

`bin/compile.py`의 기본 define은 다음 hierarchy를 전제로 한다.

```text
LOGIC_SCAN_TOP_PATH = tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP
LOGIC_TOP_PATH      = tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP
DIGITAL_TOP_PATH    = tb_top.chip.U_DIGITAL_TOP
MIPI_HOST_PATH      = U_MIPI_MODEL
```

실제 구조와의 대응:

```text
tb_top
  └─ chip                         : D2ID001A_TOP
      └─ U_DIGITAL_TOP            : D2ID001A_DIGITAL_TOP
          └─ U_LOGIC_SCAN_TOP     : logic_scan_top
              └─ U_LOGIC_TOP      : logic_top
```

주의:

- `U_DIGITAL_TOP` instance명은 `D2ID001A_TOP.v` 내부에서 확인되는 digital top instance명과 일치해야 한다.
- waveform/checker/injector/task가 compile define을 통해 hierarchical force/probe를 하면 이 path가 틀릴 경우 simulation은 compile되더라도 runtime 접근이 실패할 수 있다.
- `DUAL_CHIP` 사용 시 `chip_sub` hierarchy가 별도로 생기지만 compile.py 기본 define은 main `chip` 기준이다.

## 9. 주요 signal domain 관점

### 9.1 Pad / external interface

```text
TB U_HOST / model
  ↔ tb_top pad wires
  ↔ D2ID001A_TOP pad ports
  ↔ D2ID001A_DIGITAL_TOP / MIPI_DPHY / pad glue
  ↔ logic_scan_top
  ↔ logic_top subsystem
```

대표 pad:

- MIPI: CKN/CKP/D0N/D0P/.../D3N/D3P
- reset: RESX
- flash: F_CSN/F_SCK/F_IO[3:0]
- PMIC/I2C/SWIRE-like: PMIC_GPIO1/2, PMIC_EN2/3/4, SCL/SDA
- test/mode: EXCLK, DIS_EXCLK, T_IO[3:0], TMODE[5:0]

### 9.2 Reset/clock root

Reset/clock root는 functional 관점에서 `logic_top.u_peri` 내부 `sys` 계열이 중심이다.

상위 입력:

```text
RESX / TO_RESX_SPROUT
TO_PORB_VDD
TO_OSC_CLK / TO_OSC1_CLK
EXCLK / DIS_EXCLK / TMODE
MIPI PHY clock/reset related signals
```

`logic_top.u_peri`가 `sys_reset`, `sys_clk`, `sys_fsm`을 통해 reset/clock을 생성하여 내부 block으로 배포한다.

### 9.3 Register / command table

Register command interface는 `logic_top.u_rt`가 중심이다.

```text
MIPI command / AXI slave access
  → rt_slv
  → rt_dec
  → command group wrapper
  → cmd_xx_reg_if interfaces
  → logic_top 내부 subsystem inputs
```

`logic_top`에는 다수의 `u_cmd_xx_reg_if` interface instance가 있고, block instance들은 `.cmd_xx(u_cmd_xx_reg_if.reg_in)` 형태로 연결된다.

## 10. Debug 시 유용한 hierarchy path

주요 path 후보:

```text
tb_top.chip
tb_top.chip.U_DIGITAL_TOP
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP

tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_rt
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_bus_top
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_peri
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_mipi_dsi_top
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_vgen
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_vin
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_vpr_sdc
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_vout
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP.u_pwrctl
```

## 11. 주의점

1. `D2ID001A_TOP.v` 파일 header에는 `D2I1001A_TOP.v`로 보이는 legacy filename comment가 남아 있으나, 실제 module명은 `D2ID001A_TOP`이다.
2. `logic_top.sv`는 `DO NOT USE Verilog-Auto` 주석이 있으므로 자동 포트 정리/수정은 조심해야 한다.
3. `D2ID001A_DIGITAL_TOP.v`와 `logic_scan_top.v`는 massive port forwarding 구조라 hierarchy보다 port 연결 누락/오연결이 더 큰 리스크다.
4. compile.py hierarchy define과 실제 instance명이 다르면 checker/injector/wave script가 깨진다.
5. `DUAL_CHIP`은 별도 sub hierarchy를 만들지만 default debug path는 main chip 기준으로 봐야 한다.

## 12. 결론

현재 simulation top hierarchy는 다음과 같이 요약된다.

```text
tb_top
  └─ chip : D2ID001A_TOP
      └─ U_DIGITAL_TOP : D2ID001A_DIGITAL_TOP
          └─ U_LOGIC_SCAN_TOP : logic_scan_top
              └─ U_LOGIC_TOP : logic_top
                  ├─ u_rt
                  ├─ u_bus_top
                  ├─ u_peri
                  ├─ u_mipi_dsi_top
                  ├─ u_vgen/u_vin/u_vpr_sdc/u_vout
                  ├─ u_misc/u_errfg/u_tempctl/u_pwrctl/u_osc_cont_top
                  └─ u_irq
```

기능 debug의 실제 시작점은 대부분 `logic_top` 하위 instance이며, reset/boot는 `u_peri`, register 접근은 `u_rt + u_bus_top`, 영상 경로는 `u_mipi_dsi_top/u_vin/u_vpr_sdc/u_vout`를 우선 추적하는 것이 좋다.

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
