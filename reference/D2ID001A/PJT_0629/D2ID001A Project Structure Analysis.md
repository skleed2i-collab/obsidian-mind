---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: D2ID001A Project Structure Analysis"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/project_structure_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# Project Structure Analysis

## 1. 분석 범위

현재 폴더의 파일 목록, 주요 README, filelist(.f), top/testbench 파일을 기준으로 프로젝트 구조를 분석했다.

확인한 주요 근거 파일:

- source/rtl/rtl.f
- source/model/model.f
- tbench/tbench.f
- tbench/package/package.f
- tbench/README.md
- tbench/scenario/README.md
- bin/README.md
- source/sw/fw_link_report.md
- source/rtl/top/logic_top.sv
- source/rtl/top/logic_scan_top.v
- source/model/D2ID001A_TOP.v
- tbench/tb_top.sv

주의:
- shell `pwd` 실행은 사용자 승인에서 차단되어 절대 경로는 확인하지 못했다.
- 파일 검색 기준 총 파일 수는 2513개로 확인된다.

## 2. 전체 프로젝트 성격

이 프로젝트는 D2ID001A 계열로 보이는 디지털 RTL + 칩 레벨 모델 + 시뮬레이션/검증 환경 + 펌웨어 초기화 코드가 함께 있는 SoC/Display Driver IC 성격의 통합 디지털 프로젝트다.

상위 구성은 크게 다음 4개 축으로 나뉜다.

1. RTL 설계 본체
   - source/rtl
   - logic_top, logic_scan_top, bus, peri, rt, vin, vout, vpr_sdc, vgen 등

2. 칩/아날로그/외부 IP 모델 연결
   - source/model
   - D2ID001A_TOP.v, D2ID001A_DIGITAL_TOP.v
   - MIPI PHY, PAD, OSC, OTP, temp sensor, SRAM/STD cell model 연결

3. 검증/시뮬레이션 환경
   - tbench
   - tb_top, host, scenario, checker, coverage, package, random_model, injector, external model

4. 자동화/펌웨어/유틸리티
   - bin
   - source/sw
   - source/sw_wa

## 3. 파일 수 요약

검색 기준 파일 수:

- 전체: 2513 files
- source/rtl: 1915 files
- tbench: 415 files
- source/sw: 91 files
- source/sw_wa: 23 files
- source/model: 6 files
- bin: 61 files
- source/pre: 1 file
- source/post: 1 file

주요 확장자 기준:

- *.v: 1539 files
- *.sv: 506 files
- *.svh: 80 files
- *.f: 79 files

해석:
- Verilog/SystemVerilog RTL 비중이 매우 높다.
- filelist(.f) 기반 컴파일 플로우가 명확히 존재한다.
- tbench 쪽도 checker, scenario, package, coverage까지 포함한 통합 검증 환경이다.

## 4. 최상위 디렉토리 역할

### 4.1 bin/

프로젝트 표준 스크립트 위치다.

bin/README.md에 따르면 이 폴더는 simulation, compile, regression, log parsing, PI environment 생성 등 공통 engineering job을 위한 표준 스크립트 집합이다.

주요 파일:

- compile.py
  - design/testbench compile 및 elaboration 목적
- sim.py
  - compile.py와 연계되는 simulation 실행 스크립트
- regress_submit.py, regress_collect.py, regress_fail.py, regress_wait.py
  - regression 실행/수집/대기/실패 분석 계열
- rtl_tree.py
  - RTL hierarchy/tree 분석 용도 추정
- cmd_map.py, cmd_info.py
  - MCS command map parsing/document generation 계열
- fw/link 관련은 source/sw 쪽에 존재
- git_*.py, git_*.sh
  - 프로젝트 내 git 보조 작업
- log_parser.py, get_error.sh, chk_compile_log.sh
  - 로그/에러 분석
- slurm_*.py
  - 서버/cluster regression submit 연동 가능성

분석:
- 이 프로젝트는 단순 RTL 저장소가 아니라 compile/sim/regression 자동화까지 포함한다.
- 사용자는 PATH에 bin을 추가해서 사용하는 구조로 보인다.

### 4.2 source/

설계 소스의 본체다.

하위 구조:

source/
├── rtl/
├── model/
├── sw/
├── sw_wa/
├── pre/
└── post/

### 4.3 tbench/

검증 환경 본체다.

주요 구성:

- tb_top.sv
- host.sv
- tbench.f
- scenario/
- checker/
- coverage/
- package/
- random_model/
- injector/
- mipi_tx/
- flash/
- i2c_slave/
- pmic/
- eds_scen/

## 5. RTL 구조 분석: source/rtl

source/rtl은 총 1915개 파일이 확인되며, 프로젝트의 핵심 RTL이다.

### 5.1 최상위 RTL

주요 파일:

- source/rtl/top/logic_top.sv
- source/rtl/top/logic_scan_top.v
- source/rtl/top/irq.v
- source/rtl/top/mmap.v
- source/rtl/top/vrom_map.v
- source/rtl/top/vrom_loading.v
- source/rtl/top/axi4_if.sv
- source/rtl/top/apb_if.sv
- source/rtl/top/analog_ldo_params.vh
- source/rtl/top/analog_ldo_param_map.vh

logic_top.sv:
- module logic_top
- digital logic functional top로 보인다.
- FPGA define 조건부 포트가 존재한다.
- IO_* pad-facing signal이 매우 많이 정의되어 있다.
- line count가 6499로 매우 크다.

logic_scan_top.v:
- module logic_scan_top
- logic_top과 유사하지만 scan/test 또는 analog/column/source interface를 더 포함하는 top wrapper로 보인다.
- Last Modified가 2026/06/26으로 최근 수정 파일이다.
- line count 1637.

분석:
- logic_top은 functional digital top.
- logic_scan_top은 scan/analog boundary가 포함된 상위 wrapper 또는 DFT/scan-aware top로 보인다.
- source/rtl/rtl.f 기준 실제 컴파일에서는 logic_scan_top.v와 logic_top.sv가 모두 포함된다.

### 5.2 RTL 통합 filelist: source/rtl/rtl.f

source/rtl/rtl.f는 RTL 전체 compile entry에 가깝다.

포함 순서 요약:

1. incdir 설정
   - source/rtl
   - source/rtl/top
   - source/rtl/bus
   - source/rtl/rt/rt_inst_wire_inc
   - source/rtl/rt/cmd_reg_if_wire_inc
   - source/rtl/vpr_sdc
   - source/rtl/peri/flash_slv
   - source/rtl/peri/mcu/ibex/rv_sim_only

2. SV interface
   - rt/cmd_reg_if/cmd_reg_if.f
   - rt/ucs_reg_if.sv
   - top/axi4_if.sv
   - top/apb_if.sv

3. top
   - logic_scan_top.v
   - logic_top.sv
   - irq.v

4. SRAM wrapper
   - source/rtl/sram/mem_wrap.f
   - sram_1002/mem_wrap.f는 주석 처리됨

5. subsystem filelist
   - peri/peri.f
   - mipi/mipi.f
   - osc/osc.f
   - bus/bus.flat.f
   - rt/rt.f
   - common/common.f
   - pmic/vcode.f
   - pwrctl/pwrctl.f
   - tempctl/tempctl.f
   - errfg/errfg.f
   - misc/misc.f
   - vgen/vgen.f
   - vin/vcode.f
   - vpr_sdc/vpr_sdc.f
   - vout/vout.f

6. padcon
   - padcon.v/io/i/o/nand

7. Synopsys DesignWare sim model path
   - /app/synopsys/syn/V-2023.12-SP3/... DW01/DW02 models

8. external sim IP
   - sim_ext_ip.f

분석:
- rtl.f가 RTL compile의 중심이다.
- sram variant 선택 주석이 있어 project/package variant 관리가 filelist에서 이루어진다.
- bus는 bus.flat.f가 활성화되어 있고 bus.f는 주석 처리되어 있다.
- Synopsys DW dependency가 하드코딩되어 있어 서버 환경 종속성이 존재한다.

### 5.3 Bus 구조

경로:

- source/rtl/bus

주요 구성:

- bus_top.sv
- bus.f
- bus.flat.f
- axi4_para.v
- axi4_bus_lp.v
- axi4_sys/
- axi4_vprsram/
- axi4_vprgram/
- axi4_regflag/
- axi4_bridge_32to8_5/
- axi4_bridge_32to64_5/
- axi4_bridge_32to32_5/
- apb_source_sn/
- apb_source_s1/
- apb_otp/
- apb_g2g_lut/

분석:
- AXI4 backbone과 APB bridge가 주요 bus fabric이다.
- 다수의 DW_axi 생성물/undef/model 파일이 포함되어 있어 Synopsys DesignWare 기반 bus generator output일 가능성이 높다.
- vprgram/vprsram/regflag/sys 등 기능별 AXI segment가 분리되어 있다.

