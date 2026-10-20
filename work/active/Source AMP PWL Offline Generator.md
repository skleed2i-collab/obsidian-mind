---
date: "2026-10-20"
description: "Source AMP simulation용 S[9:0] Decimal BUS PWL/event-table 생성기 구현 및 출력 형식 정의"
project: source-amp-pwl-generator
status: active
quarter: Q4-2026
tags:
  - work-note
  - project/source-amp-pwl
  - simulation
  - pwl
  - digital-design
source_path: "/home/tim/work/source_amp_pwl"
---

# Source AMP PWL Offline Generator

## Context

Source AMP 출력 시뮬레이션을 위한 입력 stimulus 생성기를 구현했다. 개인 PC에서 서버, Python, 패키지 설치, 인터넷 연결 없이 실행할 수 있도록 단일 HTML 애플리케이션으로 제공한다.

실행 파일: `/home/tim/work/source_amp_pwl/Source_AMP_PWL_Offline.html`

## Implemented Behavior

### Frame 및 이미지 설정

- 입력: line time (µs), 총 line 수, VP line 수, Source AMP 채널 수, 1:1/2:1/3:1 MUX timing.
- 총 line = active line + VP. VP는 frame 끝에 추가된다.
- VP 동안 `DATA=0`, `GAMMA=00`, 따라서 BUS Decimal은 `0`이다.
- 패턴: 단색, Zebra (line/원본 column 기준), line gradation, Random.
- Random은 매 generation마다 `crypto.getRandomValues`를 사용하며 실제 생성값은 `mapping.csv`에 기록된다.

### 2:1 MUX 매핑

- 각 원본 짝수 column은 `R → G`, 홀수 column은 `B → G` 순서이다.
- zero-based odd line에서는 인접 Source AMP 채널이 교환된다.
- 예시 2채널:
  - line 0: `S1=R[0,0] → G[0,0]`, `S2=B[0,1] → G[0,1]`
  - line 1: `S1=B[1,1] → G[1,1]`, `S2=R[1,0] → G[1,0]`
- 1:1 기본 가정은 channel별 RGB 반복, 3:1 기본 가정은 channel 내 `R → G → B`이다. 실제 회로 mapping이 다르면 변경 필요.

### Decimal BUS PWL 형식

채널별로 `S1.pwl`, `S2.pwl`처럼 하나의 파일을 생성한다. bit별 DATA/GAMMA 파일이나 전압 값은 생성하지 않는다.

```text
S[n][9:0] = {GAMMA[1:0], DATA[7:0]}
BUS Decimal = (GAMMA << 8) | DATA = GAMMA × 256 + DATA
```

- Gamma: Red=`2'b10` = 2, Green=`2'b01` = 1, Blue=`2'b00` = 0.
- Red DATA 범위: 512~767; Green: 256~511; Blue: 0~255.
- 각 PWL 파일은 헤더 없는 `time(s) decimal_bus_value` 2열 형식이다.

### BUS Rise/Fall Event

BUS 값이 바뀔 때, 입력한 `BUS Rise / Fall 시간 (ns)`에 따라 transition window를 다음 두 event row로 기록한다.

```text
t                  previous BUS Decimal
t + rise/fall      next BUS Decimal
```

기본 예시 (`line=4µs`, `T1=1.7µs`, `rise/fall=1ns`, RGB=`255/128/64`)의 `S1.pwl` 일부:

```text
0             767
0.0000017     767
0.000001701   384
0.000004      384
0.000004001    64
```

Decimal 열은 digital BUS event로 읽는다. Decimal 값을 선형 보간하거나 scalar analog-voltage PWL source에 직접 연결하면 유효하지 않은 중간 BUS code가 생기므로 금지한다. 각 값은 다음 event timestamp까지 hold한다.

## Generated ZIP Contents

- `S<n>.pwl`: 채널별 Decimal BUS event table.
- `mapping.csv`: line/channel/MUX slot/time/color/DATA/GAMMA/BUS Decimal mapping.
- `settings.json`: 사용한 GUI 설정 및 파생 timing.
- `README.txt`: ZIP 내부 형식 안내.

## Verification

- Python reference mapping과 browser JavaScript의 Decimal BUS 결과 비교.
- 1:1, 2:1, 3:1 및 solid/Zebra/gradation pattern 검사.
- VP, BUS Decimal packing, Random fresh data, ZIP CRC, rise/fall transition row 및 invalid timing 검증.
- 2026-10-20 기준 자동 테스트 8개 통과.
- Firefox에서 `file://`로 단일 HTML을 직접 열고 ZIP 다운로드/생성 검증.
- 실제 SPICE/simulator import는 simulator별 BUS stimulus reader 문법이 필요하므로 별도 검증되지 않음.

## Related

- [[Index|Work Notes]]
- [[Home]]
