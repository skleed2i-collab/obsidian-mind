---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: D2ID001A Reset Boot Flow Analysis"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/reset_boot_flow_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# Reset / Boot Flow Analysis

## 1. 목적

이 문서는 현재 RTL의 reset 및 boot 관련 흐름을 정리한다.

중점:

- 외부 reset source가 내부 reset tree로 변환되는 경로
- power/soft/bus/sleep/mcu reset 분기
- POR 이후 system FSM의 boot 시퀀스
- OTP/VROM/MCU boot 관련 NVM FSM 경로
- debug 시 확인해야 할 주요 signal과 hierarchy

## 2. 분석 근거 파일

확인한 주요 파일:

```text
source/rtl/top/logic_top.sv
source/rtl/peri/peri.v
source/rtl/peri/peri.f
source/rtl/peri/sys/sys.v
source/rtl/peri/sys/sys_reset.v
source/rtl/peri/sys/sys_fsm.v
source/rtl/peri/sys/sys_fsm_cdc.v
source/rtl/peri/sys/sys_fsm_nvm.v
source/rtl/peri/sys/sys_fsm_nvm_reset.v
source/rtl/peri/sys/sys_fsm_nvm_vrom.v
source/rtl/peri/sys/sys_fsm_nvm_im.v
source/rtl/peri/sys/sys_fsm_nvm_mcu.v
source/rtl/peri/mcu/rv_top.v
source/rtl/top/mmap.v
```

## 3. Reset/boot 관련 상위 hierarchy

기준 path:

```text
tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP
  └─ u_peri
      └─ sys 계열
          ├─ sys_reset
          ├─ sys_clk
          ├─ sys_fsm
          ├─ sys_fsm_cdc
          ├─ sys_fsm_wakeup
          ├─ sys_fsm_nvm
          │   ├─ sys_fsm_nvm_vrom
          │   ├─ sys_fsm_nvm_reset
          │   ├─ sys_fsm_nvm_im
          │   └─ sys_fsm_nvm_mcu
          └─ sys_mst / sys_mst1
```

`logic_top.sv`에서는 `u_peri` instance가 reset/clock/system FSM을 받아서 내부 block에 필요한 reset/clock을 생성한다.

## 4. Reset source 입력

`logic_top.u_peri`로 들어가는 reset source는 다음과 같이 연결되어 있다.

```text
.i_pad_scan_reset       ( IO_LRSEL_Y      )
.i_pad_resx             ( TO_RESX_SPROUT  )
.i_por_reset            ( TO_PORB_VDD     )
.i_sofreset_dly         ( DL_OUT          )
.i_dsi_phy_rstb         ( w_phy_rstb      )
.i_scan_mode            ( w_scan_mode     )
.rc_swreset             ( rc_swreset      )
.i_pcd_ccd_soft_rstb    ( w_pcd_ccd_soft_rstb )
.i_chksum_esd_reset     ( w_chksum_esd_reset  )
```

상위 의미:

- `TO_RESX_SPROUT`: pad reset path
- `TO_PORB_VDD`: POR reset path
- `DL_OUT`: analog delay model output, soft-reset delay 관련
- `w_phy_rstb`: MIPI PHY reset status
- `rc_swreset`: UCS/register command 기반 software reset
- `w_pcd_ccd_soft_rstb`: misc/PCD/CCD 관련 soft reset mask
- `w_chksum_esd_reset`: checksum/ESD reset source
- `IO_LRSEL_Y` + `w_scan_mode`: scan mode reset mux path

## 5. sys_reset reset tree

`source/rtl/peri/sys/sys_reset.v`가 reset tree의 핵심이다.

### 5.1 기본 reset 조합

```text
w_rstb_pad = buffer(i_pad_resx)
w_rstb_por = buffer(i_por_reset)
w_rstb_pwr = w_rstb_pad & w_rstb_por
```

즉 power reset 계열의 가장 기본 조건은 RESX와 POR가 모두 high인 것이다.

### 5.2 delayed reset

```text
w_rstb_dly = w_rstb_pwr & buffer(i_sofreset_dly)
```

`i_sofreset_dly`는 `logic_top`에서 `DL_OUT`으로 들어오며, `D2ID001A_TOP`의 `analog_delay_model`과 연결된다.

### 5.3 soft reset

```text
w_rstb_sof = w_rstb_pwr
           & rc_swreset
           & i_pcd_ccd_soft_rstb
           & i_dsi_phy_rstb
```

중요:

- soft reset release는 단순히 RESX/POR만으로 충분하지 않다.
- `rc_swreset`, `i_pcd_ccd_soft_rstb`, `i_dsi_phy_rstb`가 모두 release되어야 한다.
- MIPI PHY reset 상태가 `rstb_sof`에 직접 영향을 준다.

