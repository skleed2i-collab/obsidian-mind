---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: D2ID001A Compile Flow Analysis"
tags:
  - reference
  - rtl
  - d2id001a
  - pjt-0629
  - hermes-import
status: active
quarter: Q3-2026
project: D2ID001A
source_path: /home/tim/pjt_work/D2ID001A/PJT_0629/doc/compile_flow_analysis.md
imported_at: 2026-07-03 10:19:28 KST
---

# Compile / Simulation Flow Analysis

## 1. 목적

이 문서는 현재 프로젝트의 compile/simulation 실행 구조를 정리한다.

분석 대상:

- bin/compile.py
- bin/sim.py
- bin/regress_submit.py
- bin/d2ilib.py
- source/rtl/rtl.f
- source/model/model.f
- tbench/tbench.f
- tbench/package/package.f
- tbench/coverage/coverage.f
- tbench/checker/checker.f
- source/pre/pre.f
- source/post/post.f

## 2. 전체 실행 축

이 프로젝트의 실행 flow는 크게 3단계로 나뉜다.

```text
환경 설정
  ↓
compile/elaboration
  ↓
simulation run
  ↓
regression submit/collect
```

중심 스크립트는 다음과 같다.

```text
bin/d2ilib.py          : 공통 환경/모듈/path utility
bin/compile.py         : xrun/vericom compile + elaborate
bin/sim.py             : compiled snapshot 기반 xrun -R 실행
bin/regress_submit.py  : regress.txt/expand 기반 다중 sim.py 제출
```

## 3. 환경 설정 구조

### 3.1 pjt_path

bin/d2ilib.py의 set_pjt_path()가 핵심이다.

동작:

1. 이미 환경변수 pjt_path가 있으면 그대로 사용
2. 없으면 `git rev-parse --show-toplevel`로 프로젝트 top을 탐지
3. 실패하면 exception 발생

의미:

- 모든 filelist는 `${pjt_path}` 기준으로 작성되어 있다.
- compile/sim 실행 위치가 하위 디렉토리여도 git top을 기준으로 source를 찾는 구조다.
- git working tree 밖에서 실행하면 pjt_path 자동 설정이 실패할 수 있다.

관련 파일:

- bin/d2ilib.py: set_pjt_path(), proj_top()
- source/rtl/rtl.f
- source/model/model.f
- tbench/tbench.f

### 3.2 Tool module

bin/d2ilib.py의 set_tool_env()가 `/usr/bin/modulecmd python`을 사용한다.

compile.py 기본 module:

```text
xcelium/2024.09
vipcat
```

compile.py --verdi 사용 시:

```text
vcs/2024.09
verdi/2024.09
vipcat
```

sim.py 기본 module:

```text
xcelium/2024.09
vipcat
```

주의:

- module 환경이 없는 shell에서는 실행 실패 가능성이 있다.
- /usr/share/Modules 기준 module 환경을 가정한다.
- local PC보다는 사내 CAD 서버 실행을 전제로 한다.

## 4. compile.py 분석

### 4.1 compile.py 역할

compile.py는 compile 및 elaboration 담당 스크립트다.

기본 simulator:

```text
xrun
```

Verdi browse용 compile:

```text
vericom
```

### 4.2 기본 DUT filelist

compile.py 내부의 dut_file_list:

```python
RTL:
  -f $pjt_path/source/model/model.f
  -f $pjt_path/source/rtl/rtl.f

PRE:
  -f $pjt_path/source/model/model_pre.f
  -f $pjt_path/source/pre/pre.f

POST:
  -f $pjt_path/source/model/model_post.f
  -f $pjt_path/source/post/post.f
```

해석:

- RTL sim은 model.f + rtl.f를 같이 compile한다.
- PRE/POST sim은 각각 model_pre/model_post와 pre/post netlist filelist를 사용한다.
- model.f는 chip wrapper/analog model/external model 연결을 포함한다.
- rtl.f는 digital RTL 본체를 포함한다.

### 4.3 기본 testbench filelist

compile.py 내부의 tb_file_list:

```python
RTL:
  -f $pjt_path/tbench/tbench.f
  -f $pjt_path/tbench/dpi.f

PRE:
  -f $pjt_path/tbench/tbench_pre.f
  -f $pjt_path/tbench/dpi.f

POST:
  -f $pjt_path/tbench/tbench_post.f
  -f $pjt_path/tbench/dpi.f
```

해석:

- 기본은 RTL testbench tbench.f + dpi.f다.
- --design-only 옵션을 쓰면 testbench filelist는 제외된다.
- --verdi에서는 dpi.f를 제외한다.

### 4.4 기본 define

compile.py 기본 define_options:

```text
__PAD_DELAY_TYP__
__CLK___TB_FRQ__=10
MIPI_HOST_PATH=U_MIPI_MODEL
MIPI_HOST_TXFSM_PATH=U_MIPI_MODEL.MIPI_TX_FSM
LOGIC_SCAN_TOP_PATH=tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP
LOGIC_TOP_PATH=tb_top.chip.U_DIGITAL_TOP.U_LOGIC_SCAN_TOP.U_LOGIC_TOP
DIGITAL_TOP_PATH=tb_top.chip.U_DIGITAL_TOP
DW_SUPPRESS_WARN
SYNC_2STEP_DELAY
__FLASH_WB__
PJT_PATH="<pjt_path>"
```

중요한 점:

- hierarchy path define이 compile.py에 하드코딩되어 있다.
- waveform/checker/task에서 이 define을 참조할 가능성이 높다.
- `SYNC_2STEP_DELAY`가 기본 enable되어 tbench/injector/sync_2step_delay_injector.sv가 동작할 수 있다.
- `__FLASH_WB__`는 flash model/vendor 선택에 영향을 줄 수 있다.

### 4.5 RTL/PRE/POST mode

compile.py option:

- default: RTL sim
- --presim: pre-sim
- --postsim SS|FF: post-sim

RTL mode:

```text
-timescale 1ns/100ps
-define __SIM_RTL__
-define FUNCTIONAL
```

PRE mode:

```text
-timescale 1ns/100ps
-delay_mode zero
-seq_udp_delay 100ps
-gateloopwarn
-define __SIM_PRE__
```

POST mode:

```text
-timescale 1ns/1ps
-sdf_orig_dir
-sdf_verbose
-sdfstats sdf_stats.log
-maxdelays for SS
-mindelays for FF
-define __SIM_POST__
-define __SIM_SDF_SS__ or __SIM_SDF_FF__
```

### 4.6 override file

compile.py는 현재 실행 디렉토리에 아래 파일이 있으면 기본값 대신 사용한다.

```text
compile.args : xrun option 추가
 tool.f      : default tool option 대체
 define.f    : default define option 대체
 dut.f       : default dut filelist 대체
 tb.f        : default testbench filelist 대체
```

의미:

- sim_rtl/<block> 같은 개별 실행 디렉토리에서 block-specific compile 구성이 가능하다.
- 문제 재현 환경에서는 해당 디렉토리의 override 파일 존재 여부를 반드시 확인해야 한다.

### 4.7 output / artifact

compile.py는 실행 후 다음을 생성/갱신한다.

```text
xcelium.d/ 또는 xcelium.d_alt/
compile.log
compile.history
compile.pylog optional
.json.compile
```

--clean 사용 시:

- xcelium.d 또는 xcelium.d_alt 삭제 시도
- lock file이 있으면 삭제하지 않음
- compile.log/history/pylog 삭제

## 5. source/rtl/rtl.f 구조

rtl.f는 RTL compile의 핵심 filelist다.

주요 순서:

```text
incdir 설정
  ↓
SV interface
  - cmd_reg_if.f
  - ucs_reg_if.sv
  - axi4_if.sv
  - apb_if.sv
  ↓
top
  - logic_scan_top.v
  - logic_top.sv
  - irq.v
  ↓
SRAM wrapper
  - source/rtl/sram/mem_wrap.f
  ↓
subsystem filelist
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
  ↓
padcon files
  ↓
Synopsys DW model
  ↓
sim_ext_ip.f
```

중요 포인트:

- `source/rtl/sram/mem_wrap.f`가 활성화되어 있다.
- `source/rtl/sram_1002/mem_wrap.f`는 주석 처리되어 있다.
- bus는 `bus.flat.f`가 활성화되어 있다.
- DesignWare model 경로가 `/app/synopsys/syn/V-2023.12-SP3/...`로 하드코딩되어 있다.

## 6. source/model/model.f 구조

model.f는 chip-level simulation model dependency를 포함한다.

주요 구성:

```text
UMC40HV STD cell model
model_sram.f
D2ID001A_TOP.v
D2ID001A_DIGITAL_TOP.v
external pad model
MIPI PHY model
VROM roms_lib.v
OSC model
analog_delay_model.v
GPIO model
external eMemory OTP model
TEMP DTS protected model
```