### 5.4 Peripheral 구조: source/rtl/peri

주요 하위 블록:

- sys/
- mcu/
- dma/
- vrom_slv/
- otp_slv/
- flash_slv/

sys:
- sys.v
- sys_clk.v
- sys_reset.v
- sys_fsm*.v
- sys_exi*.v
- sys_clk_mux2/4, sys_clk_hlpm, sys_clk_osc_overshoot_protect

mcu:
- rv_top.v
- rv_mst/slv 계열
- ibex core 포함
- bus_bridges 포함

vrom_slv:
- vrom_slv_top.v
- vrom_ctrl.sv
- axi4_slv_vrom.sv
- roms_lib.v
- roms_*_modify.vh

dma:
- dma.v
- dma_core.v
- dma_core_mst/slv/fifo
- mipidma/

otp_slv:
- otp_slv_top.sv
- otp_ctrl.sv
- otp_reg_table.sv
- apb_slv_otp.sv

flash_slv:
- flash_slv_top.v
- flash_slv_spi/read_fifo/para/errfg/cdc/bus

분석:
- sys/peri가 reset, clock, boot, NVM/VROM, MCU, DMA의 중심이다.
- Ibex RISC-V core가 포함되어 있고, FW 초기화 코드와 연동되는 구조로 보인다.
- VROM/OTP/Flash가 부팅/초기화 경로에 중요하다.

### 5.5 RT/Register Table 구조: source/rtl/rt

주요 구성:

- rt.v
- rt_slv.v
- rt_dec.sv
- rt_ucs.v
- rt_ucs_rddsdr.v
- rt_sram_arbiter.v
- rt_sys_sram.v
- rt_gpo_sram.v
- rt_c6_sram.v
- rt_clk_gate.v
- rt_clk_gate_enable_include.v
- rt_inst_wire_inc/
- cmd_reg_if/
- cmd_reg_if_wire_inc/
- rt_global0~4/
- vpr_sdc, vpr_sdc0~3/
- vin/
- vout/
- mipi/
- regflag/

cmd_reg_if:
- cmd_xx_reg_if.sv 파일들이 다수 존재한다.
- cmd_reg_if.f로 묶인다.

rt_inst_wire_inc:
- wrapper instance/wire include 자동 생성물로 보이는 파일들이 많다.

분석:
- RT는 command/register map 기반의 register table 또는 runtime register access block으로 보인다.
- cmd_xx 단위로 address/page/register block이 분할되어 있다.
- vpr_sdc/vin/vout/mipi/global/regflag 등 subsystem register bank와 연결된다.
- 이전 디버그 맥락상 clk_axi_sys_lp/readback path 분석에 중요한 블록이다.

### 5.6 VPR/SDC 구조: source/rtl/vpr_sdc

주요 구성:

- vpr_sdc.sv
- vpr_sdc.f
- vpr_sdc/
  - vpr_sdc_finit.v
  - vpr_sdc_cdc.v
  - vpr_sdc_bist.v
  - vpr_sdc_bist_interpol.v
- vpr_sdc_wrap/
- vpr_sdc_wrap_register/
- vpr_sdc_wrap_regflag/
- vpr_sdc_wrap_sramslv/
- vpr_sdc_wrap_gramslv/
- vpr_sdc_wrap_grammst/
- vpr_sdc_wrap_ip08/
- vpr_sdc_wrap_ip15/
- vpr_sdc_wrap_ip23/

특징:
- register, regflag, SRAM slave, GRAM slave/master, IP-specific wrapper가 세분화되어 있다.
- axi4_sram_ctl 하위에 AXI SRAM controller 관련 FIFO/ECC/router/register/rbeat/wbeat 모듈이 있다.
- IP08/IP15/IP23 등 project-specific processing block이 있다.

분석:
- VPR/SDC는 이 프로젝트의 핵심 영상/보정/메모리 subsystem 중 하나로 보인다.
- SRAM/GRAM 접근과 DMA/AXI 경로가 복잡하므로 debug 시 vpr_sdc_wrap_gramslv, gramslv/grammst, axi4_sram_ctl을 우선 확인해야 한다.

### 5.7 VIN 구조: source/rtl/vin

주요 구성:

- vin.v
- vin_sram.v
- vin_pic*.v
- vin_dsc*.v
- vin_cdc*.v
- vin_apb_reg_ctrl.sv
- ip_scl/
- ip_pixelshift/
- ip_orbit/
- ip_g2g/
- ip_foveation/
- ip_dsc/
- ip_dsc_6p/
- ip_letterbox/

분석:
- VIN은 입력 영상 처리 pipeline이다.
- DSC decode, scaler, orbit, pixel shift, foveation, G2G, letterbox 등 영상 처리 IP가 포함된다.
- ip_dsc와 ip_dsc_6p가 공존하므로 variant 또는 lane/pixel parallelism 차이일 수 있다.

### 5.8 VOUT/GPO 구조: source/rtl/vout

주요 구성:

- vout.v
- vout_video_cdc.v
- source/
  - center/
  - side/
- gpo/

source/center:
- source_center_top.v
- timing/sync/gamma/asbc/fsm/dma_request 등

source/side:
- source_sx_top.sv
- source_sx_sfr, timing_control, sync_generator, rt, output, fsm 등

gpo:
- vout_gpo_top.sv
- vout_gpo_reg/slv/sel/pwr/cdc/eq/dotc/dc_opt 등
- d001a_vout_gpo_*.sv

분석:
- VOUT은 panel/source output path 담당이다.
- center/side source driver 제어가 분리되어 있고 GPO 제어 logic도 포함한다.

### 5.9 VGEN 구조: source/rtl/vgen

주요 구성:

- vgen.v
- vgen.f
- vgen_ctl, vgen_dec, vgen_cdc, vgen_cg
- vgen_vdo*, vgen_linefg
- vgen_arp/

분석:
- video pattern/timing/frame generation 또는 adaptive refresh pipeline 성격으로 보인다.
- vgen_arp 하위에는 arp_ctrl, arp_te, arp_img_compare 등 ARP 관련 제어가 있다.

### 5.10 MIPI 구조: source/rtl/mipi

주요 구성:

- mipi.f
- mipi_dsi_top.v
- dsi_link, dsi_ecc, dsi_return_crc, dsi_clkcon, dsi2reg.sv
- dphy_digital, dphy_d0ctrl, dphy_d123ctrl, dphy_lp11, dphy_timer
- dsi_axi4_rd_mst.sv

분석:
- MIPI DSI digital RX/TX 또는 DSI interface logic이다.
- DPHY digital control과 DSI packet/CRC/ECC/link 처리가 포함된다.
- AXI read master 연동도 존재한다.

### 5.11 PMIC/PWRCTL/EXCTL/ERRFG/OSC/TEMP

PMIC:
- source/rtl/pmic
- swire, i2c, axi 하위 구조
- pmic.v, pmic_pad_ctl.v, pmic_cdc_from_vpr.v

PWRCTL:
- source/rtl/pwrctl
- pwrctl.v, pwr_sync, pwr_dly_gen, pwr_div_gen
- ldoctl 하위 analog LDO control

EXCTL:
- source/rtl/exctl
- spi, pmic, acon 등 외부 제어 block

ERRFG:
- source/rtl/errfg
- err flag, voltage detect, checksum FSM

OSC:
- source/rtl/osc
- osc_cont_top, osc_sscg, aofc_top, aofc_* 관련

TEMPCTL:
- tempctl.v, temp_dts1/2, temp_clk_gen

분석:
- analog/power/PMIC/oscillator/temp 관련 digital control logic이 상당히 큰 비중을 차지한다.
- 초기 reset/clock/power sequence debug 시 sys + osc + pwrctl + pmic + padcon을 함께 봐야 한다.

### 5.12 SRAM 구조

경로:

- source/rtl/sram
- source/rtl/sram_1002
- source/rtl/sram_4001

특징:
- mem_wrap.f가 각각 존재한다.
- 현재 rtl.f에서는 source/rtl/sram/mem_wrap.f가 활성화되어 있고 sram_1002는 주석 처리되어 있다.
- ra*/rf* macro wrapper/model 파일이 다수 존재한다.
- ip08_gram, ip15_gram, ip23_gram wrapper가 존재한다.

분석:
- SRAM macro variant가 여러 project/revision별로 공존한다.
- 현재 compile 선택은 source/rtl/sram이다.
- 다른 variant로 전환할 때는 rtl.f와 model_sram.f의 정합성을 확인해야 한다.

## 6. Model 구조: source/model

파일 수: 6개

