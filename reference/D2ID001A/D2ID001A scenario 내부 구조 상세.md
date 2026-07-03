---
date: 2026-06-10
description: "D2ID001A tbench scenario 폴더의 scenario_main branch, 하위 폴더별 파일, task, include, 조건 define 상세 분석."
tags:
  - reference
  - rtl
  - verification
  - scenario
  - d2id001a
status: active
quarter: Q2-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/TB/tbench/scenario/SCENARIO_INTERNAL_STRUCTURE_DETAIL.md
---

# D2ID001A scenario 내부 구조 상세

상위 index: [[D2ID001A Reference]]  
관련 전체 구조: [[D2ID001A tbench 구조 리뷰]]

- 생성 위치: `/home/tim/pjt_work/D2ID001A/TB/tbench/scenario`
- 생성 시각: 2026-06-09 20:49:40
- 목적: `scenario/scenario_main.v`의 compile-time branch와 scenario 하위 폴더별 파일/역할/task 구성을 상세 정리

## 1. scenario 전체 실행 흐름

```text
host.sv
  `include "tbench/scenario/scenario_main.v"
        |
        +-- task t_main()
              |
              +-- TEST_VECTOR_ON mode
              |     +-- t_tv_initial()
              |     +-- t_scenario_main()
              |     +-- t_tv_end
              |
              +-- Normal mode
                    +-- REG_EXCLK = 1
                    +-- t_otp_initial()
                    +-- t_main_wakeup()
                    +-- t_scenario_main()
                    +-- t_idle_wait(1000)