중요 포인트:

- `/PROJECT/...` 절대 경로가 많다.
- SNPS/CDS RNM nettype dependency가 있다.
- 로컬 단독 compile보다 사내 서버 환경에서의 xrun/VCS 실행을 전제로 한다.

## 7. tbench/tbench.f 구조

TB compile entry는 tbench/tbench.f다.

구성 순서:

```text
incdir
  - ${pjt_path}
  - ${pjt_path}/tbench/flash

MIPI TX model
  - tbench/mipi_tx/mipi_tx.f

Packages
  - tbench/package/package.f
  - tbench/random_model/random_model.f

Checkers
  - tbench/checker/checker.f

Models
  - flash_model.v
  - external flash vendor libraries
  - i2c_slave model
  - PMIC model

Coverage
  - tbench/coverage/coverage.f

Injectors
  - clock_delay_injector.sv
  - sync_2step_delay_injector.sv

IBEX simulation-only support

Handlers
  - vrom_hashtag_define.v
  - vrom_handler.sv
  - uvm_handler.sv

TB top
  - host.sv
  - tb_top.sv
```

중요 포인트:

- clock delay injector가 기본 포함된다.
- sync_2step delay injector도 compile.py 기본 define과 맞물려 기본 enable될 수 있다.
- flash model은 local model + vendor absolute path를 같이 사용한다.
- MCU/IBEX simulation model도 TB compile에 포함된다.

## 8. sim.py 분석

### 8.1 역할

sim.py는 compile.py로 생성된 snapshot을 기반으로 simulation을 실행한다.

기본 snapshot:

```text
xcelium.d
```

--alt 사용 시:

```text
xcelium.d_alt
```

### 8.2 실행 전 체크

sim.py는 다음을 확인한다.

```text
xcelium.d 존재 여부
xcelium.d/run.d/invoke_failure 존재 여부
```

snapshot이 없거나 incomplete이면 종료한다.

### 8.3 test directory 생성

sim.py는 plusarg 기반으로 test directory 이름을 만든다.

예:

```text
sim.py +scen=top +foo=1
→ test+scen=top+foo=1
```

plusarg가 없으면:

```text
test-default
```

옵션:

- --tdir: 기존 test directory 사용
- --rerun: 기존 sim.tcl 재사용
- --random: test directory 이름에 random suffix 추가
- --clean: log/wave/db 삭제

sim.optout 파일이 있으면 directory 이름에서 제외할 plusarg를 지정할 수 있다.

### 8.4 sim.tcl 생성

sim.py는 test directory 안에 sim.tcl을 생성한다.

기본 내용:

```tcl
if { [file exists ../init.tcl] } {
    source ../init.tcl
}

# dump.tcl 있으면 source
assertion -style -multiline
run
assertion -list -failed -multiline
```

waveform dump:

- 현재 test directory의 dump.tcl 우선
- 없으면 ../dump.tcl 사용
- --nodump이면 dump 비활성화
- --dumpstart로 dump 시작 전 run 가능

### 8.5 simulation command

sim.py 기본 xrun option:

```text
-R
-licq
-l sim.log
-xmsimargs "-loadrun ${CDN_VIP_LIB_PATH}/64bit/libcdnvipcuvm.so"
```

그리고 다음을 추가한다.

```text
-xmlibdirname ../xcelium.d
-input sim.tcl
```

interactive mode:

```text
-tcl
```

실행 후:

```text
.json.sim
sim.log
waveform db optional
```

## 9. regression flow

bin/regress_submit.py는 regression 제출 wrapper다.

파일 주석 기준 current policy:

- CWD에 xcelium.d가 있으면 compile 생략 후 sim.py 실행
- --clean이면 compile.py --clean 실행 후 sim.py 진행
- sim.py가 Slurm 제출을 담당하며 regress_submit.py는 --sbatch를 덧붙인다
- single mode는 CWD의 regress.txt 사용
- roster mode는 roster.txt에 나열된 각 디렉토리의 regress.txt 사용
- stdout은 console + sim.submit.log에 tee
- command/jobID는 regress.jobs에 기록

추가 기능:

- --replay <test_dir>
  - test directory 이름에서 plusarg 추출
  - sim.log에서 SVSEED 추출
  - snapshot symlink 생성 가능
  - 동일 seed/plusarg로 재실행