### 5.4 bus reset

```text
w_rstb_bus = w_rstb_sof & (~i_bus_reset)
```

`i_bus_reset`은 `sys_fsm`에서 생성되는 bus reset 제어 신호다.

### 5.5 sleep reset

```text
w_rstb_slp = w_rstb_bus & (i_sleep_reset || rm_man_sleep_reset_on==8'h5a)
```

의미:

- sleep reset은 bus reset 이후에 추가적으로 sleep FSM 상태와 manual override register 영향을 받는다.
- `rm_man_sleep_reset_on == 8'h5a`는 manual sleep reset release key로 보인다.

### 5.6 MBIST / flash / MCU / ESD reset

```text
w_rstb_mbist = w_rstb_bus & (~w_mbist_reset)
r_rstb_flash = synchronized release under clk_axi_sys, based on rs_flash_slv_rst
r_rstb_mcu   = registered rm_mcu_reset under clk_axi_sys after w_rstb_bus
w_rstb_esd   = rm_esd_reset ? 1'b0 : w_rstb_slp & (~i_chksum_esd_reset)
```

MCU reset에서 특히 중요한 점:

```verilog
always @(posedge clk_axi_sys or negedge w_rstb_bus)
    if (~w_rstb_bus) r_rstb_mcu <= 1'b0;
    else             r_rstb_mcu <= rm_mcu_reset;
```

따라서 MCU reset release는 `w_rstb_bus` release 이후 `rm_mcu_reset` register 값에 의해 결정된다.

### 5.7 scan reset mux

`sys_reset.v`는 scan mode에서 reset outputs를 scan reset으로 mux한다.

```text
CLK_MUX(... .S(i_scan_mode), .Y(rstb_*))
```

주의:

- scan mode가 enable되면 functional reset tree가 아닌 `i_pad_scan_reset` 기반으로 reset이 움직인다.
- debug 시 `w_scan_mode`/`i_scan_mode` 상태를 반드시 확인해야 한다.

## 6. Reset output 종류와 의미

`peri.v`와 `sys_reset.v` 기준 reset output:

```text
rstb_por      : POR only
rstb_pwr      : POR + RESX
rstb_dly      : power + analog delay
rstb_sof      : power + soft reset + MIPI PHY reset related
rstb_bus      : soft reset + bus reset control
rstb_slp      : bus reset + sleep reset control
rstb_mbist    : bus reset + MBIST reset pulse
rstb_vdo      : video mode reset, currently sleep reset base
rstb_mcu      : bus reset + rm_mcu_reset registered release
rstb_esd      : sleep reset + checksum/ESD reset control
```

## 7. Clock generation dependency

`u_peri`는 reset뿐 아니라 clock source와 clock gates도 생성한다.

입력:

```text
i_exclk       = IO_EXCLK_Y
i_osc_dat     = TO_OSC_CLK
i_osc2        = TO_OSC1_CLK
clk_mipi_rt
clk_ref_byte
clk_mipi_vid
i_clk_scan_pipe = IO_TMODE_Y[0]
```

제어:

```text
i_pad_exclk_sel
i_aofc_en
i_clk_axi_sys_lp
i_clk_rt_axi_sys_on
i_clk_rt_mipi_on
i_*_sram_*_clk_en
IP_OSC_CLK_EN
```

출력 예:

```text
clk_sys
clk_axi_sys
clk_axi_sys_lp
clk_rt
clk_rt_mipi_fifo
clk_mcu
clk_mcu_watchdog
clk_otp
clk_flash_slv
clk_vin
clk_vpr_sdc
clk_pwr_ldo
```

디버그 포인트:

- reset이 release되어도 clock gate가 닫혀 있으면 FSM이 진행하지 않는다.
- 특히 `clk_sys`, `clk_axi_sys`, `clk_axi_sys_lp`, `clk_rt`, `clk_mcu`가 각각 어느 reset domain을 움직이는지 구분해야 한다.

## 8. sys_fsm boot sequence 개요

`source/rtl/peri/sys/sys_fsm.v`는 system boot/power/sleep FSM이다.

주요 state:

```text
SYS________________________IDLE
SYS_RST_____OSC____________WAIT
SYS_RST_____RVDD_2ND_______WAIT
SYS_RST_____LOAD____________DLY
SYS_RST_____VROM____________REQ
SYS_RST_____VROM____________DON
SYS_RST_MCU_OTP_____________REQ
SYS_RST_MCU_EXTRA0__________REQ
SYS_RST_MCU_MEMOFF__________REQ
...
SYS______________________ACTIVE
SYS_POF_* / SYS_PON_* / SYS_SLEEP_*
```

초기 reset boot 흐름의 핵심 후보:

