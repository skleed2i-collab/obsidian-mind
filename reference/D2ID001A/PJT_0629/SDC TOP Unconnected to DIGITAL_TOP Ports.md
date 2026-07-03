---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: SDC TOP Unconnected to DIGITAL_TOP Ports"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/sdc_top_unconnected_to_digital_top_ports.md
imported_at: 2026-07-03 10:19:28 KST
---

# SDC_TOP ports not connected to DIGITAL_TOP

- DIGITAL_TOP: `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_DIGITAL_TOP.v`
- SDC_TOP: `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_SDC_TOP.v`
- ATOP: `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/ATOP.v`

- SDC_TOP total ports: 368
- Connected by same-name opposite-direction rule: 266
- Connected by bit-select vector rule: 14 SDC vectors / 56 DIGITAL numeric ports
- Not connected to DIGITAL_TOP: 88

## Bit-select connected SDC vectors

- `output [0:1641] LV_UMV_GR` -> `LV_UMV_GR_0` uses `LV_UMV_GR[0]`, `LV_UMV_GR_820` uses `LV_UMV_GR[820]`, `LV_UMV_GR_821` uses `LV_UMV_GR[821]`, `LV_UMV_GR_1641` uses `LV_UMV_GR[1641]`
- `output [0:1641] LV_UMV_GI` -> `LV_UMV_GI_0` uses `LV_UMV_GI[0]`, `LV_UMV_GI_820` uses `LV_UMV_GI[820]`, `LV_UMV_GI_821` uses `LV_UMV_GI[821]`, `LV_UMV_GI_1641` uses `LV_UMV_GI[1641]`
- `output [0:1641] LV_GR` -> `LV_GR_0` uses `LV_GR[0]`, `LV_GR_820` uses `LV_GR[820]`, `LV_GR_821` uses `LV_GR[821]`, `LV_GR_1641` uses `LV_GR[1641]`
- `output [0:1641] LV_GI` -> `LV_GI_0` uses `LV_GI[0]`, `LV_GI_820` uses `LV_GI[820]`, `LV_GI_821` uses `LV_GI[821]`, `LV_GI_1641` uses `LV_GI[1641]`
- `output [0:1641] LV_UMV_EM_INIT` -> `LV_UMV_EM_INIT_0` uses `LV_UMV_EM_INIT[0]`, `LV_UMV_EM_INIT_820` uses `LV_UMV_EM_INIT[820]`, `LV_UMV_EM_INIT_821` uses `LV_UMV_EM_INIT[821]`, `LV_UMV_EM_INIT_1641` uses `LV_UMV_EM_INIT[1641]`
- `output [0:1641] LV_UMV_EM` -> `LV_UMV_EM_0` uses `LV_UMV_EM[0]`, `LV_UMV_EM_820` uses `LV_UMV_EM[820]`, `LV_UMV_EM_821` uses `LV_UMV_EM[821]`, `LV_UMV_EM_1641` uses `LV_UMV_EM[1641]`
- `output [0:1641] LV_UMV_GC` -> `LV_UMV_GC_0` uses `LV_UMV_GC[0]`, `LV_UMV_GC_820` uses `LV_UMV_GC[820]`, `LV_UMV_GC_821` uses `LV_UMV_GC[821]`, `LV_UMV_GC_1641` uses `LV_UMV_GC[1641]`
- `output [0:1641] LV_GB` -> `LV_GB_0` uses `LV_GB[0]`, `LV_GB_820` uses `LV_GB[820]`, `LV_GB_821` uses `LV_GB[821]`, `LV_GB_1641` uses `LV_GB[1641]`
- `output [0:1641] LV_UMV_GW` -> `LV_UMV_GW_0` uses `LV_UMV_GW[0]`, `LV_UMV_GW_820` uses `LV_UMV_GW[820]`, `LV_UMV_GW_821` uses `LV_UMV_GW[821]`, `LV_UMV_GW_1641` uses `LV_UMV_GW[1641]`
- `output [0:1641] LV_EM` -> `LV_EM_0` uses `LV_EM[0]`, `LV_EM_820` uses `LV_EM[820]`, `LV_EM_821` uses `LV_EM[821]`, `LV_EM_1641` uses `LV_EM[1641]`
- `output [0:1641] LV_GW` -> `LV_GW_0` uses `LV_GW[0]`, `LV_GW_820` uses `LV_GW[820]`, `LV_GW_821` uses `LV_GW[821]`, `LV_GW_1641` uses `LV_GW[1641]`
- `output [0:1641] LV_UMV_GB` -> `LV_UMV_GB_0` uses `LV_UMV_GB[0]`, `LV_UMV_GB_820` uses `LV_UMV_GB[820]`, `LV_UMV_GB_821` uses `LV_UMV_GB[821]`, `LV_UMV_GB_1641` uses `LV_UMV_GB[1641]`
- `output [0:1641] LV_EM_INIT` -> `LV_EM_INIT_0` uses `LV_EM_INIT[0]`, `LV_EM_INIT_820` uses `LV_EM_INIT[820]`, `LV_EM_INIT_821` uses `LV_EM_INIT[821]`, `LV_EM_INIT_1641` uses `LV_EM_INIT[1641]`
- `output [0:1641] LV_GC` -> `LV_GC_0` uses `LV_GC[0]`, `LV_GC_820` uses `LV_GC[820]`, `LV_GC_821` uses `LV_GC[821]`, `LV_GC_1641` uses `LV_GC[1641]`

