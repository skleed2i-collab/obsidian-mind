---
date: 2026-06-10
description: "D2ID001A chip-level Verilog/SystemVerilog testbench의 filelist, top, host, checker, package, task 구조 요약."
tags:
  - reference
  - rtl
  - verification
  - testbench
  - d2id001a
status: active
quarter: Q2-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/TB/tbench/TBENCH_STRUCTURE_REVIEW.md
---

# D2ID001A tbench 구조 리뷰

상위 index: [[D2ID001A Reference]]  
관련 상세: [[D2ID001A scenario 내부 구조 상세]]

- 생성 위치: `/home/tim/pjt_work/D2ID001A/TB/tbench`
- 생성 시각: 2026-06-09 20:49:40
- 목적: tbench 폴더의 내부 구조, filelist 기준 compile 구성, 주요 파일 역할을 한눈에 파악하기 위한 정리 문서

## 1. 전체 구조 요약

이 폴더는 block 단위 단순 TB가 아니라 `D2ID001A_TOP`을 대상으로 하는 chip-level 통합 testbench 구조이다.

핵심 흐름은 다음과 같다.

```text
tbench.f
  -> mipi_tx/mipi_tx.f
  -> package/package.f
  -> random_model/random_model.f
  -> checker/checker.f
  -> flash / i2c_slave / pmic / injector / ibex sim model
  -> vrom_handler.sv / uvm_handler.sv
  -> host.sv
       -> task/* include
       -> scenario/scenario_main.v include
  -> tb_top.sv
       -> host U_HOST
       -> D2ID001A_TOP chip
       -> flash_model / PMIC model
```

## 2. 폴더별 규모

| 폴더 | 파일 수 | 크기 | 주요 확장자 |
|---|---:|---:|---|
| `.` | 13 | 102.3 KB | .f:6, .sv:5, .md:1, .txt:1 |
| `checker` | 105 | 1418.6 KB | .sv:93, .F:8, .v:2, .f:1, .vf:1 |
| `coverage` | 11 | 2225.0 KB | .sv:10, .f:1 |
| `define` | 4 | 9.7 KB | .vh:4 |
| `eds_scen` | 61 | 1600.7 KB | .v:60, .sv:1 |
| `firmware` | 34 | 1820.8 KB | .h:13, .c:5, .d:3, .o:3, .dump:1, .ld:1 |
| `flash` | 1 | 28.5 KB | .v:1 |
| `function` | 2 | 6.6 KB | .svh:2 |
| `i2c_slave` | 2 | 19.0 KB | .v:2 |
| `injector` | 3 | 7.9 KB | .sv:3 |
| `mipi_tx` | 10 | 112.4 KB | .sv:7, .v:2, .f:1 |
| `misc` | 9 | 196.6 KB | .sv:7, .v:2 |
| `monitor` | 5 | 43.3 KB | .sv:2, .F:1, .pl:1, .md:1 |
| `package` | 47 | 3114.1 KB | .svh:19, .sv:18, .md:3, .f:3, (noext):2, .h:1 |
| `pmic` | 4 | 222.4 KB | .v:3, .vp:1 |
| `random_model` | 11 | 81.1 KB | .sv:6, .svh:4, .f:1 |
| `scenario` | 70 | 2498.4 KB | .svh:47, .md:20, .v:1, .py:1, .txt:1 |
| `task` | 35 | 4353.5 KB | .v:27, .sv:8 |

## 3. Top-level 파일

### `tbench.f`
- line 수: 120
- 역할: 기본 compile filelist. package/random_model/checker/model/handler/host/tb_top을 연결한다.

### `tbench_pre.f`
- line 수: 190
- 역할: pre simulation용 확장 filelist로 보이며 package/adaptor 구성이 더 직접적으로 펼쳐져 있다.

### `tbench_post.f`
- line 수: 188
- 역할: post simulation용 확장 filelist로 보이며 random_model adaptor 및 post 관련 구성이 더 포함된다.