- --expand <file>
  - flag subset expansion
  - parameter sweep product expansion
  - +scen=[...]이 있으면 regress.txt를 읽지 않음

## 10. 일반 실행 예시

### 10.1 RTL compile

```bash
compile.py --clean
```

실제 내부 구조:

```text
xrun
  -elaborate
  -uvm
  -access rwc
  -define __SIM_RTL__
  -define FUNCTIONAL
  -f $pjt_path/source/model/model.f
  -f $pjt_path/source/rtl/rtl.f
  -f $pjt_path/tbench/tbench.f
  -f $pjt_path/tbench/dpi.f
```

### 10.2 design-only compile

```bash
compile.py --clean --design-only
```

TB 없이 DUT만 compile한다.

### 10.3 simulation

```bash
sim.py +scen=top
```

예상 동작:

```text
xcelium.d snapshot 확인
test+scen=top directory 생성
sim.tcl 생성
xrun -R -xmlibdirname ../xcelium.d -input sim.tcl +scen=top
```

### 10.4 Slurm simulation

```bash
sim.py +scen=top --sbatch
```

또는 regression:

```bash
regress_submit.py
```

## 11. 주요 리스크

1. 환경 module dependency
   - xcelium/2024.09, vipcat, vcs/2024.09, verdi/2024.09가 필요하다.

2. 절대 경로 dependency
   - /PROJECT 및 /app/synopsys path가 많다.

3. override 파일 영향
   - compile.args, tool.f, define.f, dut.f, tb.f, sim.args가 있으면 기본 flow와 달라진다.

4. hierarchy define 고정
   - compile.py의 LOGIC_TOP_PATH 등이 실제 hierarchy와 불일치하면 checker/task가 깨질 수 있다.

5. snapshot 위치
   - sim.py는 실행 위치 기준 xcelium.d를 찾는다.
   - --compdir 또는 symlink 정책을 이해해야 replay가 가능하다.

6. injector 기본 포함
   - clock_delay_injector, sync_2step_delay_injector가 debug 결과에 영향을 줄 수 있다.

## 12. Debug 체크리스트

compile 문제 발생 시:

```text
1. 현재 위치가 git tree 내부인지 확인
2. pjt_path 값 확인
3. compile.args/tool.f/define.f/dut.f/tb.f 존재 확인
4. source/model/model.f의 /PROJECT path 접근 가능 여부 확인
5. source/rtl/rtl.f의 SRAM variant 확인
6. xcelium module load 여부 확인
7. compile.log의 첫 error 이전 warning/error context 확인
```

sim 문제 발생 시:

```text
1. xcelium.d 또는 xcelium.d_alt 존재 확인
2. xcelium.d/run.d/invoke_failure 확인
3. sim.args 또는 ../sim.args 확인
4. dump.tcl 또는 ../dump.tcl 확인
5. test directory 이름에서 plusarg 확인
6. sim.tcl이 기존 파일인지 자동 생성 파일인지 확인
7. sim.log의 assertion failure 및 UVM fatal 확인
```

regression 문제 발생 시:

```text
1. regress.txt 또는 roster.txt 확인
2. xcelium.d 존재 여부 확인
3. sim.submit.log 확인
4. regress.jobs 확인
5. Slurm pending/running limit 확인
6. replay 시 test directory 이름과 sim.log SVSEED 확인
```

## 13. 결론

이 프로젝트의 compile/sim flow는 다음과 같이 요약된다.

```text
compile.py
  ├─ d2ilib.set_pjt_path()
  ├─ d2ilib.set_tool_env()
  ├─ model.f + rtl.f
  ├─ tbench.f + dpi.f
  └─ xcelium.d snapshot 생성

sim.py
  ├─ xcelium.d 확인
  ├─ test+plusargs directory 생성
  ├─ sim.tcl 생성
  └─ xrun -R 실행

regress_submit.py
  ├─ regress.txt / roster.txt / expand file 해석
  ├─ 필요 시 compile.py 실행
  └─ sim.py --sbatch 제출
```

따라서 compile/sim 디버그의 핵심은 다음 3가지다.

1. 현재 directory의 override 파일 존재 여부
2. pjt_path 및 /PROJECT, /app path 접근성
3. compile.py가 구성한 hierarchy/define/filelist와 실제 TB/DUT hierarchy의 일치 여부

## Related

- [[D2ID001A Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
