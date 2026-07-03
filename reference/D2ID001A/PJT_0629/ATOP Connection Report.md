---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: ATOP Connection Report"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/ATOP_connection_report.md
imported_at: 2026-07-03 10:19:28 KST
---

# ATOP connection report

- Generated file: `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/ATOP.v`
- DIGITAL module: `D2ID001A_DIGITAL_TOP` from `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_DIGITAL_TOP.v`
- SDC module: `D2ID001A_SDC_TOP` from `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_SDC_TOP.v`
- ATOP external ports: 572
- Internal DIGITAL -> SDC connections: 261
- Internal SDC -> DIGITAL connections: 5
- Width mismatches on internal connections: 0
- _N bit-select mappings applied: 0

## Notable unconnected-by-name ports
These remain ATOP external ports because no opposite-direction same-name match was found.

- SDC inputs exposed externally: 64
  - AVC_EN_LV_D2A
  - AVC_EN_VH_LV_D2A
  - AVC_EN_VL_LV_D2A
  - BGR_PD_LV_D2A
  - CFG_IREF_SEL_D2A
  - CFG_VREF1_SEL_D2A
  - CFG_VREF_TEMP_SEL_D2A
  - CFG_VREG1_SEL_D2A
  - ELVDD
  - ELVSS
  - MON_VAINT_B
  - MON_VAINT_G
  - MON_VAINT_R
  - PERI_EN_TEST_LV_D2A
  - POR3P3_PD_LV_D2A
  - PTC_EN_LV_D2A
  - PTC_EN_VH_LV_D2A
  - PTC_EN_VL_LV_D2A
  - SRC_CLK_STRM_LV_D2A
  - TEMP2_DSM_INN_MV_P2A
  - TEMP2_DSM_INP_MV_P2A
  - VAINT_B
  - VAINT_G
  - VAINT_R
  - VDD11
  - VDD33
  - VDD33_G
  - VDD33_S
  - VGH
  - VGL_EB
  - VGL_EM
  - VGL_GC
  - VGL_GW
  - VGL_VEM
  - VPRE
  - VREF1
  - VREF_PD_LV_D2A
  - VREF_TEMP_D2I
  - VREG1
  - VSS11
  - VSS33
  - VSS33_G
  - VSS33_S
  - VREF_CFG_TL08_LV_D2A
  - VREF_CFG_2P8_LV_D2A
  - PERI_SEL_TEST_LV_D2A
  - PTC_CFG_LV_D2A
  - VREF_CFG_TL06_LV_D2A
  - AVC_CFG_CCH_LV_D2A
  - AVC_CFG_SLEW_VH_LV_D2A
  - AVC_CFG_SLEW_VL_LV_D2A
  - AVC_CFG_CCL_LV_D2A
  - VREF_CFG_1P0_LV_D2A
  - VREF_CFG_TL07_LV_D2A
  - BGR_CFG_TC_LV_D2A
  - AVC_CFG_GAIN_VL_LV_D2A
  - IREF_CFG_LV_D2A
  - VREF_CFG_TL_LV_D2A
  - PTC_CFG_GAIN_VL_LV_D2A
  - VREF_CFG_TL05_LV_D2A
  - PTC_CFG_GAIN_VH_LV_D2A
  - VREF_CFG_0P5_LV_D2A
  - AVC_CFG_GAIN_VH_LV_D2A
  - VREF_CFG_TH11_LV_D2A
- DIGITAL outputs exposed externally: 404
  - SRC_CLK_STRIM_LV_D2A
  - HS_C_OPT0_I
  - HS_C_OPT1_I
  - HS_C_OPT2_I
  - HS_C_OPT3_I
  - HS_C_OPTC_I
  - HS_DATA_EN_I
  - HS_DESCLK0_EN_I
  - HS_DESCLK1_EN_I
  - HS_DESCLK2_EN_I
  - HS_DESCLK3_EN_I
  - HS_DES_BUF_I
  - HS_DUM_I
  - HS_EQ_BIASDN_I
  - HS_HSEQ_I
  - HS_HSRX_EN0_I
  - HS_HSRX_EN1_I
  - HS_HSRX_EN2_I
  - HS_HSRX_EN3_I
  - HS_HSRX_ENC_I
  - HS_HSRX_PNSW_I
  - HS_HS_BIAS_I
  - HS_ITRIM_I
  - HS_LDO_OD_EN_I
  - HS_LPCD_EN0_I
  - HS_LPCD_EN1_I
  - HS_LPCD_EN2_I
  - HS_LPCD_EN3_I
  - HS_LPRX_EN0_I
  - HS_LPRX_EN1_I
  - HS_LPRX_EN2_I
  - HS_LPRX_EN3_I
  - HS_LPRX_ENC_I
  - HS_LPTXLOAD_I
  - HS_LPTX_EN0_I
  - HS_LPTX_EN1_I
  - HS_LPTX_EN2_I
  - HS_LPTX_EN3_I
  - HS_LPTX_N0_I
  - HS_LPTX_N1_I
  - HS_LPTX_N2_I
  - HS_LPTX_N3_I
  - HS_LPTX_P0_I
  - HS_LPTX_P1_I
  - HS_LPTX_P2_I
  - HS_LPTX_P3_I
  - HS_LP_FILT_I
  - HS_RTERM_EN0_I
  - HS_RTERM_EN1_I
  - HS_RTERM_EN2_I
  - HS_RTERM_EN3_I
  - HS_RTERM_ENC_I
  - HS_RTERM_SEL_I
  - HS_ULPS_EN_I
  - HS_VDDLP_EN2_I
  - HS_VDDLP_EN_I
  - HS_VTRIM_I
  - IO_DIS_EXCLK_CI
  - IO_DIS_EXCLK_CPU
  - IO_EXCLK_CI
  - IO_EXCLK_CPU
  - IO_F_CSN_CE
  - IO_F_CSN_CI
  - IO_F_CSN_CPU
  - IO_F_CSN_DRV
  - IO_F_CSN_IN
  - IO_F_IO_0_DRV
  - IO_F_IO_1_DRV
  - IO_F_IO_2_DRV
  - IO_F_IO_3_DRV
  - IO_F_IO_CE
  - IO_F_IO_CI
  - IO_F_IO_CPU
  - IO_F_IO_IN
  - IO_F_SCK_CE
  - IO_F_SCK_CI
  - IO_F_SCK_CPU
  - IO_F_SCK_IN
  - IO_F_SCLK_DRV
  - IO_LRSEL_CI
  - ... 324 more

## Possible spelling difference
- DIGITAL has `SRC_CLK_STRIM_LV_D2A`, SDC has `SRC_CLK_STRM_LV_D2A`; left unconnected/external because names differ.

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