### `tb_top.sv`
- line 수: 585
- 역할: chip-level top TB. pad/wire/clock 선언, host/DUT/flash/PMIC instance를 포함한다.
- module: tb_top
- 주요 조건 define:
  - TEST_VECTOR_BACK_ANNO
  - DUAL_CHIP

### `host.sv`
- line 수: 555
- 역할: stimulus shell. package import, define/function/task/scenario/test-vector include를 담당한다.
- module: host
- 주요 include:
  - uvm_macros.svh
  - tbench/define/inst_path_define.vh
  - tbench/define/max_param_define.vh
  - tbench/define/index_name_define.vh
  - tbench/define/common_util_define.vh
  - tbench/function/func_common.svh
  - tbench/function/func_power.svh
  - tbench/misc/image_dump.v
  - tbench/task/task_util.v
  - tbench/task/task_mipi.v
  - tbench/task/task_dbi.v
  - tbench/task/task_ucs.v
  - tbench/task/task_flash.sv
  - tbench/task/task_otp.v
  - tbench/task/task_gpo.v
  - tbench/task/task_gpo_reg.sv
  - tbench/task/task_reg_command.v
  - tbench/task/task_reg_command_rd_val.sv
  - tbench/task/task_reg_sram.sv
  - tbench/task/task_force.v
  - tbench/task/task_dma.v
  - tbench/task/task_sram.v
  - tbench/task/task_pmic.sv
  - tbench/task/task_sdc_ip.sv
  - tbench/task/task_core.v
  - tbench/task/task_vrr.v
  - tbench/task/task_chop.v
  - tbench/task/task_vcd_reg_set_all.v
  - tbench/task/task_man_pwr.v
  - tbench/task/task_src.v
  - tbench/task/task_te.sv
  - tbench/task/task_bist.v
  - tbench/task/task_asbc.v
  - tbench/task/task_gamma.v
  - tbench/scenario/scenario_main.v
  - ... (19개 생략)
- 주요 조건 define:
  - _OSC_TRIM_EXCLK_PERIOD
  - TEST_VECTOR_ON
  - STRINGIFY
  - VERDI_BROWSE
  - __SIM_RTL__
  - TEST_VECTOR_BACK_ANNO
  - HTOL_VEC
  - NO_IMAGE_FORCE

### `vrom_handler.sv`
- line 수: 67
- 역할: VROM/ROMCODE 관련 handler module.
- module: vrom_handler
- 주요 조건 define:
  - STRINGIFY
  - DUAL_CHIP

### `uvm_handler.sv`
- line 수: 29
- 역할: UVM report customization handler. trim_report_server를 적용한다.
- module: uvm_handler
- 주요 include:
  - uvm_macros.svh

### `sdf_handler.sv`
- line 수: 83
- 역할: SDF annotation 관련 handler.
- module: sdf_handler
- 주요 조건 define:
  - __SIM_SDF_SS__
  - __SIM_SDF_FF__
  - SRC_RTL

### `tbench_structure.txt`
- line 수: 500
- 역할: 기존 raw snapshot 문서. 일부 define 이름은 실제 scenario_main.v와 차이가 있다.

## 4. Filelist별 active 구성

