# Intermodulation self-desense 가능성 분석

## 결론 요약

외부 RF(`f_RF`)가 DDI 내부 클럭(`f_clk`)과 DDI/패키지/보드/패널 주변의 비선형 소자에서 섞여 `f_RF ± n×f_clk` 성분을 만들고, 이것이 안테나 또는 수신 프론트엔드로 재결합되어 self-desense를 유발할 가능성은 있다.

다만 현재 clock-tree 분석 결과만으로는 intermodulation 발생을 직접 증명할 수는 없다. 현재 결과는 “강한 내부 clock switching source가 존재한다”는 1차 위험도 지표이며, 실제 intermodulation 여부는 RF 주파수, 내부 클럭 주파수, 비선형 소자 위치, coupling path, 수신 대역 overlap, spur 레벨 측정으로 확인해야 한다.

## 현재 분석 결과와의 연결

기존 high-strength clock BUF/INV 분석 결과:

- 1002: high-strength CKBUF/CKINV 총 4,863개
  - CKBUF 69개
  - CKINV 4,794개
  - Unique scope 445개
- 1101: high-strength CKBUF/CKINV 총 1,327개
  - CKBUF 1개
  - CKINV 1,326개
  - Unique scope 223개

따라서 intermodulation 관점의 잠재 clock aggressor 강도는 1002가 1101보다 높다. 1002는 고구동 CKBUF도 69개 존재하므로, 단순 inverter 기반 switching보다 clock tree drive/source 관점에서 더 주의가 필요하다.

## 가능한 물리 시나리오

1. 외부 RF 유입

안테나, FPCB, 패널 배선, 전원/그라운드, MIPI/패널 interface, ESD 구조 등을 통해 외부 RF가 DDI 주변으로 유입된다.

2. 내부 clock switching 존재

DDI 내부에는 PLL, pixel/display clock, MIPI 관련 clock, scan/logic clock, SDC/core clock 등이 존재한다. 특히 high-strength CKBUF/CKINV가 많은 clock tree는 supply/substrate/ground bounce를 만들 수 있다.

3. 비선형 mixing 발생

비선형 소자에서 전류/전압 특성이 완전 선형이 아니기 때문에 RF 성분과 clock 성분이 곱셈 성분처럼 섞일 수 있다.

가능한 비선형 지점:

- IO pad ESD diode/clamp
- input buffer, level shifter
- power clamp, LDO, charge pump, bias 회로
- substrate/well junction
- package/PCB 접촉부의 비선형성
- panel/FPCB 주변 접촉/ESD 구조

4. 신규 spur 생성

생성 가능한 성분:

- `f_RF + n×f_clk`
- `f_RF - n×f_clk`
- `m×f_RF ± n×f_clk`
- clock harmonic 기반 `f_RF ± n×f_clk_harmonic`

5. 재방사 또는 수신부 재결합

생성된 spur가 안테나, FPCB, ground return, panel cable, 전원망 등을 통해 다시 RF receiver 쪽으로 coupling되면 self-desense가 발생할 수 있다.

## 가능성이 높아지는 조건

- `f_RF ± n×f_clk`가 수신 대역 또는 receiver IF/baseband 관측 대역에 들어오는 경우
- 내부 clock amplitude가 크거나 edge rate가 빠른 경우
- high-strength CKBUF/CKINV가 특정 영역에 밀집된 경우
- clock tree가 RF-sensitive 경로, antenna feed, FPCB, panel cable, PMIC/LDO 근처와 coupling되는 경우
- ESD clamp나 IO pad가 RF를 받아 비선형 동작할 수 있는 경우
- 전원/ground impedance가 높아 clock switching noise가 supply/substrate modulation으로 커지는 경우
- 특정 display pattern 또는 operating mode에서 clock/toggle activity가 증가하는 경우

## 현재 결과 기준 hotspot 후보

### 1002 전체 scope 기준

1. `logic_scan_top.U_LOGIC_TOP`
   - Total 579
   - CKBUF 4
   - CKINV 575

2. `logic_scan_top.U_LOGIC_TOP.u_rt.u_rt_vpr_sdc_wrapper`
   - Total 170
   - CKBUF 1
   - CKINV 169

3. `logic_scan_top.U_LOGIC_TOP.u_rt.u_rt_slv.u_mipi_cmd_fifo`
   - Total 143
   - CKBUF 1
   - CKINV 142

### 1002 sdc_core 기준

1. `u_ip15`
   - Total 425
   - CKBUF 4
   - CKINV 421

2. `u_ip11`
   - Total 261
   - CKBUF 14
   - CKINV 247

3. `u_ip16`
   - Total 148
   - CKBUF 0
   - CKINV 148