- model.f
- model_sram.f
- model_pre.f
- model_post.f
- D2ID001A_TOP.v
- D2ID001A_DIGITAL_TOP.v

D2ID001A_TOP.v:
- module D2ID001A_TOP
- real nettype 사용:
  - SNPS: snps_msv_nettype_pkg / wreal_rf_avg
  - else: cds_rnm_pkg / CDS_res_wrealavg
- MIPI realnet, power realnet, analog pad, logic pad, source/GPO output 등 칩 wrapper 포트 포함

model.f:
- UMC40HV STD cell model include
- model_sram.f include
- D2ID001A_TOP.v / D2ID001A_DIGITAL_TOP.v include
- 외부 pad model path
- MIPI PHY model path
- VROM roms_lib.v
- OSC model
- analog delay model
- GPIO model
- eMemory OTP model
- temp sensor model

분석:
- source/model은 RTL-only가 아니라 mixed-signal/top-chip simulation wrapper 성격이다.
- xrun/VCS 또는 mixed-signal capable simulator 환경 의존성이 크다.
- /PROJECT/... absolute path가 많아, 로컬 단독 빌드는 어렵고 회사 서버 환경 기준이다.

## 7. Testbench 구조: tbench

파일 수: 415개

### 7.1 TB top

주요 파일:

- tbench/tb_top.sv
- tbench/host.sv
- tbench/tbench.f
- tbench/tbench_pre.f
- tbench/tbench_post.f

TB 구조 주석에 따르면 tb_top.sv는 다음을 포함한다.

- Pad signal between chip and host/flash
- Clock
- Instance
  - TB Inst: Host
  - DUT Inst: Chip
  - TB Inst: Flash
  - TB Inst: PMIC

module:
- tb_top

분석:
- tb_top은 chip-level simulation 최상위다.
- DUT instance는 README 기준 chip 이름으로 쓰는 것으로 보인다.
- host, flash, PMIC model을 TB 환경에서 붙인다.

### 7.2 TB filelist: tbench/tbench.f

구성 순서:

1. incdir
   - ${pjt_path}
   - ${pjt_path}/tbench/flash

2. MIPI TX model
   - tbench/mipi_tx/mipi_tx.f

3. Packages
   - tbench/package/package.f
   - tbench/random_model/random_model.f

4. Checkers
   - tbench/checker/checker.f

5. Models
   - tbench/flash/flash_model.v
   - 외부 FLASH model absolute path 및 -y library path
   - i2c_slave models
   - PMIC model

6. Coverage
   - tbench/coverage/coverage.f

7. Injector
   - clock_delay_injector.sv 기본 enabled
   - sync_2step_delay_injector.sv 기본 enabled, SYNC_2STEP_DELAY define로 gate
   - sync_jitter/mipi_hs_miss injector는 disabled by default

8. MCU/IBEX simulation support
   - rv_sim_only prim_pkg, prim_ram, ibex_tracer, simulator_ctrl

9. Handler
   - vrom_hashtag_define.v
   - vrom_handler.sv
   - uvm_handler.sv

10. TB top
   - host.sv
   - tb_top.sv

분석:
- tbench.f는 chip-level simulation compile entry다.
- 외부 flash model path와 ibex simulation model path가 포함된다.
- clock/CDC injector가 기본 적용되어 RTL sim stress/realism을 높이는 구조다.

### 7.3 Scenario 구조

경로:

- tbench/scenario

README 핵심:
- legacy scenario 구조에서 test flow/helper/random logic을 분리했다.
- scenario_main.v에서 compile-time define으로 scenario를 선택한다.
- 기본 구조:

scenario/<name>/
├── <name>_helper.svh
├── <name>_scenario.svh
└── <name>_random.svh optional

Layer 역할:
- Scenario Layer: test entry 및 전체 flow 제어, sequence 실행, DUT interaction trigger
- Helper Layer: 공통 로직/초기화/유틸/model/randomization/argument parsing
- Random Layer: random class/constraint 기반 stimulus

적용 scenario:
- aofc
- dcc
- dbi
- err_fg
- flash
- gpo
- i2c
- mcu
- mipi
- otp
- pmic
- power
- rt
- sdc_ip
- sram
- src
- te
- top
- ucs
- vgen
- vin

분석:
- 검증 시나리오가 기능 단위로 잘 분리되어 있다.
- 신규 scenario 작성 시 helper/scenario/random 계층 규칙을 따라야 한다.
- scenario_main.v가 test selection의 중심이다.