```text
IDLE
  → OSC wait
  → RVDD / reset load delay
  → VROM default load request
  → VROM done
  → MCU OTP / extra / memoff request
  → ACTIVE
```

정확한 next 조건은 `sys_fsm.v`의 state transition을 봐야 하지만, state 이름상 reset 이후 다음 작업이 포함된다.

- oscillator wakeup wait
- RVDD/boost wait
- reset loading
- VROM default loading
- MCU OTP/IM setup
- active 진입

## 9. sys_fsm CDC

`sys_fsm`는 여러 clock domain 신호를 `clk_sys`로 동기화한다.

대표 입력 domain:

- `clk_wakeup`: OSC wakeup counter
- `clk_rt`: register pulse, sleep in/out command, OTP done pulse
- `clk_axi_sys`: DMA/default load done 등
- `clk_vin`: display on/off done, vgen termination
- `clk_otp`: OTP done 관련

`sys_fsm_cdc`에서 주요 sync:

```text
rs_sleep_in/out     @clk_rt     → clk_sys
rs_otp_load_*_done  @clk_rt     → clk_sys
i_vrom1_default_load_dma_end @clk_axi_sys → clk_sys
i_dtc_disp_*_don    @clk_vin    → clk_sys
```

디버그 포인트:

- source domain에서 pulse가 발생해도 destination `clk_sys`가 gated/off이면 pulse를 놓칠 수 있다.
- first valid clock edge 이전에 pulse가 지나가면 FSM stuck처럼 보일 수 있다.

## 10. NVM/boot orchestration

`sys_fsm_nvm.v`는 VROM/OTP/MCU boot related sub-flow를 관리한다.

입력 request:

```text
i_vrom_req
i_mcu_reset_otp_req
i_mcu_slpout_otp_req
i_mcu_flash_req
i_mcu_flash_ip08_req
i_mcu_normal_req
i_mcu_memoff_req
i_mcu_extra0_req ... i_mcu_extra4_req
```

주요 submodule:

```text
sys_fsm_nvm_vrom   : VROM default load / manual VROM load
sys_fsm_nvm_reset  : RESET OTP loading S/W instruction memory start address loading
sys_fsm_nvm_im     : MCU instruction memory address setup for various modes
sys_fsm_nvm_mcu    : MCU on/off request handling
```

NVM FSM state:

```text
NVM_IDLE
NVM_VROM_REQ
NVM_RESET_OTP_IM
NVM_SLPOUT_OTP_IM
NVM_FLASH_IM
NVM_IP08_IM
NVM_MEMOFF_IM
NVM_EXTRA0_IM ... EXTRA4_IM
NVM_MCU_REQ
NVM_MCU_OFF_WAIT
NVM_NORMAL_IM
NVM_NORML_MCU_REQ
NVM_DON
```

## 11. RESET OTP → MCU IM address loading

`sys_fsm_nvm_reset.v`의 목적 주석:

```text
RESET OTP loading S/W(MCU) instruction memory start address loading
```

동작 요약:

1. `i_reset_otp_req` 수신
2. `MMAP_SYS_CLK_OTP`에 write하여 OTP clock/path enable
3. OTP 영역에서 MCU IM header를 3→0 순서로 read
4. 유효 header가 없으면 default address를 read
5. `MMAP_SYS_MCU_IM_ADR`에 write
6. CRC header/default를 read 후 `MMAP_SYS_MCU_IM_CRC_ADR`에 write
7. control header/default를 read 후 control address에 write
8. done pulse 발생

관여 address symbol:

```text
MMAP_SYS_CLK_OTP
MMAP_SYS_OTP_MCU_IM__HEADER3/2/1/0
MMAP_SYS_MCU_RESET_OTP_LOAD_IM_ADR
MMAP_SYS_MCU_IM_ADR
MMAP_SYS_OTP_MCU_CRC_HEADER3/2/1/0
MMAP_SYS_MCU_RESET_OTP_IM_CRC0
MMAP_SYS_MCU_IM_CRC_ADR
MMAP_SYS_OTP_MCU_CTL_HEADER3/2/1/0
```

주의:

- OTP read 값이 `32'h0` 또는 all-1이면 다음 header 후보를 시도한다.
- 결국 default read path로 fallback한다.
- sys master ack가 없으면 해당 state에서 정지한다.

## 12. MCU boot 관련

`source/rtl/peri/mcu/rv_top.v`는 RISC-V top이다.

주요 입력:

```text
rstb_sof
rstb_slp
rstb_mcu
clk_mcu
clk_axi_sys
clk_axi_sys_lp
clk_rt
rm_mcu_act
rm_mcu_im_adr
rm_mcu_im_crc_adr
rm_mcu_im_crc_on
rm_mcu_im_trans_x4byte
```