## Unconnected SDC_TOP ports by direction

### output (17)

- `output COL_TEST_OUT_DN_LV_A2D`
- `output [0:1641] EM`
- `output COL_TEST_OUT_UP_LV_A2D`
- `output LV_MV_CLA_DRV_D`
- `output LV_MV_CLA_DRV_U`
- `output LV_MV_CLB_DRV_D`
- `output LV_MV_CLB_DRV_U`
- `output LV_MV_CLC_DRV_D`
- `output LV_MV_CLC_DRV_U`
- `output LV_MV_CLD_DRV_D`
- `output LV_MV_CLD_DRV_U`
- `output LV_MV_CLE_DRV_D`
- `output LV_MV_CLE_DRV_U`
- `output LV_MV_CLF_DRV_D`
- `output LV_MV_CLF_DRV_U`
- `output POR1P1_LV_A2D`
- `output POR3P3_LV_A2D`

### inout (7)

- `inout IREF_SRC_D2I`
- `inout [3:0] SRC_MON_OUT_DN_A2P`
- `inout [3:0] SRC_MON_OUT_UP_A2P`
- `inout MCD_L`
- `inout MCD_R`
- `inout PERI_MON_IREF_A2P`
- `inout PERI_MON_VREF_A2P`

### input (64)

- `input AVC_EN_LV_D2A`
- `input [5:0] VREF_CFG_TL08_LV_D2A`
- `input [5:0] VREF_CFG_2P8_LV_D2A`
- `input [4:0] PERI_SEL_TEST_LV_D2A`
- `input [5:0] PTC_CFG_LV_D2A`
- `input [5:0] VREF_CFG_TL06_LV_D2A`
- `input [1:0] AVC_CFG_CCH_LV_D2A`
- `input [1:0] AVC_CFG_SLEW_VH_LV_D2A`
- `input [1:0] AVC_CFG_SLEW_VL_LV_D2A`
- `input [1:0] AVC_CFG_CCL_LV_D2A`
- `input [5:0] VREF_CFG_1P0_LV_D2A`
- `input [5:0] VREF_CFG_TL07_LV_D2A`
- `input [3:0] BGR_CFG_TC_LV_D2A`
- `input [1:0] AVC_CFG_GAIN_VL_LV_D2A`
- `input [3:0] IREF_CFG_LV_D2A`
- `input [1:0] VREF_CFG_TL_LV_D2A`
- `input [1:0] PTC_CFG_GAIN_VL_LV_D2A`
- `input [5:0] VREF_CFG_TL05_LV_D2A`
- `input [1:0] PTC_CFG_GAIN_VH_LV_D2A`
- `input [5:0] VREF_CFG_0P5_LV_D2A`
- `input [1:0] AVC_CFG_GAIN_VH_LV_D2A`
- `input [5:0] VREF_CFG_TH11_LV_D2A`
- `input AVC_EN_VH_LV_D2A`
- `input AVC_EN_VL_LV_D2A`
- `input BGR_PD_LV_D2A`
- `input CFG_IREF_SEL_D2A`
- `input CFG_VREF1_SEL_D2A`
- `input CFG_VREF_TEMP_SEL_D2A`
- `input CFG_VREG1_SEL_D2A`
- `input ELVDD`
- `input ELVSS`
- `input MON_VAINT_B`
- `input MON_VAINT_G`
- `input MON_VAINT_R`
- `input PERI_EN_TEST_LV_D2A`
- `input POR3P3_PD_LV_D2A`
- `input PTC_EN_LV_D2A`
- `input PTC_EN_VH_LV_D2A`
- `input PTC_EN_VL_LV_D2A`
- `input SRC_CLK_STRM_LV_D2A`
- `input TEMP2_DSM_INN_MV_P2A`
- `input TEMP2_DSM_INP_MV_P2A`
- `input VAINT_B`
- `input VAINT_G`
- `input VAINT_R`
- `input VDD11`
- `input VDD33`
- `input VDD33_G`
- `input VDD33_S`
- `input VGH`
- `input VGL_EB`
- `input VGL_EM`
- `input VGL_GC`
- `input VGL_GW`
- `input VGL_VEM`
- `input VPRE`
- `input VREF1`
- `input VREF_PD_LV_D2A`
- `input VREF_TEMP_D2I`
- `input VREG1`
- `input VSS11`
- `input VSS33`
- `input VSS33_G`
- `input VSS33_S`

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