### 7.4 Checker 구조

경로:

- tbench/checker

주요 하위:

- vpr_sdc
- sys
- vin
- vout
- video
- vgen
- te
- src/source
- pwrctl
- pmic
- otp
- mipi
- gpo
- flash
- err_flag
- dcdc
- dcc
- chip
- sync
- autogen

분석:
- subsystem별 checker가 촘촘히 구성되어 있다.
- sys 쪽에는 sys_fsm_checker, dma_trace, mipi_trace 등 debug용 checker/trace가 있다.
- gpo/source/vpr_sdc checker가 특히 많아 영상/패널 출력 검증에 비중이 크다.

### 7.5 Coverage 구조

경로:

- tbench/coverage

주요 파일:

- coverage.f
- vpr_sdc_finit_coverage.sv
- vgen_coverage.sv
- sys_fsm_coverage.sv
- sram_coverage.sv
- sdc_core_coverage.sv
- pixel_align_coverage.sv
- otp_coverage.sv
- mbist_coverage.sv
- gpo_coverage.sv
- coverage_macro.sv

분석:
- 기능 coverage가 block-level로 분리되어 있다.
- sys_fsm, SRAM, OTP, MBIST, GPO, VPR/SDC 등 검증 target이 명확하다.

### 7.6 Package 구조

경로:

- tbench/package

package.f 포함:

- svlib_pkg.sv
- d2i_pkg.sv
- cmd_map_pkg.sv
- img_pkg.sv
- dsc_pkg.sv
- reg_access_pkg.sv
- uvm_report_pkg.sv
- deposit_pkg.sv
- test-specific pkg:
  - sdc_ip_test_pkg.sv
  - src_test_pkg.sv
  - hlpm_test_pkg.sv
  - chop_test_pkg.sv
- tv_pkg.sv

분석:
- testbench 공통 utility, image/DSC 처리, register access abstraction이 package로 묶여 있다.
- UVM full environment라기보다는 SV package + handler + scenario 기반 구조로 보인다.

## 8. Firmware/SW 구조

### 8.1 source/sw

파일 수: 91개

주요 구성:

- fw_link_check.py
- fw_link_baseline.json
- fw_link_report.md
- rules.mk
- mmap_define/
  - cmd_map_define.h
  - mem_map_define.h
- vrom2_code_gen/
  - vrom2_code_gen.c/cfg
  - vrom2_code*.txt
  - hashcode2model.sh
- otp/
  - otp_load
  - otp_main_prog
  - otp_trim_prog
  - gpo_reg_init
  - common
- normal/
  - app
  - include
  - common
- finit_1st/
- finit_2nd/

fw_link_report.md 기준 FW units:

- finit_1st
- finit_2nd
- otp_load
- otp_main_prog
- otp_trim_prog
- normal

Section size는 모두 OK이나 baseline hash 대비 변경 사항이 5개 있다.

분석:
- VROM/OTP/normal boot 또는 init firmware가 프로젝트 내 관리된다.
- FW binary/hash/model link를 검증하는 flow가 있다.
- RTL의 VROM/MCU/DMA/OTP/flash path와 SW가 밀접하게 연결된다.

### 8.2 source/sw_wa

파일 수: 23개

주요 구성:

- common/
- vrom2flash_otp/
  - otp.c/common/load
  - flash_auto_main.c
  - flash_init_read.c
  - gpo_reg_init.c
  - otp.bin/otp.hex
  - Makefile/link.ld/common.mk

분석:
- workaround 또는 alternate VROM-to-flash/OTP flow로 보인다.
- flash/otp bring-up 관련 디버그 시 source/sw와 함께 확인해야 한다.

## 9. Compile/Simulation 의존성

### 9.1 pjt_path 기준 상대 include

README와 filelist 모두 ${pjt_path}를 기준으로 한다.

- tbench README는 include 파일들을 ${pjt_path} 기준 상대 경로로 관리한다고 설명한다.
- 예: `include "tbench/scenario/scenario_sub_src.v"`

분석:
- compile/sim 실행 시 pjt_path environment variable이 필수일 가능성이 높다.
- bin/README는 project top의 environment.env source를 언급한다. 현재 검색 목록에서는 environment.env 파일은 확인되지 않았지만, 환경 설정 의존성이 있다는 점은 중요하다.

### 9.2 절대 경로 dependency