### `tbench.f`
- active entry 수: 39
  - +incdir+${pjt_path}
  - +incdir+${pjt_path}/tbench/flash
  - -f ${pjt_path}/tbench/mipi_tx/mipi_tx.f
  - -f ${pjt_path}/tbench/package/package.f
  - -f ${pjt_path}/tbench/random_model/random_model.f
  - -f ${pjt_path}/tbench/checker/checker.f
  - ${pjt_path}/tbench/flash/flash_model.v
  - /PROJECT/D2ID001A-00/Digital/ip_from_ext/FLASH/WB_16MB/W25Q16JWxxIQ_V1.0.v
  - /PROJECT/D2ID001A-00/Digital/ip_from_ext/FLASH/WB_32MB/W25Q32JW.v
  - +libext+.v
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_4mbit
  - -y /PROJECT/D2ID001A-00/Digital/ip_from_ext/FLASH/
  - ${pjt_path}/tbench/i2c_slave/i2c_slv.v
  - ${pjt_path}/tbench/i2c_slave/i2c_s.v
  - ${pjt_path}/tbench/pmic/PMIC.v
  - ${pjt_path}/tbench/misc/sdc_ip_rgb_overrider.sv
  - ${pjt_path}/tbench/injector/clock_delay_injector.sv
  - +incdir+${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/prim_pkg.sv
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/prim_ram_1p_pkg.sv
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/prim_ram_1p.sv
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/ibex_tracer_pkg.sv
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/ibex_tracer.sv
  - ${pjt_path}/source/rtl/peri/mcu/ibex/rv_sim_only/simulator_ctrl.sv
  - ${pjt_path}/tbench/vrom_handler.sv
  - ${pjt_path}/tbench/uvm_handler.sv
  - ${pjt_path}/tbench/host.sv
  - ${pjt_path}/tbench/tb_top.sv

### `tbench_pre.f`
- active entry 수: 41
  - +incdir+${pjt_path}
  - +incdir+${pjt_path}/tbench/flash
  - -f ${pjt_path}/tbench/mipi_tx/mipi_tx.f
  - +incdir+${pjt_path}/dv/lib/svlib/src
  - ${pjt_path}/tbench/package/svlib/src/svlib_pkg.sv
  - +incdir+\$\{pjt_path\}/tbench/package/img
  - \$\{pjt_path\}/tbench/package/img/img_pkg.sv
  - +incdir+${pjt_path}/tbench/package/dsc
  - ${pjt_path}/tbench/package/dsc/dsc_pkg.sv
  - ${pjt_path}/tbench/package/d2i_pkg.sv
  - ${pjt_path}/tbench/package/cmd_map_pkg.sv
  - ${pjt_path}/tbench/package/test/sdc_ip_test_pkg.sv
  - ${pjt_path}/tbench/misc/sdc_ip_rgb_overrider.sv
  - ${pjt_path}/tbench/package/test/src_test_pkg.sv
  - ${pjt_path}/tbench/package/test/hlpm_test_pkg.sv
  - ${pjt_path}/tbench/package/test/chop_test_pkg.sv
  - ${pjt_path}/tbench/package/deposit_pkg.sv  // force & deposit function
  - ${pjt_path}/tbench/package/uvm_report_pkg.sv // customize uvm_log message
  - ${pjt_path}/tbench/package/reg_access/reg_access_pkg.sv
  - ${pjt_path}/tbench/random_model/base/base_model_pkg.sv
  - ${pjt_path}/tbench/random_model/panel/panel_model_pkg.sv
  - ${pjt_path}/tbench/random_model/vgen/vgen_model_pkg.sv
  - ${pjt_path}/tbench/random_model/gpo/gpo_model_pkg.sv
  - ${pjt_path}/tbench/flash/flash_model.v
  - +libext+.v
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_4mbit
  - ${pjt_path}/tbench/i2c_slave/i2c_slv.v
  - ${pjt_path}/tbench/i2c_slave/i2c_s.v
  - ${pjt_path}/tbench/pmic/PMIC.v
  - ${pjt_path}/tbench/tb_top.sv

### `tbench_post.f`
- active entry 수: 45
  - +incdir+${pjt_path}
  - +incdir+${pjt_path}/tbench/flash
  - -f ${pjt_path}/tbench/mipi_tx/mipi_tx.f
  - +incdir+${pjt_path}/dv/lib/svlib/src
  - ${pjt_path}/tbench/package/svlib/src/svlib_pkg.sv
  - +incdir+\$\{pjt_path\}/tbench/package/img
  - \$\{pjt_path\}/tbench/package/img/img_pkg.sv
  - +incdir+${pjt_path}/tbench/package/dsc
  - ${pjt_path}/tbench/package/dsc/dsc_pkg.sv
  - ${pjt_path}/tbench/package/d2i_pkg.sv
  - ${pjt_path}/tbench/package/cmd_map_pkg.sv
  - ${pjt_path}/tbench/package/test/sdc_ip_test_pkg.sv
  - ${pjt_path}/tbench/misc/sdc_ip_rgb_overrider.sv
  - ${pjt_path}/tbench/package/test/src_test_pkg.sv
  - ${pjt_path}/tbench/package/test/hlpm_test_pkg.sv
  - ${pjt_path}/tbench/package/test/chop_test_pkg.sv
  - ${pjt_path}/tbench/package/deposit_pkg.sv  // force & deposit function
  - ${pjt_path}/tbench/package/uvm_report_pkg.sv // customize uvm_log message
  - ${pjt_path}/tbench/package/reg_access/reg_access_pkg.sv
  - ${pjt_path}/tbench/random_model/base/base_model_pkg.sv
  - ${pjt_path}/tbench/random_model/panel/panel_model_pkg.sv
  - ${pjt_path}/tbench/random_model/vgen/vgen_model_pkg.sv
  - ${pjt_path}/tbench/random_model/gpo/gpo_model_pkg.sv
  - ${pjt_path}/tbench/random_model/panel/panel_adaptor.sv
  - ${pjt_path}/tbench/random_model/vgen/vgen_adaptor.sv
  - ${pjt_path}/tbench/random_model/host/host_model_pkg.sv
  - ${pjt_path}/tbench/random_model/host/host_adaptor.sv
  - ${pjt_path}/tbench/flash/flash_model.v
  - +libext+.v
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/winbond_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_16mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_32mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/macronix_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/GD_4mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_8mbit
  - -y /PROJECT/D2I3101A-00/Digital/ip_from_ext/FLASH/PUYA_4mbit
  - ${pjt_path}/tbench/i2c_slave/i2c_slv.v
  - ${pjt_path}/tbench/i2c_slave/i2c_s.v
  - ${pjt_path}/tbench/pmic/PMIC.v
  - ${pjt_path}/tbench/tb_top.sv

### `package/package.f`
- active entry 수: 17
  - +incdir+${pjt_path}/tbench/package/svlib/src
  - ${pjt_path}/tbench/package/svlib/src/svlib_pkg.sv
  - ${pjt_path}/tbench/package/d2i_pkg.sv
  - ${pjt_path}/tbench/package/cmd_map_pkg.sv
  - +incdir+${pjt_path}/tbench/package/img
  - +incdir+${pjt_path}/tbench/package/dsc
  - +incdir+${pjt_path}/tbench/package/reg_access
  - ${pjt_path}/tbench/package/img/img_pkg.sv
  - ${pjt_path}/tbench/package/dsc/dsc_pkg.sv
  - ${pjt_path}/tbench/package/reg_access/reg_access_pkg.sv
  - ${pjt_path}/tbench/package/uvm_report_pkg.sv
  - ${pjt_path}/tbench/package/deposit_pkg.sv
  - ${pjt_path}/tbench/package/test/sdc_ip_test_pkg.sv
  - ${pjt_path}/tbench/package/test/src_test_pkg.sv
  - ${pjt_path}/tbench/package/test/hlpm_test_pkg.sv
  - ${pjt_path}/tbench/package/test/chop_test_pkg.sv
  - ${pjt_path}/tbench/package/tv_pkg.sv

### `random_model/random_model.f`
- active entry 수: 5
  - ${pjt_path}/tbench/random_model/base/base_model_pkg.sv
  - ${pjt_path}/tbench/random_model/panel/panel_model_pkg.sv
  - ${pjt_path}/tbench/random_model/vgen/vgen_model_pkg.sv
  - ${pjt_path}/tbench/random_model/host/host_model_pkg.sv
  - ${pjt_path}/tbench/random_model/gpo/gpo_model_pkg.sv

### `checker/checker.f`
- active entry 수: 15
  - ${pjt_path}/tbench/checker/tb_mask.sv
  - ${pjt_path}/tbench/checker/tb_define.sv
  - ${pjt_path}/tbench/checker/tb_config.sv
  - ${pjt_path}/tbench/checker/tb_watchdog.sv
  - ${pjt_path}/tbench/checker/sys/sys_exi_i2c_monitor.sv
  - ${pjt_path}/tbench/checker/sys/sys_exi_i2c_checker.sv
  - ${pjt_path}/tbench/checker/video/video_sync_checker.sv
  - ${pjt_path}/tbench/checker/video/video_data_checker.sv
  - ${pjt_path}/tbench/checker/video/video_checker.sv
  - ${pjt_path}/tbench/checker/vgen/vgen_checker.sv
  - ${pjt_path}/tbench/checker/flash/flash_scoreboard.sv
  - ${pjt_path}/tbench/checker/flash/flash_pad_checker.sv
  - -F ${pjt_path}/tbench/checker/gpo/checker_list.F
  - -F ${pjt_path}/tbench/checker/src/checker_list.F
  - ${pjt_path}/tbench/checker/dcc/dcc_checker.sv

### `mipi_tx/mipi_tx.f`
- active entry 수: 9
  - ${pjt_path}/tbench/mipi_tx/mipi_pkg.sv
  - ${pjt_path}/tbench/mipi_tx/tranif_rnm.sv
  - ${pjt_path}/tbench/mipi_tx/mipi_hostmodel.sv
  - ${pjt_path}/tbench/mipi_tx/mipi_tx_phy_afe.sv
  - ${pjt_path}/tbench/mipi_tx/mipi_tx_lnsw.v
  - ${pjt_path}/tbench/mipi_tx/mipi_tx_driver.sv
  - ${pjt_path}/tbench/mipi_tx/mipi_tx_fsm.sv
  - ${pjt_path}/tbench/mipi_tx/mipi_strength.v
  - ${pjt_path}/tbench/mipi_tx/psr_generator.sv

### `coverage/coverage.f`
- active entry 수: 6
  - ${pjt_path}/tbench/coverage/coverage_macro.sv
  - ${pjt_path}/tbench/coverage/sys_fsm_coverage.sv
  - ${pjt_path}/tbench/coverage/reg_default_cov/t_reg_default_cov.sv
  - ${pjt_path}/tbench/coverage/vgen_coverage.sv
  - ${pjt_path}/tbench/coverage/pixel_align_coverage.sv
  - ${pjt_path}/tbench/coverage/gpo_coverage.sv

## 5. 주요 하위 폴더별 역할

### `checker/`
- 역할: TB/chip/video/flash/GPO/SRC/VGEN/VIN/PMIC/Power/DCC/SDC 등 checker와 scoreboard 자산. 기본 checker.f에는 일부 subset만 active.
- 대표 파일:
  - checker/autogen/cdc_reconv_checker.sv
  - checker/checker.f
  - checker/chip/pad_checker.sv
  - checker/dcc/dcc_checker.sv
  - checker/dcdc/bst_con_checker.sv
  - checker/dcdc/dcdc_checker.F
  - checker/dcdc/dcdc_checker.sv
  - checker/err_flag/errfg_checker.F
  - checker/err_flag/errfg_checker.sv
  - checker/flash/flash_pad_checker.sv
  - checker/flash/flash_scoreboard.sv
  - checker/gpo/checker_list.F
  - checker/gpo/d001a_gpo_update_checker.sv
  - checker/gpo/sva_delay_repeat_pkg.sv
  - checker/gpo/we103a_gpo00_01_checker.sv
  - checker/gpo/we103a_gpo04_checker.sv
  - checker/gpo/we103a_gpo05_checker.sv
  - checker/gpo/we103a_gpo20_25_checker.sv
  - checker/gpo/we103a_gpo20_49_dc_opt_checker.sv
  - checker/gpo/we103a_gpo42_47_checker.sv
  - checker/gpo/we103a_gpo48_49_checker.sv
  - checker/gpo/we103a_gpo50_checker.sv
  - checker/gpo/we103a_gpo_ab_opt_checker.sv
  - checker/gpo/we103a_gpo_dc_pol_checker.sv
  - checker/gpo/we103a_gpo_dotc_checker.sv

### `coverage/`
- 역할: coverage macro와 block별 coverage model. 현재 tbench.f에서는 coverage.f가 주석 처리되어 기본 compile에는 빠져 있다.
- 대표 파일:
  - coverage/coverage.f
  - coverage/coverage_macro.sv
  - coverage/gpo_coverage.sv
  - coverage/mbist_coverage.sv
  - coverage/otp_coverage.sv
  - coverage/pixel_align_coverage.sv
  - coverage/reg_default_cov/t_reg_default_cov.sv
  - coverage/sdc_core_coverage.sv
  - coverage/sram_coverage.sv
  - coverage/sys_fsm_coverage.sv
  - coverage/vgen_coverage.sv

### `define/`
- 역할: instance path, max parameter, index name, common utility define.
- 대표 파일:
  - define/common_util_define.vh
  - define/index_name_define.vh
  - define/inst_path_define.vh
  - define/max_param_define.vh

### `eds_scen/`
- 역할: EDS 및 test-vector scenario와 reg_set task.
- 대표 파일:
  - eds_scen/eds/reg_set/task_aofc_setting.v
  - eds_scen/eds/reg_set/task_avc_setting.v
  - eds_scen/eds/reg_set/task_dis_system_setting.v
  - eds_scen/eds/reg_set/task_dsc_setting.v
  - eds_scen/eds/reg_set/task_eds_setting.v
  - eds_scen/eds/reg_set/task_gpo_setting.v
  - eds_scen/eds/reg_set/task_gram_bist_setting.v
  - eds_scen/eds/reg_set/task_hlpm_setting.v
  - eds_scen/eds/reg_set/task_ldo_settting.v
  - eds_scen/eds/reg_set/task_man_power_on.v
  - eds_scen/eds/reg_set/task_mipi_setting.v
  - eds_scen/eds/reg_set/task_pcd_ccd_setting.v
  - eds_scen/eds/reg_set/task_pmic_setting.v
  - eds_scen/eds/reg_set/task_power_mode_set.v
  - eds_scen/eds/reg_set/task_power_on_off_seq.v
  - eds_scen/eds/reg_set/task_psr_mode_setting.v
  - eds_scen/eds/reg_set/task_sdc_ip_setting.v
  - eds_scen/eds/reg_set/task_sdc_ip_setting_all.v
  - eds_scen/eds/reg_set/task_source_setting.v
  - eds_scen/eds/reg_set/task_sram_bist_setting.v
  - eds_scen/eds/reg_set/task_system_init.v
  - eds_scen/eds/reg_set/task_te_setting.v
  - eds_scen/eds/reg_set/task_tig_setting.v
  - eds_scen/eds/reg_set/task_tpg_setting.v
  - eds_scen/eds/reg_set/task_tsp_te_setting.v

### `firmware/`
- 역할: MCU/ibex simulation firmware collateral 및 build output.
- 대표 파일:
  - 없음

### `flash/`
- 역할: flash_model.v. 외부 vendor flash library와 함께 사용.
- 대표 파일:
  - flash/flash_model.v

### `function/`
- 역할: 공통 function/helper include.
- 대표 파일:
  - function/func_common.svh
  - function/func_power.svh

### `i2c_slave/`
- 역할: I2C slave behavioral model.
- 대표 파일:
  - i2c_slave/i2c_s.v
  - i2c_slave/i2c_slv.v

### `injector/`
- 역할: clock delay, sync jitter, MIPI HS miss 등 stress/injection model. clock_delay는 기본 enabled.
- 대표 파일:
  - injector/clock_delay_injector.sv
  - injector/mipi_hs_miss_injector.sv
  - injector/sync_jitter_injector.sv

### `mipi_tx/`
- 역할: MIPI DSI host model, PHY/driver/FSM/PSR generator.
- 대표 파일:
  - mipi_tx/mipi_hostmodel.sv
  - mipi_tx/mipi_pkg.sv
  - mipi_tx/mipi_strength.v
  - mipi_tx/mipi_tx.f
  - mipi_tx/mipi_tx_driver.sv
  - mipi_tx/mipi_tx_fsm.sv
  - mipi_tx/mipi_tx_lnsw.v
  - mipi_tx/mipi_tx_phy_afe.sv
  - mipi_tx/psr_generator.sv
  - mipi_tx/tranif_rnm.sv

### `misc/`
- 역할: image force/dump, RGB overrider, random/test helper.
- 대표 파일:
  - misc/aofc_random.sv
  - misc/c_command.sv
  - misc/c_video.sv
  - misc/gpo_rand_class.sv
  - misc/gpo_test_pkg.sv
  - misc/image_dump.v
  - misc/image_force.v
  - misc/image_forcer.sv
  - misc/sdc_ip_rgb_overrider.sv

### `monitor/`
- 역할: ARM AXI protocol checker/monitor collateral. 현재 tbench.f에서는 주석 처리.
- 대표 파일:
  - monitor/arm_axi_pc/arm_axi_pc.F
  - monitor/arm_axi_pc/arm_axi_pc_bind.sv
  - monitor/arm_axi_pc/arm_axi_pc_ctrl.sv

### `package/`
- 역할: d2i/cmd_map/reg_access/img/dsc/test/svlib/uvm_report package.
- 대표 파일:
  - package/cmd_map_pkg.sv
  - package/d2i_pkg.sv
  - package/deposit_pkg.sv
  - package/dsc/dsc_config.sv
  - package/dsc/dsc_encode.sv
  - package/dsc/dsc_pkg.sv
  - package/img/Image.sv
  - package/img/img_pkg.sv
  - package/package.f
  - package/reg_access/reg_access_pkg.sv
  - package/reg_access/reg_item_cl.sv
  - package/reg_access/reg_list_cl.sv
  - package/svlib/src/svlib.f
  - package/svlib/src/svlib_dpi_imports.svh
  - package/svlib/src/svlib_impl_Cfg.svh
  - package/svlib/src/svlib_impl_Enum.svh
  - package/svlib/src/svlib_impl_File.svh
  - package/svlib/src/svlib_impl_Regex.svh
  - package/svlib/src/svlib_impl_Str.svh
  - package/svlib/src/svlib_impl_cfgNode_classes.svh
  - package/svlib/src/svlib_impl_cfgScalar_classes.svh
  - package/svlib/src/svlib_impl_svlibCfgBase.svh
  - package/svlib/src/svlib_macros.svh
  - package/svlib/src/svlib_pkg.sv
  - package/svlib/src/svlib_pkg_Cfg.svh

### `pmic/`
- 역할: PMIC 및 I2C 관련 external behavioral model.
- 대표 파일:
  - pmic/I2C.v
  - pmic/PMIC.v
  - pmic/sm_I2C_dummy.v

### `random_model/`
- 역할: panel/vgen/gpo/host random model package.
- 대표 파일:
  - random_model/base/base_model_pkg.sv
  - random_model/gpo/gpo_model.sv
  - random_model/gpo/gpo_model_pkg.sv
  - random_model/host/host_model_pkg.sv
  - random_model/host/host_video.svh
  - random_model/panel/panel_model.svh
  - random_model/panel/panel_model_pkg.sv
  - random_model/random_model.f
  - random_model/vgen/vgen_linefg.svh
  - random_model/vgen/vgen_model_pkg.sv
  - random_model/vgen/vgen_vrr.svh

### `scenario/`
- 역할: compile define별 scenario orchestration. 상세는 별도 scenario detail 문서 참조.
- 대표 파일:
  - scenario/aofc/aofc_helper.svh
  - scenario/aofc/aofc_random.svh
  - scenario/aofc/aofc_scenario.svh
  - scenario/dbi/dbi_scenario.svh
  - scenario/dcc/dcc_helper.svh
  - scenario/dcc/dcc_scenario.svh
  - scenario/err_fg/err_fg_helper.svh
  - scenario/err_fg/err_fg_scenario.svh
  - scenario/flash/flash_helper.svh
  - scenario/flash/flash_scenario.svh
  - scenario/gpo/gpo_helper.svh
  - scenario/gpo/gpo_scenario.svh
  - scenario/mipi/mipi_helper.svh
  - scenario/mipi/mipi_random.svh
  - scenario/mipi/mipi_scenario.svh
  - scenario/otp/otp_helper.svh
  - scenario/otp/otp_scenario.svh
  - scenario/pmic/pmic_helper.svh
  - scenario/pmic/pmic_scenario.svh
  - scenario/power/power_helper.svh
  - scenario/power/power_random.svh
  - scenario/power/power_scenario.svh
  - scenario/rt/rt_helper.svh
  - scenario/rt/rt_scenario.svh
  - scenario/scenario_main.v

### `task/`
- 역할: host에서 include되는 실제 stimulus task library.
- 대표 파일:
  - task/task_asbc.v
  - task/task_bist.v
  - task/task_chop.v
  - task/task_core.v
  - task/task_dbi.v
  - task/task_dma.v
  - task/task_flash.sv
  - task/task_force.v
  - task/task_gamma.v
  - task/task_gpo.sv
  - task/task_gpo.v
  - task/task_gpo_reg.sv
  - task/task_man_pwr.v
  - task/task_mipi.v
  - task/task_otp.v
  - task/task_otp_map_inc.v
  - task/task_password_define.v
  - task/task_pmic.sv
  - task/task_rd_define_inc.v
  - task/task_reg_auto_rw.v
  - task/task_reg_command.v
  - task/task_reg_command_rd_val.sv
  - task/task_reg_sram.sv
  - task/task_register_define_inc.v
  - task/task_sdc_ip.sv

## 6. 확인된 주의사항

- `tbench_structure.txt` 문서에는 `SCENARIO_ERR_FG`, `SCENARIO_PMIC_NEW` 표기가 있으나 실제 `scenario/scenario_main.v`에서는 `SCENARIO_ERRFG`, `SCENARIO_PMIC`가 사용된다. run script/compile.py define mapping 확인이 필요하다.
- `coverage/coverage.f`는 존재하지만 기본 `tbench.f`에서는 주석 처리되어 있다. coverage run 시 별도 옵션 또는 filelist 추가가 필요하다.
- checker 자산은 많지만 기본 active subset은 `checker/checker.f` 기준이다. PMIC/power/SDC/TE/TSP_TE/VIN 등은 별도 filelist나 define 활성화 경로 확인이 필요하다.
- `scenario/.scenario_main.v.swp` swap 파일이 존재한다. 필요 없으면 정리 후보이다.
- `firmware/` 하위에 object/dependency 등 build output이 포함되어 있어 source 관리 대상 여부 확인이 필요하다.
- `tbench_pre.f`, `tbench_post.f`, `tbench.f` 간 active 구성 차이가 있으므로 pre/post/rtl sim flow별 실제 사용 filelist 확인이 필요하다.
