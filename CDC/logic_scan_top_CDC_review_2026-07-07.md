# logic_scan_top VC Static CDC review - 2026-07-07

## 요약

cdc 폴더의 VC Static CDC 리포트를 검토했다.

현재 jitter CDC 결과는 다음과 같다.

- Errors: 3638
- Warnings: 6327
- Infos: 28754
- Waived: 16

주요 error:

- CDC_UNSYNC_NOSCHEME: 3141
- CDC_UNSYNC_CTRL: 98
- CDC_UNSYNC_DATA: 43
- CDC_COHERENCY 계열: 209
- CDC_GLITCH_CTRL: 22
- SETUP 계열: 125

## 판단

이번 pass에서 RTL 자동 수정은 하지 않는 것이 맞다. 이유는 현재 리포트의 대부분이 다음 세 종류로 나뉘기 때문이다.

1. Setup/modeling 문제
   - SETUP_CLOCK_UNDECL 7건
   - SRAM/memory macro async pin modeling 관련 SETUP_LIBCELL_* 111건
   - RTL 구조 변경보다 SDC/library setup 보정이 우선이다.

2. MBIST/memory-test 경로
   - clk_mbist_ipXX -> clk_vpr_sdc 또는 SRAM macro pin으로 들어가는 경로가 다수다.
   - mission CDC에서 rm_mbist_on이 off이고 MBIST/functional access가 상호 배타적이라는 proof가 있으면 waiver 후보가 될 수 있다.
   - proof 없이 blanket ignore는 위험하다.

3. Functional reconvergence/unsync
   - PMIC swire, ERRFG checksum, VOUT GPO register-map fanout 등은 실제 protocol proof 또는 RTL CDC scheme 개선이 필요하다.
   - 자동 waiver 대상이 아니다.

## 적용한 변경

파일:
- cdc/user_constraint/logic_scan_top_constraint.tcl

변경:
- MIPI LP clock mux select의 constraint object를 get_ports에서 get_pins로 수정했다.

대상:
- U_LOGIC_TOP/u_mipi_dsi_top/u_dphy_lp11/U_LPRX_DCLK/u_dont_clkmux_out/S0
- U_LOGIC_TOP/u_mipi_dsi_top/u_dphy_lp11/U_LPRX_DCLK/u_dont_clkmux_out/S1

근거:
- 해당 object는 top-level port가 아니라 hierarchical internal mux select pin이다.
- get_ports로 두면 VC Static에서 무시되거나 dynamic mux로 남을 수 있다.

## 자동화 추가

프로젝트 쪽 파일:
- cdc/tools/cdc_report_analyzer.py
- cdc/tools/run_cdc_review_flow.sh
- cdc/analysis/CDC_REVIEW_2026-07-07.md

실행:

```sh
cd cdc
bash tools/run_cdc_review_flow.sh analyze
bash tools/run_cdc_review_flow.sh rerun-vcst
bash tools/run_cdc_review_flow.sh sim
```

생성 예정:
- cdc/analysis/cdc_violation_summary.md
- cdc/analysis/cdc_violation_summary.csv
- cdc/analysis/cdc_waiver_candidates.tcl

waiver candidate tcl은 의도적으로 모두 주석 처리한다. 검토 없이 source하지 않도록 하기 위함이다.

## 다음 확인 항목

1. SETUP_CLOCK_UNDECL 7건
   - OSC2 divider/event clock SDC object mismatch 확인
   - vrom_ctrl r_vr1_slp_updt/r_vr2_slp_updt data-as-clock 구조 확인

2. MBIST mode proof
   - rm_mbist_on mission-mode off 여부
   - MBIST clock과 functional clk_vpr_sdc access의 상호 배타성

3. Functional CDC 개선 후보
   - PMIC swire reconvergence는 related control을 하나의 bundled sync 또는 handshake로 묶는 방향 검토
   - ERRFG/GPO register fanout은 stable qualifier 또는 quasi-static proof 필요

4. Reset integrity
   - 6172 warning은 reset tree를 그룹핑해서 intentional async reset tree만 waiver 후보로 분리 필요