1002에서는 `u_ip15`, `u_ip11`, `u_ip16`이 우선 검토 대상이다. 특히 `u_ip11`은 CKBUF 14개가 집중되어 있어 clock drive 관점에서 더 중요하다.

### 1101 전체 scope 기준

1. `logic_scan_top.U_LOGIC_TOP`
   - Total 186
   - CKBUF 0
   - CKINV 186

2. `logic_scan_top.U_LOGIC_TOP.u_vout.u_vout_gpo_top.u_vout_gpo_reg`
   - Total 151
   - CKBUF 0
   - CKINV 151

3. `logic_scan_top.U_LOGIC_TOP.u_peri.u_sys.u_sys_clk`
   - Total 115
   - CKBUF 0
   - CKINV 115

### 1101 sdc_core 기준

1. `u_ip14`
   - Total 57
   - CKBUF 0
   - CKINV 57

2. `u_ip11`
   - Total 27
   - CKBUF 0
   - CKINV 27

3. `u_ip15`
   - Total 25
   - CKBUF 0
   - CKINV 25

1101은 high-strength clock source 자체는 1002보다 낮지만, `u_vout`, `u_sys_clk`, `u_ip14` 주변은 확인 대상이다.

## 위험도 판단

### 1002

가능성: 중간 이상

이유:

- high-strength clock cell 총량이 크다.
- CKBUF 기반 high-drive source도 존재한다.
- sdc_core 내부 특정 IP에 집중도가 있다.
- clock switching이 supply/substrate/ground를 통해 RF mixing 조건을 만들 가능성이 1101보다 높다.

단, 실제 self-desense 가능성을 높게 확정하려면 수신 대역 overlap과 spur 측정이 필요하다.

### 1101

가능성: 낮음~중간

이유:

- high-strength count는 1002보다 작다.
- CKBUF는 거의 없고 CKINV 중심이다.
- 그래도 `u_vout`, `u_sys_clk`, `u_ip14`처럼 clock 집중 scope가 있으므로 특정 모드에서 coupling될 수 있다.

## 확인 방법

1. 주파수 후보 매핑

필요 데이터:

- 외부 RF 주파수 또는 수신 대역
- DDI 내부 clock list
  - PLL output
  - pixel clock
  - MIPI byte/escape clock
  - display timing clock
  - SDC/core clock
  - scan/test clock
- clock harmonic 범위

계산:

- `f_spur = f_RF ± n×f_clk`
- `f_spur = f_RF ± n×f_clk_harmonic`
- receiver band 또는 IF/baseband 근처 overlap 확인

2. Mode A/B 실험

- display on/off
- 특정 panel pattern 변경
- SDC/core enable/disable
- MIPI activity 변경
- clock gating 강제 또는 frequency 변경
- 1002 vs 1101 동일 조건 비교

관찰 포인트:

- spur가 clock frequency shift만큼 같이 이동하면 intermod 가능성 증가
- display pattern/toggle activity에 따라 spur amplitude가 변하면 내부 clock coupling 가능성 증가

3. RF injection test

- 외부 RF를 sweep하면서 수신 대역 spur 관찰
- DDI clock mode를 바꾸며 `f_RF ± n×f_clk` 라인 추적
- Near-field probe로 DDI, FPCB, panel connector, PMIC 주변 hotspot 확인

4. 전원/ground/substrate coupling 확인

- DDI VDD/VSS noise spectrum 측정
- clock frequency 및 harmonic 성분 확인
- spur 발생 시점에 supply noise와 상관성 확인

## 완화 방향

설계/CTS 관점:

- hotspot scope의 high-strength CKBUF/CKINV drive strength 재검토
- 불필요한 M32/M36 cell downsize 가능성 검토
- clock tree balance 및 local skew constraint 재검토
- clock gating으로 불필요한 active clock 제거
- slew control 가능 여부 검토

물리/전원 관점:

- hotspot 주변 decap 강화
- clock route shielding
- sensitive RF/FPCB/IO 근처 clock route 이격
- return path/ground impedance 개선
- package/PCB에서 DDI와 RF 경로 coupling 최소화

시스템 관점:

- spur가 특정 수신 channel에만 문제라면 clock frequency plan 조정 검토
- spread spectrum clocking 가능성 검토
- problematic mode에서 display timing 또는 internal clock ratio 변경 검토

## 최종 판단

질문하신 intermodulation self-desense 시나리오는 실제 가능한 현상이며, 현재 clock-tree 분석 결과와도 논리적으로 연결된다.

특히 1002는 high-strength clock cell 수와 CKBUF 존재 때문에 1101보다 가능성이 더 높다. 하지만 현재 데이터는 “가능성/위험 후보” 수준이고, 실제 원인 확정에는 `f_RF ± n×f_clk` 주파수 매칭과 RF/near-field 측정이 필요하다.