여러 filelist에 /PROJECT, /app 경로가 존재한다.

예:

- /PROJECT/LDK/UMC40HV/STD_7T/...
- /PROJECT/D2ID001A-00/Digital/Model/...
- /PROJECT/D2I1002A-00/Digital/ip_from_ext/...
- /app/synopsys/syn/V-2023.12-SP3/...

분석:
- 로컬 단독 simulation은 어렵다.
- 정식 compile/run은 회사 CAD 서버 또는 동일 mount 환경이 필요하다.
- 오픈소스 simulator로는 전체 chip-level compile보다 block-level lint/check가 현실적이다.

## 10. 주요 데이터/제어 흐름 가설

현재 구조 기준으로 볼 때 전체 시스템 흐름은 다음과 같이 볼 수 있다.

### 10.1 Boot / 초기화 경로

RESX/pad/reset
→ D2ID001A_TOP / D2ID001A_DIGITAL_TOP
→ logic_scan_top / logic_top
→ sys/sys_fsm
→ VROM/OTP/Flash/MCU init
→ DMA/AXI bus
→ RT/register setup
→ 각 영상/전원/패널 subsystem enable

관련 주요 경로:

- source/rtl/peri/sys
- source/rtl/peri/vrom_slv
- source/rtl/peri/otp_slv
- source/rtl/peri/flash_slv
- source/rtl/peri/mcu
- source/rtl/peri/dma
- source/rtl/bus
- source/sw

### 10.2 Register/command 경로

Host/I2C/SPI/DBI/MIPI command
→ sys/exi 또는 mipi/dsi2reg
→ bus/APB/AXI bridge
→ rt/cmd_reg_if
→ subsystem register bank
→ vpr_sdc/vin/vout/vgen/pwrctl/pmic/etc

관련 주요 경로:

- source/rtl/rt
- source/rtl/peri/sys/sys_exi*
- source/rtl/mipi/dsi2reg.sv
- source/rtl/bus
- tbench/package/reg_access
- tbench/task/task_reg_*.v/sv

### 10.3 Video data path

MIPI/VIN input
→ VIN processing: DSC/scaler/orbit/pixelshift/G2G/foveation
→ VPR/SDC + GRAM/SRAM/AXI path
→ VGEN/timing/pattern path
→ VOUT/source/GPO output

관련 주요 경로:

- source/rtl/mipi
- source/rtl/vin
- source/rtl/vpr_sdc
- source/rtl/vgen
- source/rtl/vout
- source/rtl/common/fifo,sync,sramslv

### 10.4 Power/analog control path

sys/pads/RT command
→ pwrctl/pmic/exctl/osc/tempctl
→ analog pad/model/top-level pins

관련 주요 경로:

- source/rtl/pwrctl
- source/rtl/pmic
- source/rtl/exctl
- source/rtl/osc
- source/rtl/tempctl
- source/model
- tbench/pmic

## 11. Debug/분석 관점에서의 우선순위

### 11.1 Chip-level bring-up / reset hang

우선 확인 경로:

1. tbench/tb_top.sv
2. source/model/D2ID001A_TOP.v
3. source/model/D2ID001A_DIGITAL_TOP.v
4. source/rtl/top/logic_scan_top.v
5. source/rtl/top/logic_top.sv
6. source/rtl/peri/sys/sys_fsm*.v
7. source/rtl/peri/vrom_slv
8. source/rtl/peri/dma
9. source/rtl/bus/axi4_sys 또는 관련 bus target
10. source/sw/fw_link_report.md 및 VROM/FW image

### 11.2 Register read/write 문제

우선 확인 경로:

1. tbench/package/reg_access
2. tbench/task/task_reg_*.v/sv
3. source/rtl/rt/rt.v
4. source/rtl/rt/rt_slv.v
5. source/rtl/rt/rt_dec.sv
6. source/rtl/rt/cmd_reg_if/*.sv
7. source/rtl/rt/cmd_reg_if_wire_inc/*.sv
8. subsystem wrapper의 RT 연결

### 11.3 SRAM/GRAM/DMA 문제

우선 확인 경로:

1. source/rtl/sram/mem_wrap.f
2. source/rtl/vpr_sdc/vpr_sdc_wrap_gramslv
3. source/rtl/vpr_sdc/vpr_sdc_wrap_grammst
4. source/rtl/vpr_sdc/vpr_sdc_wrap_gramslv/axi4_sram_ctl
5. source/rtl/peri/dma
6. source/rtl/bus/axi4_vprgram 또는 axi4_vprsram
7. tbench/checker/sys/dma_trace.sv
8. tbench/checker/vpr_sdc

### 11.4 Scenario 추가/수정

따라야 할 구조:

- tbench/scenario/<name>/<name>_scenario.svh
- tbench/scenario/<name>/<name>_helper.svh
- tbench/scenario/<name>/<name>_random.svh optional
- tbench/scenario/scenario_main.v define mapping 수정
- 필요 시 tbench/package/test 또는 random_model 추가

### 11.5 Compile 문제

우선 확인 파일:

1. bin/compile.py
2. source/rtl/rtl.f
3. source/model/model.f
4. tbench/tbench.f
5. tbench/package/package.f
6. tbench/checker/checker.f
7. tbench/coverage/coverage.f
8. source/rtl/sim_ext_ip.f / syn_ext_ip.f

## 12. 구조상 리스크/주의점

1. 절대 경로 의존성
   - /PROJECT, /app/synopsys 경로가 다수 존재한다.
   - 환경 mount가 안 맞으면 compile 실패 가능성이 높다.

2. SRAM variant 혼재
   - source/rtl/sram, sram_1002, sram_4001가 공존한다.
   - rtl.f에서는 source/rtl/sram이 활성화되어 있으나 variant 전환 시 filelist 정합성을 반드시 확인해야 한다.

3. bus generated source 규모
   - source/rtl/bus 아래 DW_axi generated source가 매우 많다.
   - 직접 수정은 위험하고 generator/config 원천 확인이 필요할 수 있다.

4. logic_top 대형 단일 파일
   - logic_top.sv가 6499 lines로 매우 크다.
   - top-level change는 영향 범위가 크므로 port/interface diff 중심으로 관리해야 한다.

5. scenario refactoring 진행 이력
   - scenario README에 legacy에서 new structure로 점진 리팩토링했다고 되어 있다.
   - 일부 legacy path 또는 eds_scen과 중복 개념이 남아 있을 수 있다.

6. mixed-signal/RNM dependency
   - D2ID001A_TOP.v는 SNPS/CDS RNM nettype을 사용한다.
   - simulator 선택 및 compile define에 따라 동작이 달라질 수 있다.

7. FW hash 변경 상태
   - fw_link_report.md에 5개 unit hash 변경이 기록되어 있다.
   - RTL/FW co-sim mismatch를 볼 때 FW baseline 변경 여부를 함께 확인해야 한다.

## 13. 추천 문서화/후속 작업

앞으로 doc/ 아래에 다음 문서를 누적 생성하는 것을 추천한다.

1. doc/compile_flow_analysis.md
   - compile.py, sim.py, rtl.f, model.f, tbench.f의 실제 실행 관계 분석

2. doc/top_hierarchy_analysis.md
   - D2ID001A_TOP → D2ID001A_DIGITAL_TOP → logic_scan_top → logic_top hierarchy 정리

3. doc/reset_boot_flow_analysis.md
   - RESX, sys_fsm, VROM, DMA, MCU, FW init path 분석

4. doc/register_map_path_analysis.md
   - host command → rt/cmd_reg_if → subsystem register path 분석

5. doc/vpr_sdc_memory_path_analysis.md
   - VPR/SDC, GRAM/SRAM, AXI, DMA path 분석

6. doc/testbench_scenario_guide.md
   - scenario 추가/수정 방법과 naming convention 정리

## 14. 결론

이 프로젝트는 다음과 같은 구조로 이해하는 것이 가장 적절하다.

- source/rtl/rtl.f가 RTL compile 중심
- source/model/model.f가 chip/model/mixed-signal wrapper 중심
- tbench/tbench.f가 검증 환경 compile 중심
- tbench/scenario/scenario_main.v가 test scenario 선택 중심
- bin/compile.py, bin/sim.py, bin/regress_*.py가 실행 자동화 중심
- source/sw가 VROM/OTP/normal/finit firmware와 RTL boot path 연결 중심

따라서 분석/디버그 작업 시에는 단일 파일보다 다음 3개 축을 함께 추적해야 한다.

1. filelist 축
   - 어떤 파일이 실제 compile되는가

2. hierarchy 축
   - top/model/tb에서 어떤 instance 경로로 연결되는가

3. scenario/FW 축
   - 어떤 scenario define과 FW image가 해당 동작을 만들고 있는가

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