```

`t_scenario_main()`은 compile define에 의해 하나만 선택되는 구조이다.

## 2. scenario_main.v branch table

| Define | 호출 task | include 파일 |
|---|---|---|
| `SCENARIO_AOFC` | `t_scenario_aofc()` | `tbench/scenario/aofc/aofc_random.svh`<br>`tbench/scenario/aofc/aofc_helper.svh`<br>`tbench/scenario/aofc/aofc_scenario.svh` |
| `SCENARIO_MIPI` | `t_scenario_mipi()` | `tbench/scenario/mipi/mipi_random.svh`<br>`tbench/scenario/mipi/mipi_helper.svh`<br>`tbench/scenario/mipi/mipi_scenario.svh` |
| `SCENARIO_TE` | `t_scenario_te()` | `tbench/scenario/te/te_random.svh`<br>`tbench/scenario/te/te_helper.svh`<br>`tbench/scenario/te/te_scenario.svh` |
| `SCENARIO_TOP` | `t_scenario_top()` | `tbench/scenario/top/top_random.svh`<br>`tbench/scenario/top/top_helper.svh`<br>`tbench/scenario/top/top_scenario.svh` |
| `SCENARIO_SRC` | `t_scenario_src()` | `tbench/scenario/top/top_random.svh`<br>`tbench/scenario/src/src_helper.svh`<br>`tbench/scenario/src/src_scenario.svh` |
| `SCENARIO_POWER` | `t_scenario_power()` | `tbench/scenario/top/top_random.svh`<br>`tbench/scenario/power/power_helper.svh`<br>`tbench/scenario/power/power_scenario.svh` |
| `SCENARIO_DBI` | `t_scenario_dbi()` | `tbench/scenario/dbi/dbi_scenario.svh` |
| `SCENARIO_UCS` | `t_scenario_ucs()` | `tbench/scenario/ucs/ucs_helper.svh`<br>`tbench/scenario/ucs/ucs_scenario.svh` |
| `SCENARIO_RT` | `t_scenario_rt()` | `tbench/scenario/rt/rt_helper.svh`<br>`tbench/scenario/rt/rt_scenario.svh` |
| `SCENARIO_DCC` | `t_scenario_dcc()` | `tbench/scenario/dcc/dcc_helper.svh`<br>`tbench/scenario/dcc/dcc_scenario.svh` |
| `SCENARIO_GPO` | `t_scenario_gpo()` | `tbench/scenario/top/top_random.svh`<br>`tbench/scenario/gpo/gpo_helper.svh`<br>`tbench/scenario/gpo/gpo_scenario.svh` |
| `SCENARIO_FLASH` | `t_scenario_flash()` | `tbench/scenario/flash/flash_helper.svh`<br>`tbench/scenario/flash/flash_scenario.svh` |
| `SCENARIO_OTP` | `t_scenario_otp()` | `tbench/scenario/otp/otp_helper.svh`<br>`tbench/scenario/otp/otp_scenario.svh` |
| `SCENARIO_SRAM` | `t_scenario_sram()` | `tbench/scenario/sram/sram_helper.svh`<br>`tbench/scenario/sram/sram_scenario.svh` |
| `SCENARIO_MX_IP` | `t_scenario_mx_ip()` | `tbench/scenario/mx_ip/mx_ip_helper.svh`<br>`tbench/scenario/mx_ip/mx_ip_scenario.svh` |
| `SCENARIO_IP` | `t_scenario_sdc_ip()` | `source/rtl/top/mmap.v`<br>`tbench/scenario/sdc_ip/sdc_ip_random.svh`<br>`tbench/scenario/sdc_ip/sdc_ip_helper.svh`<br>`tbench/scenario/sdc_ip/sdc_ip_scenario.svh` |
| `SCENARIO_PMIC` | `t_scenario_pmic()` | `tbench/scenario/pmic/pmic_helper.svh`<br>`tbench/scenario/pmic/pmic_scenario.svh` |
| `SCENARIO_ERRFG` | `t_scenario_err_fg()` | `tbench/scenario/err_fg/err_fg_helper.svh`<br>`tbench/scenario/err_fg/err_fg_scenario.svh` |
| `SCENARIO_VIN` | `t_scenario_vin()` | `tbench/scenario/vin/vin_helper.svh`<br>`tbench/scenario/vin/vin_scenario.svh` |
| `SCENARIO_VGEN` | `t_scenario_vgen()` | `tbench/scenario/vgen/panel_adaptor.svh`<br>`tbench/scenario/vgen/host_adaptor.svh`<br>`tbench/scenario/vgen/vgen_adaptor.svh`<br>`tbench/scenario/vgen/gpo_adaptor.svh`<br>`tbench/scenario/vgen/vgen_helper.svh`<br>`tbench/scenario/vgen/vgen_scenario.svh` |
| `SCENARIO_EDS_SAMPLE` | `t_scenario_eds_sample()` | `tbench/eds_scen/scenario_eds_sample.v` |
| `SCENARIO_EDS_MIPI` | `t_scenario_eds_mipi()` | `tbench/eds_scen/scenario_eds_mipi.v` |
| `SCENARIO_EDS_OSC` | `t_scenario_eds_osc()` | `tbench/eds_scen/scenario_eds_osc.v` |
| `SCENARIO_EDS_PMIC` | `t_scenario_eds_pmic()` | `tbench/eds_scen/scenario_eds_pmic.v` |
| `SCENARIO_EDS_PCD_CCD` | `t_scenario_eds_pcd_ccd()` | `tbench/eds_scen/scenario_eds_pcd_ccd.v` |
| `SCENARIO_EDS_RT` | `t_scenario_eds_rt()` | `tbench/eds_scen/scenario_eds_rt.v` |
| `SCENARIO_EDS_SRAM` | `t_scenario_eds_sram()` | `tbench/eds_scen/scenario_eds_sram.v` |
| `SCENARIO_EDS_GPO` | `t_scenario_eds_gpo()` | `tbench/eds_scen/scenario_eds_gpo.v` |
| `SCENARIO_EDS_FLASH` | `t_scenario_eds_flash()` | `tbench/eds_scen/scenario_eds_flash.v` |
| `SCENARIO_EDS_OTP` | `t_scenario_eds_otp()` | `tbench/eds_scen/scenario_eds_otp.v` |
| `SCENARIO_EDS_SRC` | `t_scenario_eds_src()` | `tbench/eds_scen/scenario_eds_src.v` |
| `SCENARIO_EDS_SDCIP` | `t_scenario_eds_sdcip()` | `tbench/eds_scen/scenario_eds_sdcip.v` |
| `SCENARIO_EDS_PAD` | `t_scenario_eds_pad()` | `tbench/eds_scen/scenario_eds_pad.v` |
| `SCENARIO_EDS_VOLDET` | `t_scenario_eds_voldet()` | `tbench/eds_scen/scenario_eds_voldet.v` |
| `SCENARIO_EDS_RESET` | `t_scenario_eds_reset()` | `tbench/eds_scen/scenario_eds_reset.v` |
| `SCENARIO_EDS_HVS_HTOL` | `t_scenario_eds_hvs_htol()` | `tbench/eds_scen/scenario_eds_hvs_htol.v` |

## 3. 하위 scenario 폴더별 상세

### `aofc/`
- 추정 역할: AOFC 관련 random/helper/scenario. osc trim, AOFC fail/filter/mask, MIPI clock on/off delay 등 포함.
- 파일 수: 4
- 파일 구성:
  - `README.md` (93 lines)
    - 문서 앞부분: # AOFC Scenario README / ## Purpose / AOFC scenario의 testcase 구성과 목적을 정리한 문서.
  - `aofc_helper.svh` (171 lines)
    - task: t_aofc_config_check(), t_aofc_default(), t_aofc_slp_disp_off(), t_aofc_slp_disp_on(), t_mipi_clk_onoff_dly(), t_mipi_clk_onoff_rand_frame()
    - 조건 define: __AOFC_HELPER_SVH__
  - `aofc_random.svh` (217 lines)
    - class: c_aofc_random
  - `aofc_scenario.svh` (1061 lines)
    - task: t_scenario_aofc(), t_scenario_aofc_setup(), t_scenario_osc_trim(), t_scenario_aofc_test(), t_scenario_aofc_test_fail(), t_scenario_aofc_fail_filter(), t_scenario_aofc_fail_mask(), t_scenario_aofc_test_manual(), t_scenario_aofc_test_random(), t_scenario_aofc_sscg()
    - 조건 define: __AOFC_SCENARIO_SVH__, __SIM_RTL__
- scenario entry/task 후보:
  - t_scenario_aofc()
  - t_scenario_aofc_setup()
  - t_scenario_osc_trim()
  - t_scenario_aofc_test()
  - t_scenario_aofc_test_fail()
  - t_scenario_aofc_fail_filter()
  - t_scenario_aofc_fail_mask()
  - t_scenario_aofc_test_manual()
  - t_scenario_aofc_test_random()
  - t_scenario_aofc_sscg()

### `dbi/`
- 추정 역할: DBI access scenario.
- 파일 수: 1
- 파일 구성:
  - `dbi_scenario.svh` (178 lines)
    - task: t_scenario_dbi(), t_scenario_dbi_setup(), t_scenario_dbi_access()
    - 조건 define: AXI_VIP_MONITOR
- scenario entry/task 후보:
  - t_scenario_dbi()
  - t_scenario_dbi_setup()
  - t_scenario_dbi_access()

### `dcc/`
- 추정 역할: DCC/CCD 관련 initial/helper/scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (35 lines)
    - 문서 앞부분: # Scenario: dcc / ## Entry / - t_scenario_dcc
  - `dcc_helper.svh` (130 lines)
    - task: t_dcc_initial()
  - `dcc_scenario.svh` (1138 lines)
    - task: t_scenario_dcc(), t_ccd_pass(), t_ccd_en_pol(), t_ccd_mon(), t_ccd_power_on(), t_ccd_start_width(), t_ccd_detect_duration(), t_ccd_min_width(), t_ccd_margin_long(), t_ccd_margin_short(), t_ccd_nothing(), t_ccd_non_fall(), t_ccd_soft_reset(), t_ccd_soft_reset_direct()
- scenario entry/task 후보:
  - t_scenario_dcc()

### `err_fg/`
- 추정 역할: error flag, checksum, fail-safe, bus fail detection 관련 scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (158 lines)
    - 문서 앞부분: # ERR_FG / CHECKSUM Scenario / ## 개요 / 본 scenario는 panel 동작 중 발생 가능한 error flag (ERR_FG) 및 checksum 관련 동작을 검증하기 위한 테스트 케이스 모음이다.
  - `err_fg_helper.svh` (217 lines)
    - task: t_scenario_errfg_init(), reg_command_init(), v_detec_en(), show_result()
    - 조건 define: __ERR_FG_HELPER_SVH__
  - `err_fg_scenario.svh` (1852 lines)
    - task: t_scenario_err_fg(), t_scen_data_chksum(), t_scenario_overall_errfg(), t_scenario_dis_sta(), t_scen_fail_safe(), t_scen_bus_fail_det(), t_scen_chksum_ucs(), t_scen_chksum_clk(), t_scenario_overall_esd_fg(), t_scenario_overall_v_detect_rstb(), t_scenario_esdfg(), t_scenario_analog_vcd()
    - 조건 define: __SIM_POST__
- scenario entry/task 후보:
  - t_scenario_err_fg()
  - t_scenario_overall_errfg()
  - t_scenario_dis_sta()
  - t_scenario_overall_esd_fg()
  - t_scenario_overall_v_detect_rstb()
  - t_scenario_esdfg()
  - t_scenario_analog_vcd()

### `flash/`
- 추정 역할: flash SPI mode/register/write/read/check flow.
- 파일 수: 3
- 파일 구성:
  - `README.md` (42 lines)
    - 문서 앞부분: # Scenario: flash / ## Entry / - t_scenario_flash
  - `flash_helper.svh` (323 lines)
    - task: t_scenario_flash_initial(), t_scenario_flash_spi_mode_setting(), t_scenario_flash_reg_setting(), t_scenario_flash_wdata(), t_scenario_flash_wstrb(), t_scenario_flash_result()
    - 조건 define: AXI_VIP_MONITOR
  - `flash_scenario.svh` (1759 lines)
    - task: t_scenario_flash(), t_scenario_flash_case01(), t_scenario_flash_case02(), t_scenario_flash_case03(), t_scenario_flash_case04(), t_scenario_flash_case05(), t_scenario_flash_case06(), t_scenario_flash_case07(), t_scenario_flash_case08(), t_scenario_flash_case09(), t_scenario_flash_case10(), t_scenario_flash_case11(), t_scenario_flash_case12(), t_scenario_flash_case13(), t_scenario_flash_case14(), t_scenario_flash_case15(), t_scenario_flash_case16(), t_scenario_flash_case17(), t_scenario_flash_case18()
    - 조건 define: __SIM_POST__, FLASH_SCOREBOARD
- scenario entry/task 후보:
  - t_scenario_flash()
  - t_scenario_flash_case01()
  - t_scenario_flash_case02()
  - t_scenario_flash_case03()
  - t_scenario_flash_case04()
  - t_scenario_flash_case05()
  - t_scenario_flash_case06()
  - t_scenario_flash_case07()
  - t_scenario_flash_case08()
  - t_scenario_flash_case09()
  - t_scenario_flash_case10()
  - t_scenario_flash_case11()
  - t_scenario_flash_case12()
  - t_scenario_flash_case13()
  - t_scenario_flash_case14()
  - t_scenario_flash_case15()
  - t_scenario_flash_case16()
  - t_scenario_flash_case17()
  - t_scenario_flash_case18()

### `gpo/`
- 추정 역할: GPO sync/select/polarity/power sequence/dot clock 등 광범위한 GPO scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (53 lines)
    - 문서 앞부분: # Scenario: gpo / ## Entry / - t_scenario_gpo
  - `gpo_helper.svh` (119 lines)
    - task: t_vdo_rand(), t_vdo_frame(), t_vdo_vrr_random_frame()
  - `gpo_scenario.svh` (4644 lines)
    - task: t_scenario_gpo(), t_scenario_sub_gpo(), t_scenario_sub_gpo0_1_sync_sel(), t_scenario_sub_gpo__4_sync_sel(), t_scenario_sub_gpo__5_sync_sel(), t_scenario_sub_gpo__5_abnormal_1(), t_scenario_sub_gpo__5_abnormal_2(), t_scenario_sub_gpo20_25_overflow(), t_scenario_sub_gpo20_25_piece(), t_scenario_sub_gpo20_25_piece_abnormal(), t_scenario_sub_gpo42_50_overflow(), t_scenario_sub_gpo_auto_pwr(), t_scenario_sub_gpo_man_pwr(), t_scenario_sub_gpo_sync_dly(), t_scenario_sub_gpo_h_del(), t_scenario_sub_gpo_porch_dly_dc_opt(), t_scenario_sub_gpo_ab_opt(), t_scenario_sub_gpo_eq(), t_scenario_sub_gpo_select(), t_scenario_sub_gpo_full_resol(), t_scenario_sub_gpo_eq_slew_con(), t_scenario_sub_gpo_neq_slew_con1(), t_scenario_sub_gpo_neq_slew_con2(), t_scenario_sub_gpo_norm(), t_scenario_sub_gpo_0_1_rand() ...
- scenario entry/task 후보:
  - t_scenario_gpo()
  - t_scenario_sub_gpo()
  - t_scenario_sub_gpo0_1_sync_sel()
  - t_scenario_sub_gpo__4_sync_sel()
  - t_scenario_sub_gpo__5_sync_sel()
  - t_scenario_sub_gpo__5_abnormal_1()
  - t_scenario_sub_gpo__5_abnormal_2()
  - t_scenario_sub_gpo20_25_overflow()
  - t_scenario_sub_gpo20_25_piece()
  - t_scenario_sub_gpo20_25_piece_abnormal()
  - t_scenario_sub_gpo42_50_overflow()
  - t_scenario_sub_gpo_auto_pwr()
  - t_scenario_sub_gpo_man_pwr()
  - t_scenario_sub_gpo_sync_dly()
  - t_scenario_sub_gpo_h_del()
  - t_scenario_sub_gpo_porch_dly_dc_opt()
  - t_scenario_sub_gpo_ab_opt()
  - t_scenario_sub_gpo_eq()
  - t_scenario_sub_gpo_select()
  - t_scenario_sub_gpo_full_resol()
  - t_scenario_sub_gpo_eq_slew_con()
  - t_scenario_sub_gpo_neq_slew_con1()
  - t_scenario_sub_gpo_neq_slew_con2()
  - t_scenario_sub_gpo_norm()
  - t_scenario_sub_gpo_0_1_rand()
  - t_scenario_sub_gpo_4_5_rand()
  - t_scenario_sub_gpo_20_25_rand()
  - t_scenario_sub_gpo_42_50_rand()
  - t_scenario_sub_gpo_pwr_rand()
  - t_scenario_sub_gpo_nov_rand()
  - ... (5개 생략)

### `mipi/`
- 추정 역할: MIPI command/random/PRBS/ULPS/reset/read-write DDB 등 DSI host stimulus scenario.
- 파일 수: 5
- 파일 구성:
  - `README.md` (176 lines)
    - 문서 앞부분: # MIPI Scenario README (기능 기준 정리) / ## 1. 목적 / 본 문서는 MIPI scenario의 testcase를 **기능(Function) 기준으로 분류**하고,
  - `mipi_helper.svh` (591 lines)
    - task: randomize_global_timing(), global_para(), final_check_mipi(), D8_DEC(), D8_INC(), D8_RAND(), t_set_d8_cmd(), t_reset_d8_cmd(), PACKET_D8_CMD()
  - `mipi_random.svh` (104 lines)
    - class: c_global_dphy
    - 조건 define: EDS_CLK
  - `mipi_scenario.svh` (8400 lines)
    - task: t_scenario_mipi(), t_scenario_mipi_setup(), t_scenario_mipi_cmd_random(), t_scenario_mipi_prbs_test(), t_scenario_mipi_ulps(), t_scenario_mipi_trigger_reset(), t_scenario_mipi_rw_ddb(), t_scenario_mipi_random_single_gpara(), t_scenario_mipi_mcmd(), t_scenario_mipi_mcmd_01(), t_scenario_mipi_mcmd_10(), t_scenario_mipi_mcmd_sramtest(), t_scenario_mipi_mcmd_time_cnt(), t_scenario_mipi_pps_cmd_read(), t_scenario_mipi_pps_packet_read(), t_scenario_mipi_vdo_bmp(), t_scenario_mipi_lpnsw(), t_scenario_mipi_vdo_dsc(), t_scenario_mipi_sram_write(), t_scenario_mipi_sram_short(), t_scenario_mipi_clock_esd(), t_scenario_mipi_lptx(), t_scenario_mipi_lpcd(), t_scenario_mipi_lprx(), t_scenario_mipi_sot_error() ...
    - include: tbench/scenario/sdc_ip_reg_setting/register_022.vh, tbench/scenario/sdc_ip_reg_setting/flash_staddr.vh, tbench/task/task_wr_define_inc.v
    - 조건 define: __SIM_POST__
  - `plan.txt` (148 lines)
- scenario entry/task 후보:
  - t_scenario_mipi()
  - t_scenario_mipi_setup()
  - t_scenario_mipi_cmd_random()
  - t_scenario_mipi_prbs_test()
  - t_scenario_mipi_ulps()
  - t_scenario_mipi_trigger_reset()
  - t_scenario_mipi_rw_ddb()
  - t_scenario_mipi_random_single_gpara()
  - t_scenario_mipi_mcmd()
  - t_scenario_mipi_mcmd_01()
  - t_scenario_mipi_mcmd_10()
  - t_scenario_mipi_mcmd_sramtest()
  - t_scenario_mipi_mcmd_time_cnt()
  - t_scenario_mipi_pps_cmd_read()
  - t_scenario_mipi_pps_packet_read()
  - t_scenario_mipi_vdo_bmp()
  - t_scenario_mipi_lpnsw()
  - t_scenario_mipi_vdo_dsc()
  - t_scenario_mipi_sram_write()
  - t_scenario_mipi_sram_short()
  - t_scenario_mipi_clock_esd()
  - t_scenario_mipi_lptx()
  - t_scenario_mipi_lpcd()
  - t_scenario_mipi_lprx()
  - t_scenario_mipi_sot_error()
  - t_scenario_mipi_eot_error()
  - t_scenario_mipi_esc_error()
  - t_scenario_mipi_lpsync_error()
  - t_scenario_mipi_timer_error()
  - t_scenario_mipi_false_error()
  - ... (18개 생략)

### `otp/`
- 추정 역할: OTP programming/read/check/register RW/default/trim 관련 scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (46 lines)
    - 문서 앞부분: # Scenario: otp / ## Entry / - t_scenario_otp
  - `otp_helper.svh` (136 lines)
    - task: t_otp_max_fail(), t_otp_trim_check(), t_otp_main_check()
  - `otp_scenario.svh` (2263 lines)
    - task: t_scenario_otp(), t_otp_registerrw(), t_otp_dbif_read(), t_otp_pad_skip(), t_otp_por_loadtest(), t_otp_direct_rd(), t_otp_trim_prog(), t_otp_main_prog(), t_otp_trim__eco(), t_otp_trim__man(), t_otp_init_read(), t_otp_pg2ld4comp(), t_otp_src_load(), t_otp_norm_ld4comp(), t_otp_full_prog(), t_otp_trim_rand(), t_otp_main_rand(), t_otp_reg_test(), t_otp_inter_vpp_man(), t_otp_inter_vpp(), t_otp_self_chk(), t_otp_flash_firm(), t_otp_mcu_chksm(), t_otp_load_test(), t_otp_dma__load() ...
    - 조건 define: __SIM_RTL__
- scenario entry/task 후보:
  - t_scenario_otp()

### `pmic/`
- 추정 역할: PMIC GPIO/ASWIRE/ESWIRE/I2C/power control 관련 scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (63 lines)
    - 문서 앞부분: # Scenario: pmic / ## Entry / - t_scenario_pmic
  - `pmic_helper.svh` (176 lines)
    - task: wait_hs_trans_fall(), send_frame_vdo_for_pmic()
  - `pmic_scenario.svh` (3036 lines)
    - task: t_scenario_pmic(), t_scenario_pmic_gpio_sel(), t_scenario_pmic_aswire_on_off(), t_scenario_pmic_atset_line_en(), t_scenario_pmic_aswire_change(), t_scenario_pmic_aswire_cov(), t_scenario_pmic_aswire_slpout(), t_scenario_pmic_aswire_ini_seq(), t_scenario_pmic_aswire_vlout3(), t_scenario_pmic_eswire_on_off(), t_scenario_pmic_etset_line_en(), t_scenario_pmic_etset_line_all_case(), t_scenario_pmic_eswire_initial_over0(), t_scenario_pmic_eswire_initial_over1(), t_scenario_pmic_eswire_initial_over2(), t_scenario_pmic_eswire_dyn_in_2nd_frame(), t_scenario_pmic_eswire_man_el_off(), t_scenario_pmic_eswire_on_off_bug(), t_scenario_pmic_eswire_refresh(), t_scenario_pmic_eswire_refresh_vrr(), t_scenario_pmic_eswire_refresh_wa(), t_scenario_pmic_eswire_refresh_eco(), t_scenario_pmic_swire_mst_man_off(), t_scenario_pmic_eswire_lhbm(), t_scenario_pmic_eswire_lhbm_ref() ...
    - 조건 define: __SIM_POST__
- scenario entry/task 후보:
  - t_scenario_pmic()
  - t_scenario_pmic_gpio_sel()
  - t_scenario_pmic_aswire_on_off()
  - t_scenario_pmic_atset_line_en()
  - t_scenario_pmic_aswire_change()
  - t_scenario_pmic_aswire_cov()
  - t_scenario_pmic_aswire_slpout()
  - t_scenario_pmic_aswire_ini_seq()
  - t_scenario_pmic_aswire_vlout3()
  - t_scenario_pmic_eswire_on_off()
  - t_scenario_pmic_etset_line_en()
  - t_scenario_pmic_etset_line_all_case()
  - t_scenario_pmic_eswire_initial_over0()
  - t_scenario_pmic_eswire_initial_over1()
  - t_scenario_pmic_eswire_initial_over2()
  - t_scenario_pmic_eswire_dyn_in_2nd_frame()
  - t_scenario_pmic_eswire_man_el_off()
  - t_scenario_pmic_eswire_on_off_bug()
  - t_scenario_pmic_eswire_refresh()
  - t_scenario_pmic_eswire_refresh_vrr()
  - t_scenario_pmic_eswire_refresh_wa()
  - t_scenario_pmic_eswire_refresh_eco()
  - t_scenario_pmic_swire_mst_man_off()
  - t_scenario_pmic_eswire_lhbm()
  - t_scenario_pmic_eswire_lhbm_ref()
  - t_scenario_pmic_i2c_start()
  - t_scenario_pmic_i2c_init_start()
  - t_scenario_pmic_i2c_dly_initial()
  - t_scenario_pmic_i2c_scl()
  - t_scenario_pmic_i2c_retry()
  - ... (16개 생략)

### `power/`
- 추정 역할: DCDC/HLPM/power mode/on-off/source/video frame 관련 power scenario.
- 파일 수: 4
- 파일 구성:
  - `README.md` (64 lines)
    - 문서 앞부분: # Scenario: power / ## Entry / - t_scenario_power
  - `power_helper.svh` (2598 lines)
    - task: t_power_init(), t_scenario_top_vdo_vrr_random_frame(), t_scenario_top_mode_chg_vdo_frame(), t_scenario_top_mode_chg_vdo_frame_rand(), t_scenario_top_vdo_mode_change(), t_scenario_top_vdo_frame(), t_power_ldo_trim_setting(), t_power_ldo_set_setting(), t_power_vint_pn_trans(), t_power_avc_ctl(), t_power_avc_manual_ctrl(), t_power_man_el_off(), t_power_dyn_vgh(), t_power_dyn_vgl(), t_power_dstb(), t_power_man_seq(), t_power_otp_man(), t_power_seq_setting(), t_power_offset_test(), t_power_offset_compare(), t_power_offset_read(), t_power_sdc_ip_offset_force(), t_power_rvdd(), t_power_ldo_dis(), t_power_ldo_on_off() ...
    - 조건 define: POST
  - `power_random.svh` (352 lines)
    - class: dcdc_transaction, hlpm_transaction
  - `power_scenario.svh` (2150 lines)
    - task: t_scenario_power(), t_scenario_power_once(), t_scenario_power_test(), t_scenario_power_avc_test(), t_scenario_power_total(), t_scenario_power_cosim(), t_scenario_scan(), t_scenario_debug()
    - include: /PROJECT/WE103A-0/Digital/WORKSPACE/wonwoo.lee/we103a-0/utility/cmd_to_reg/out.sv
    - 조건 define: POST, DEBUG_TMD
- scenario entry/task 후보:
  - t_scenario_power()
  - t_scenario_power_once()
  - t_scenario_power_test()
  - t_scenario_power_avc_test()
  - t_scenario_power_total()
  - t_scenario_power_cosim()
  - t_scenario_scan()
  - t_scenario_debug()

### `rt/`
- 추정 역할: register test, lock/default/write/void 등 RT register scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (44 lines)
    - 문서 앞부분: # Scenario: rt / ## Entry / - t_scenario_rt
  - `rt_helper.svh` (249 lines)
    - task: t_dcc_input_force(), t_dcc_io_check()
    - 조건 define: __SIM_RTL__
  - `rt_scenario.svh` (3018 lines)
    - task: t_scenario_rt(), t_scenario_reg_test(), t_scenario_reg_lock(), t_scenario_mcs_default(), t_scenario_rt2_default(), t_scenario_mcs_write(), t_scenario_mcs_void(), t_scenario_mcs_write_ff(), t_scenario_mcs_rs(), t_scenario_rt2_write(), t_scenario_mcs_reg(), t_scenario_ucs_sdr(), t_scenario_ucs_cmd(), t_scenario_ucs_signal_mode(), t_scenario_ucs_display_mode(), t_scenario_ucs_power_mode(), t_scenario_ucs_reg(), t_scenario_ucs_get_channel(), t_scenario_ucs_dbv(), t_scenario_ucs_pps(), t_scenario_ucs_ddb(), t_scenario_reg_bound(), t_scenario_reg_sram(), t_scenario_reg_multi_cmd(), t_scenario_reg_dcc_mon()
    - 조건 define: __SIM_RTL__, DV_DSC_COMPATIBLE_DBV
- scenario entry/task 후보:
  - t_scenario_rt()
  - t_scenario_reg_test()
  - t_scenario_reg_lock()
  - t_scenario_mcs_default()
  - t_scenario_rt2_default()
  - t_scenario_mcs_write()
  - t_scenario_mcs_void()
  - t_scenario_mcs_write_ff()
  - t_scenario_mcs_rs()
  - t_scenario_rt2_write()
  - t_scenario_mcs_reg()
  - t_scenario_ucs_sdr()
  - t_scenario_ucs_cmd()
  - t_scenario_ucs_signal_mode()
  - t_scenario_ucs_display_mode()
  - t_scenario_ucs_power_mode()
  - t_scenario_ucs_reg()
  - t_scenario_ucs_get_channel()
  - t_scenario_ucs_dbv()
  - t_scenario_ucs_pps()
  - t_scenario_ucs_ddb()
  - t_scenario_reg_bound()
  - t_scenario_reg_sram()
  - t_scenario_reg_multi_cmd()
  - t_scenario_reg_dcc_mon()

### `sdc_ip/`
- 추정 역할: SDC IP testcase/random/helper/scenario.
- 파일 수: 4
- 파일 구성:
  - `README.md` (76 lines)
    - 문서 앞부분: # Scenario: sdc_ip / ## Entry / - t_scenario_sdc_ip
  - `sdc_ip_helper.svh` (539 lines)
    - task: t_scenario_finit_skip()
  - `sdc_ip_random.svh` (129 lines)
    - class: randclass_hcg
  - `sdc_ip_scenario.svh` (5700 lines)
    - task: t_scenario_sdc_ip(), t_scenario_pwr_ip_all_en(), t_scenario_overall_3_34(), t_scenario_fpga_case60(), t_scenario_hcg_reg_ctrl(), t_scenario_hcg_sleep(), t_scenario_hcg_gen_rand(), t_scenario_hcg_gate_rand(), t_scenario_hcg_rand(), t_scenario_mipi_ip_access(), t_scenario_sdcip_bp(), t_scenario_eds_vector(), t_scenario_dsc_sel(), t_scenario_dsc_bp(), t_scenario_fq_sel(), t_scenario_finit_sleep(), t_scenario_finit_sram_read(), t_scenario_finit_sleep_fail(), t_scenario_finit_orbit(), t_scenario_finit(), t_scenario_finit_all_bp(), t_scenario_bypass_src(), t_scenario_ip_bp_test(), t_scenario_finit_order(), t_scenario_finit_err() ...
    - 조건 define: FPGA, AXI_VIP_MONITOR
- scenario entry/task 후보:
  - t_scenario_sdc_ip()
  - t_scenario_pwr_ip_all_en()
  - t_scenario_overall_3_34()
  - t_scenario_fpga_case60()
  - t_scenario_hcg_reg_ctrl()
  - t_scenario_hcg_sleep()
  - t_scenario_hcg_gen_rand()
  - t_scenario_hcg_gate_rand()
  - t_scenario_hcg_rand()
  - t_scenario_mipi_ip_access()
  - t_scenario_sdcip_bp()
  - t_scenario_eds_vector()
  - t_scenario_dsc_sel()
  - t_scenario_dsc_bp()
  - t_scenario_fq_sel()
  - t_scenario_finit_sleep()
  - t_scenario_finit_sram_read()
  - t_scenario_finit_sleep_fail()
  - t_scenario_finit_orbit()
  - t_scenario_finit()
  - t_scenario_finit_all_bp()
  - t_scenario_bypass_src()
  - t_scenario_ip_bp_test()
  - t_scenario_finit_order()
  - t_scenario_finit_err()
  - t_scenario_finit_errfg()
  - t_scenario_finit_errcnt()
  - t_scenario_ip15_ref_set_num()
  - t_scenario_ip15_ref_set_num_mipi()
  - t_scenario_ip15_com_set_num()
  - ... (19개 생략)

### `sram/`
- 추정 역할: SRAM scenario 및 SRAM helper.
- 파일 수: 3
- 파일 구성:
  - `README.md` (48 lines)
    - 문서 앞부분: # Scenario: sram / ## Entry / - t_scenario_sram
  - `sram_helper.svh` (29 lines)
  - `sram_scenario.svh` (4962 lines)
    - task: t_scenario_sram(), t_scenario_sramslv_ctlreg_access(), t_scenario_sramslv_gram_access(), t_scenario_sramslv_gram_selfchk(), t_scenario_sramslv6_gram_mbist(), t_scenario_sramslv8_gram_mbist(), t_scenario_sramslv4_ip11_mbist(), t_scenario_peri_rv_mcuic_mbist(), t_scenario_vout_src_sorc_mbist(), t_scenario_mipi_multicmd_mbist(), t_scenario_c6_gpo_sram_mbist(), t_scenario_vin_cbrb_lbob_mbist(), t_scenario_peri_flashslv_mbist(), t_scenario_vprsdc_ip05_mem_mbist(), t_scenario_vprsdc_ip06_mem_mbist(), t_scenario_vprsdc_ip09_mem_mbist(), t_scenario_vprsdc_ip11_mem_mbist(), t_scenario_vprsdc_ip12_mem_mbist(), t_scenario_vprsdc_ip15_mem_mbist(), t_scenario_vprsdc_ip16_mem_mbist(), t_scenario_vprsdc_ip23_mem_mbist(), t_scenario_vprsdc_ip31_mem_mbist(), t_scenario_vprsdc_regis_mbist(), t_scenario_loadotp_repaire_check(), t_scenario_vout_src_bus_dmaacc() ...
    - 조건 define: axi_vip_monitor, __sim_post__, __sim_pnet__, __sim_rtl__
- scenario entry/task 후보:
  - t_scenario_sram()
  - t_scenario_sramslv_ctlreg_access()
  - t_scenario_sramslv_gram_access()
  - t_scenario_sramslv_gram_selfchk()
  - t_scenario_sramslv6_gram_mbist()
  - t_scenario_sramslv8_gram_mbist()
  - t_scenario_sramslv4_ip11_mbist()
  - t_scenario_peri_rv_mcuic_mbist()
  - t_scenario_vout_src_sorc_mbist()
  - t_scenario_mipi_multicmd_mbist()
  - t_scenario_c6_gpo_sram_mbist()
  - t_scenario_vin_cbrb_lbob_mbist()
  - t_scenario_peri_flashslv_mbist()
  - t_scenario_vprsdc_ip05_mem_mbist()
  - t_scenario_vprsdc_ip06_mem_mbist()
  - t_scenario_vprsdc_ip09_mem_mbist()
  - t_scenario_vprsdc_ip11_mem_mbist()
  - t_scenario_vprsdc_ip12_mem_mbist()
  - t_scenario_vprsdc_ip15_mem_mbist()
  - t_scenario_vprsdc_ip16_mem_mbist()
  - t_scenario_vprsdc_ip23_mem_mbist()
  - t_scenario_vprsdc_ip31_mem_mbist()
  - t_scenario_vprsdc_regis_mbist()
  - t_scenario_loadotp_repaire_check()
  - t_scenario_vout_src_bus_dmaacc()
  - t_scenario_ip23_gram_access()
  - t_scenario_ip15_gram_access()
  - t_scenario_vout_src_sorc_acc()
  - t_scenario_sram_power_onoff()

### `src/`
- 추정 역할: source/video/source setting scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (74 lines)
    - 문서 앞부분: # Scenario: src / ## Entry / - t_scenario_src
  - `src_helper.svh` (509 lines)
    - task: t_scenario_top_vdo_frame(), t_scenario_src_init(), t_scenario_top_mode_chg_vdo_frame(), t_scenario_top_vdo_vrr_random_frame(), t_power_sleep_wait()
  - `src_scenario.svh` (8368 lines)
    - task: t_scenario_src(), t_scenario_local_hbm(), t_scenario_src_tpg(), t_scenario_src_tpg_image_dump(), t_scenario_src_asbc(), t_scenario_src_normal_ss(), t_scenario_src_data_por(), t_scenario_src_normal(), t_scenario_src_normal_ss_unused(), t_scenario_src_sorc(), t_scenario_src_sorc_gtcon(), t_scenario_src_sorc_opt1(), t_scenario_src_sorc_opt2(), t_scenario_src_sorc_opt3(), t_scenario_src_sorc_opt_s0w100(), t_scenario_src_sorc_opt_s0w99(), t_scenario_src_sorc_opt1_ss(), t_scenario_src_unused_sorc(), t_scenario_src_unused_ss_sorc(), t_scenario_src_unused_normal(), t_scenario_src_ss_sorc(), t_scenario_src_gamma(), t_scenario_src_timing(), t_scenario_src_amp(), t_scenario_src_srb() ...
- scenario entry/task 후보:
  - t_scenario_src()
  - t_scenario_local_hbm()
  - t_scenario_src_tpg()
  - t_scenario_src_tpg_image_dump()
  - t_scenario_src_asbc()
  - t_scenario_src_normal_ss()
  - t_scenario_src_data_por()
  - t_scenario_src_normal()
  - t_scenario_src_normal_ss_unused()
  - t_scenario_src_sorc()
  - t_scenario_src_sorc_gtcon()
  - t_scenario_src_sorc_opt1()
  - t_scenario_src_sorc_opt2()
  - t_scenario_src_sorc_opt3()
  - t_scenario_src_sorc_opt_s0w100()
  - t_scenario_src_sorc_opt_s0w99()
  - t_scenario_src_sorc_opt1_ss()
  - t_scenario_src_unused_sorc()
  - t_scenario_src_unused_ss_sorc()
  - t_scenario_src_unused_normal()
  - t_scenario_src_ss_sorc()
  - t_scenario_src_gamma()
  - t_scenario_src_timing()
  - t_scenario_src_amp()
  - t_scenario_src_srb()
  - t_scenario_src_dhyoo()
  - t_scenario_src_chop()
  - t_scenario_src_hbm()
  - t_scenario_src_offsel()
  - t_scenario_src_blkdly()
  - ... (22개 생략)

### `te/`
- 추정 역할: TE random/helper/scenario.
- 파일 수: 4
- 파일 구성:
  - `README.md` (143 lines)
    - 문서 앞부분: # TE Scenario README / ## 목적 / 본 문서는 TE scenario의 testcase를 간단히 정리한 문서입니다.
  - `te_helper.svh` (105 lines)
    - task: t_te_random_ctx_sync(), t_te_common_init(), t_te_write_vrr_bd(), t_te_write_vrr_ba_bd(), t_te_reg_force(), t_te_vs__force()
    - 조건 define: __SIM_RTL__
  - `te_random.svh` (296 lines)
    - class: c_te_random
  - `te_scenario.svh` (417 lines)
    - task: t_scenario_te(), t_scenario_te_post(), t_scenario_setup(), t_scenario_all_rand_case(), t_scenario_rand_case(), t_scenario_te_onoff(), t_scenario_te_gen(), t_scenario_te_vrr()
    - 조건 define: __SIM_RTL__, __SIM_POST__
- scenario entry/task 후보:
  - t_scenario_te()
  - t_scenario_te_post()
  - t_scenario_setup()
  - t_scenario_all_rand_case()
  - t_scenario_rand_case()
  - t_scenario_te_onoff()
  - t_scenario_te_gen()
  - t_scenario_te_vrr()

### `top/`
- 추정 역할: top smoke/mode/video VRR random scenario.
- 파일 수: 4
- 파일 구성:
  - `README.md` (468 lines)
    - 문서 앞부분: # SCENARIO_TOP README / ## Testcase / `SCENARIO_TOP`은 `+scen=<NAME>` plusarg로 testcase를 선택한다.
  - `top_helper.svh` (392 lines)
    - task: t_scenario_top_init(), t_scenario_top_vdo_vrr_random_frame(), t_scenario_top_vdo_frame(), t_scenario_top_mode_chg_vdo_frame(), t_scenario_top_vdo_vfpmiss(), t_scenario_top_vdo_rand(), t_scenario_top_vdo_mode_change(), t_mipi_mcu_step_on(), t_mipi_mcu_step_off(), t_mipi_mcu_step(), t_top_dbi_mbist()
  - `top_random.svh` (160 lines)
    - class: c_vdo_vrr_ramdom
  - `top_scenario.svh` (5887 lines)
    - task: t_scenario_top(), t_scenario_top_setup(), t_scenario_top_fail_bist(), t_scenario_top_boost_fail_bist(), t_scenario_top_dtc_on_off(), t_scenario_top_random_fail(), t_scenario_top_pad_otp_skip(), t_scenario_top_test_mode(), t_scenario_top_dbi_mbist(), t_scenario_top_bist(), t_scenario_top_sys_clk_test(), t_scenario_top_mcu_step_test(), t_scenario_top_monitor_pad(), t_scenario_top_dr1_ahg_1(), t_scenario_top_dr1_ahg_2(), t_scenario_top_dr1_ahg_3(), t_scenario_top_dr1_ahg_4(), t_scenario_top_dr1_ahg_5(), t_scenario_top_dr1_ahg_6(), t_scenario_top_dr1_ahg_7(), t_scenario_top_dr1_vrr_1(), t_scenario_top_dr1_vrr_2_fqcon0_0(), t_scenario_top_dr1_vrr_2_fqcon0_1(), t_scenario_top_dr1_vrr_2_fqcon1_0(), t_scenario_top_dr1_vrr_2_fqcon1_1() ...
    - 조건 define: AXI_VIP_MONITOR
- scenario entry/task 후보:
  - t_scenario_top()
  - t_scenario_top_setup()
  - t_scenario_top_fail_bist()
  - t_scenario_top_boost_fail_bist()
  - t_scenario_top_dtc_on_off()
  - t_scenario_top_random_fail()
  - t_scenario_top_pad_otp_skip()
  - t_scenario_top_test_mode()
  - t_scenario_top_dbi_mbist()
  - t_scenario_top_bist()
  - t_scenario_top_sys_clk_test()
  - t_scenario_top_mcu_step_test()
  - t_scenario_top_monitor_pad()
  - t_scenario_top_dr1_ahg_1()
  - t_scenario_top_dr1_ahg_2()
  - t_scenario_top_dr1_ahg_3()
  - t_scenario_top_dr1_ahg_4()
  - t_scenario_top_dr1_ahg_5()
  - t_scenario_top_dr1_ahg_6()
  - t_scenario_top_dr1_ahg_7()
  - t_scenario_top_dr1_vrr_1()
  - t_scenario_top_dr1_vrr_2_fqcon0_0()
  - t_scenario_top_dr1_vrr_2_fqcon0_1()
  - t_scenario_top_dr1_vrr_2_fqcon1_0()
  - t_scenario_top_dr1_vrr_2_fqcon1_1()
  - t_scenario_top_dr1_vrr_update()
  - t_scenario_top_dr1_vrr_update_clk_gate_off()
  - t_scenario_top_dr1_line_flag()
  - t_scenario_top_vin_mbist()
  - t_scenario_top_man_clk_dat_div()
  - ... (1개 생략)

### `ucs/`
- 추정 역할: UCS helper/scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (22 lines)
    - 문서 앞부분: # Scenario: ucs / ## Entry / - t_scenario_ucs
  - `ucs_helper.svh` (144 lines)
    - task: reg_command_init()
  - `ucs_scenario.svh` (254 lines)
    - task: t_scenario_ucs(), t_scen_read_ddb(), t_scen_read_pps(), t_scen_read_sdr()
- scenario entry/task 후보:
  - t_scenario_ucs()

### `vgen/`
- 추정 역할: panel/vgen/gpo/host adaptor 기반 VGEN scenario.
- 파일 수: 7
- 파일 구성:
  - `README.md` (260 lines)
    - 문서 앞부분: # VGEN Scenario README / ## 1. Overview / VGEN scenario는 **function-level register randomization 기반으로 DUT 동작을 검증**하기 위한 시나리오이다.
  - `gpo_adaptor.svh` (446 lines)
    - class: gpo_adaptor
    - task: apply_sync_sel(), apply_aid_cyc_off_dur(), apply_overflow(), apply_piece(), apply_eq(), apply_gpo0_1(), apply_gpo4_5(), apply_gpo20_25(), apply_gpo42_49(), apply_gpo50(), apply_all(), default_apply()
  - `host_adaptor.svh` (425 lines)
    - class: host_video_adaptor
    - task: snapshot_apply()
    - 조건 define: __SIM_RTL__, DV_VIDEO_CHECKER_OFF, DV_VIDEO_DATA_CHECKER_OFF
  - `panel_adaptor.svh` (46 lines)
    - class: panel_adaptor
    - task: snapshot_apply_to()
  - `vgen_adaptor.svh` (121 lines)
    - class: vgen_vrr_adaptor, vgen_linefg_adaptor
    - task: snapshot_apply_to(), snapshot_apply()
  - `vgen_helper.svh` (186 lines)
  - `vgen_scenario.svh` (172 lines)
    - task: t_scenario_vgen(), t_scenario_run(), apply_model_to_adaptor(), slpout_normal()
- scenario entry/task 후보:
  - t_scenario_vgen()
  - t_scenario_run()

### `vin/`
- 추정 역할: VIN helper/scenario.
- 파일 수: 3
- 파일 구성:
  - `README.md` (26 lines)
    - 문서 앞부분: # Scenario: vin / ## Entry / - t_scenario_vin
  - `vin_helper.svh` (694 lines)
    - task: test_init_set(), t_check_vin_dsc_ppm(), t_dump_vin_ppm(), dump_ppm_4pixel()
  - `vin_scenario.svh` (206 lines)
    - task: t_scenario_vin(), test_basic(), test_cond(), test_seq_vdo(), test_rand_height(), test_out_opt_vdo()
- scenario entry/task 후보:
  - t_scenario_vin()

## 4. scenario 구조상 주의사항

- `scenario_main.v`는 `ifdef/elsif` compile-time 분기이므로 한 번의 compile에서 선택되는 `t_scenario_main()` 구현은 하나이다.
- scenario folder별 `*_helper.svh`, `*_random.svh`, `*_scenario.svh` 패턴이 많지만, 모든 폴더가 동일 패턴은 아니다. 예: `dbi/`는 현재 `dbi_scenario.svh`만 존재한다.
- 일부 scenario는 `top_random.svh` 같은 다른 scenario folder의 파일을 재사용한다. 예: SRC/GPO/POWER branch에서 top random include가 있다.
- `SCENARIO_ERRFG`, `SCENARIO_PMIC` define 이름은 기존 구조 문서의 `SCENARIO_ERR_FG`, `SCENARIO_PMIC_NEW`와 다르므로 compile define mapping 확인이 필요하다.
- EDS scenario는 `scenario/` 하위 폴더가 아니라 `eds_scen/` 하위 파일을 include하는 branch도 포함한다.