내부 구조:

```text
rv_top
  ├─ rv_mst : MCU master / instruction loading / core side
  └─ rv_slv : AXI slave / IRQ/watchdog/control registers
```

MCU가 실제로 동작하려면 최소한 다음이 맞아야 한다.

```text
rstb_mcu released
clk_mcu active
rm_mcu_act asserted or expected mode enabled
rm_mcu_im_adr valid
instruction memory load/CRC path done
IRQ/watchdog reset 상태 정상
```

## 13. Reset/boot debug checklist

### 13.1 외부 reset 단계

확인 signal:

```text
TO_RESX_SPROUT
TO_PORB_VDD
DL_OUT
w_phy_rstb
rc_swreset
w_pcd_ccd_soft_rstb
w_scan_mode
```

판단:

- `TO_RESX_SPROUT`와 `TO_PORB_VDD`가 high인지
- `DL_OUT`이 기대 시간 후 high가 되는지
- `w_phy_rstb`가 low에 머물러 soft reset을 막지 않는지
- scan mode가 의도치 않게 reset mux를 바꾸지 않는지

### 13.2 내부 reset tree

확인 signal:

```text
rstb_por
rstb_pwr
rstb_dly
rstb_sof
rstb_bus
rstb_slp
rstb_mcu
rstb_mbist
rstb_esd
```

판단:

- 어느 reset 단계에서 release가 멈추는지 찾는다.
- `rstb_sof`가 안 풀리면 soft reset 조건을 역추적한다.
- `rstb_bus`가 안 풀리면 `i_bus_reset/sys_fsm`을 본다.
- `rstb_mcu`가 안 풀리면 `rm_mcu_reset`과 `clk_axi_sys` edge를 본다.

### 13.3 clock 단계

확인 signal:

```text
clk_sys
clk_axi_sys
clk_axi_sys_lp
clk_rt
clk_mcu
clk_wakeup
clk_otp
IP_OSC_CLK_EN
w_clk_rt_axi_sys_on
w_clk_rt_mipi_on
```

판단:

- FSM clock이 실제 toggle하는지
- reset release와 clock enable 순서가 맞는지
- LP clock과 normal clock이 gating 때문에 멈춰 있지 않은지

### 13.4 sys_fsm/NVM 단계

확인 signal:

```text
ro_sys_fsm
ro_sys_fsm_nvm_fsm
ro_sys_fsm_nvm_vrom_ctl_fsm
ro_sys_fsm_nvm_reset_fsm
ro_sys_fsm_nvm_im_fsm
ro_sys_fsm_nvm_mcu_fsm
o_vrom1_default_load_dma_srt
i_vrom1_default_load_dma_end
o_mst_fsm_wreq/o_mst_fsm_rreq
i_mst_fsm_rden
```

판단:

- `SYS_RST_*` state 중 어느 곳에서 멈추는지
- NVM FSM이 `NVM_VROM_REQ`, `NVM_RESET_OTP_IM`, `NVM_MCU_REQ` 등에서 ack/done을 기다리는지
- sys master access가 bus에서 ack를 받는지

## 14. 잠재 리스크

1. MIPI PHY reset이 `rstb_sof` 조건에 들어가므로 MIPI model/PHY 초기화 실패가 전체 soft reset을 막을 수 있다.
2. `rstb_mcu`는 `rm_mcu_reset` register를 `clk_axi_sys`에서 sampling하므로 register write와 clock availability 순서가 중요하다.
3. NVM reset load는 sys master ack를 기다리므로 bus/OTP/RT clock/reset이 하나라도 막히면 boot가 멈춘다.
4. OTP header가 0/all-1일 때 fallback path가 있으나 default address register도 잘못되어 있으면 MCU IM address가 비정상일 수 있다.
5. scan mode는 reset mux 전체를 바꾸므로 `w_scan_mode`가 잘못 잡히면 functional reset 분석이 무의미해진다.
6. CDC pulse는 destination clock이 없거나 reset 중이면 유실될 수 있다.

## 15. 결론

Reset/boot flow는 다음 순서로 보는 것이 좋다.

```text
External PAD/POR/analog reset
  → sys_reset reset tree
  → clock generation/gating
  → sys_fsm reset boot states
  → sys_fsm_nvm VROM/OTP/IM/MCU setup
  → rv_top rstb_mcu + clk_mcu + rm_mcu_act
  → normal active state
```

가장 먼저 확인할 debug breakpoint는 다음이다.

```text
rstb_pwr → rstb_sof → rstb_bus → rstb_slp → rstb_mcu
ro_sys_fsm
ro_sys_fsm_nvm_fsm
ro_sys_fsm_nvm_reset_fsm
clk_sys / clk_axi_sys / clk_rt / clk_mcu
```

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
