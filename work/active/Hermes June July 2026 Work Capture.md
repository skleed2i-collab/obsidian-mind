---
date: 2026-07-03
description: "6월~7월 Hermes 세션에서 생성/확인됐지만 Obsidian에 흩어져 있거나 저장되지 않았던 RTL 작업 내용을 모은 작업 캡처"
tags:
  - work-note
  - rtl
  - hermes-capture
  - d2id001a
  - d2i1002
status: active
quarter: Q3-2026
project: Hermes RTL Work Capture
source_path: session_search summaries and local project docs
updated_at: 2026-07-03 10:19:28 KST
---

# Hermes June July 2026 Work Capture

Back to [[Index]], [[D2ID001A Reference]], [[D2I1002 Reference]].

이 노트는 2026년 6월~7월 Hermes 세션 중 Obsidian에 아직 정리되지 않았거나 project-local `doc/`에만 남아 있던 내용을 vault로 모은 캡처이다.

## 1. D2ID001A / PJT_0629 RTL 통합 작업

주요 경로:

```text
/home/tim/pjt_work/D2ID001A/PJT_0629
```

핵심 산출물:

- [[ATOP Connection Report]]
- [[SDC TOP Port Groups]]
- [[logic_scan_top Underscore Number Ports]]
- [[DIGITAL_TOP logic_scan Numeric Port Update]]
- [[SDC TOP Unconnected to DIGITAL_TOP Ports]]
- [[D2ID001A Top Hierarchy Analysis]]
- [[D2ID001A Compile Flow Analysis]]
- [[D2ID001A Project Structure Analysis]]

### ATOP wrapper 생성 요약

- 입력 module:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_DIGITAL_TOP.v`
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_SDC_TOP.v`
- 생성 파일:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/ATOP.v`
- 생성 리포트:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/doc/ATOP_connection_report.md`
- 파싱 결과:
  - `D2ID001A_DIGITAL_TOP`: 736 ports
  - `D2ID001A_SDC_TOP`: 368 ports
  - common ports: 266
  - DIGITAL output -> SDC input: 261
  - SDC output -> DIGITAL input: 5
  - ATOP external ports: 572
  - width mismatch: 0
- 특이사항:
  - `SRC_CLK_STRIM_LV_D2A` vs `SRC_CLK_STRM_LV_D2A` 이름 불일치는 자동 연결하지 않고 외부 port로 남겼다.

### SDC_TOP port 그룹화 요약

- 대상:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/model/D2ID001A_SDC_TOP.v`
- 생성 문서:
  - [[SDC TOP Port Groups]]
- 결과:
  - total ports: 368
  - output: 36
  - inout: 7
  - input: 325
  - `NAME[?] -> NAME_?` 변경 대상: 0

### logic_scan_top `_숫자` port 확인

- 대상:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/rtl/top/logic_scan_top.v`
- 결과:
  - total ports parsed: 868 또는 869 수준으로 세션별 파서 차이가 있었음
  - `_숫자` 형태 port: 56개
  - 모두 input port
  - base signal 14개 각각 `[0]`, `[820]`, `[821]`, `[1641]` tap 형태
- 대표 매핑:
  - `LV_EM_0 -> LV_EM[0]`
  - `LV_EM_820 -> LV_EM[820]`
  - `LV_UMV_GR_1641 -> LV_UMV_GR[1641]`

### logic_scan_top pass-through wrapper 재생성

- 작업 경로:
  - `/home/tim/pjt_work/D2ID001A/PJT_0629/source/rtl/top`
- 요청:
  - `logic_scan_top`을 `logic_top.sv`만 instance하고 `logic_top` port를 그대로 외부로 노출하는 wrapper로 수정
- 결과:
  - `logic_scan_top.v` backup 생성: `logic_scan_top.v.bak_20260702_200729`
  - wrapper port: 869
  - instance connection: 869
  - missing/duplicate wrapper ports: 0
  - missing/duplicate instance connections: 0
- 주의:
  - 전체 `iverilog` 검증은 include/parameter visibility 및 기존 하위 module 부재 때문에 완전 통과하지 못했다.

## 2. D2I1002 / logic_scan_top reset-low DMA hang 작업

주요 경로:

```text
/home/tim/pjt_work/D2I1002/top
```

관련 노트:

- [[logic_scan_top Reset-Low Workaround Rationale]]
- [[logic_scan_top Debug README]]
- [[logic_scan_top VROM Read clk_vout_s1 Domain Finding]]
- [[logic_scan_top Reset-Low Follow-up Execution Plan]]
- [[logic_scan_top RT Readback Checklist]]
- [[logic_scan_top FSM Debug Plan]]
- [[DMA EXE3 External Sequence Workaround]]
- [[DMA Hang IC Analysis]]

핵심 stop point:

```text
sys_fsm      = SYS_RST_VROM
nvm_fsm      = NVM_VROM_LOAD_REQ
vrom_ctl_fsm = VROM_CTL_DMA_LV2
vrom_lv2_fsm = VROM_UCS__9E
DMA slave    = EXE3 ACT
DMA master   = RADR0_DONE / RDAT0_BURST_SRT
```

즉시 증상:

```text
VROM 9E DMA completion is not returning.
```

핵심 결론:

- reset-low `800us~1.3ms` fail window는 reset-low 동안 내부 synchronous RTL counter가 돈 결과로 보기 어렵다.
- `TO_RESX_SPROUT` low 동안 OSC가 off이므로, duration memory는 IO/OSC/POR/pad/retention 같은 physical node 상태일 가능성이 높다.
- 최종 유력 구조는 reset release 이후 1회성 clock transient 또는 clock/mux/reset-mask availability sequencing이 `clk_vout_s1` domain의 OTP/VROM slave handshake를 깨뜨리고, timeout/recovery 부재로 VROM 9E DMA read가 영구 wait 상태에 빠지는 것이다.
- 사용자의 명시적 제약: workaround guidance에서 RTL 수정은 하지 않는다.

중요 RTL 경로:

```text
sys_fsm_nvm_vrom on clk_sys
-> DMA EXE3 on clk_axi_sys
-> DW_axi_a2x async bridge
-> axi4_vout_s1
-> otp_vrom_slv_top_new on clk_vout_s1
```

특이 포인트:

- `peri.v:807`의 `.clk_axi(clk_vout_s1)`와 `//need to fix` 주석이 설계 검토 포인트다.
- VROM SLPOUT write가 같은 slave/domain을 통과했으므로 `clk_vout_s1`이 처음부터 죽었다는 가설은 배제된다.
- in-band MIPI debug는 read/write domain 차이와 live state 노출 부족 때문에 사실상 한계에 도달했다.
- reset-low W/A는 trigger avoidance로 유효하며, fail window `800us~1.3ms`를 피하는 방향이다.

## 3. D2ID001A TB/tbench 구조 분석

기존 Obsidian 노트:

- [[D2ID001A tbench 구조 리뷰]]
- [[D2ID001A scenario 내부 구조 상세]]

주요 경로:

```text
/home/tim/pjt_work/D2ID001A/TB/tbench
```

핵심 구조:

```text
compile.py / sim_rtl option
-> SCENARIO_* define
-> tbench.f
-> tb_top.sv
-> host U_HOST
-> t_main()
-> t_main_wakeup()
-> t_scenario_main()
-> scenario별 *.svh
-> task_* 실행
```

## 4. D2ID001A xrun/Xcelium setup 관련 미완료 캡처

세션에서 사용자는 `module load xcelium` 기반으로 `xrun` simulation setup을 요청했다.

확인된 기본 filelist:

```text
/home/tim/pjt_work/D2ID001A/TB/tbench/tbench.f
/home/tim/pjt_work/D2ID001A/TB/tbench/tbench_pre.f
/home/tim/pjt_work/D2ID001A/TB/tbench/tbench_post.f
/home/tim/pjt_work/D2ID001A/TB/tbench/checker/checker.f
/home/tim/pjt_work/D2ID001A/TB/tbench/package/package.f
/home/tim/pjt_work/D2ID001A/TB/tbench/random_model/random_model.f
/home/tim/pjt_work/D2ID001A/TB/tbench/mipi_tx/mipi_tx.f
```

필요 환경:

```bash
export pjt_path=/home/tim/pjt_work/D2ID001A
module load xcelium
command -v xrun
xrun -version
```

미확인:

- wrapper script 실제 생성 여부
- 기본 scenario define
- DUT RTL filelist 포함 방식
- `/PROJECT/...` vendor FLASH model 경로 접근 가능 여부

## 5. Hermes / Telegram / local AI 환경 캡처

- Telegram gateway는 systemd user service로 동작 중임이 여러 차례 확인됨.
- home/allowed user id: `1708402580`
- bot: `@Sk4091bot` / `Timbot`
- 서버 사양 확인 기록:
  - Rocky Linux 8.10
  - RTX 3060 12GB
  - RAM 125GiB
  - Ollama 0.21.1
  - Hugging Face CLI 사용 가능
- browser tool은 Rocky 8.10 glibc 제약으로 실패한 적 있음:
  - `GLIBC_2.29 not found`
  - `GLIBC_2.30 not found`

## 6. 이번 Obsidian 정리에서 가져온 project-local docs

### D2ID001A PJT_0629 docs

- [[ATOP Connection Report]]
- [[SDC TOP Port Groups]]
- [[D2ID001A Compile Flow Analysis]]
- [[DIGITAL_TOP logic_scan Numeric Port Update]]
- [[logic_scan_top Numeric Ports in DIGITAL Instance]]
- [[logic_scan_top Numeric Ports vs DIGITAL_TOP]]
- [[logic_scan_top Underscore Number Ports]]
- [[D2ID001A Project Structure Analysis]]
- [[D2ID001A Register Map Path Analysis]]
- [[D2ID001A Reset Boot Flow Analysis]]
- [[SDC TOP Unconnected to DIGITAL_TOP Ports]]
- [[D2ID001A Top Hierarchy Analysis]]

### D2I1002 debug docs

- [[logic_scan_top Debug README]]
- [[logic_scan_top FSM Debug Plan]]
- [[logic_scan_top Possible Cause Analysis]]
- [[logic_scan_top Reset-Low Follow-up Execution Plan]]
- [[logic_scan_top Reset-Low IO OSC Discharge Hypothesis]]
- [[logic_scan_top RT Readback Checklist]]
- [[logic_scan_top Top10 Candidates Reset 800us 1p3ms]]
- [[logic_scan_top VROM Read clk_vout_s1 Domain Finding]]
- [[Claude DMA Debug Work Instruction]]
- [[Claude Reset-Low IO OSC PPT Work Instruction]]
- [[DMA EXE3 External Sequence Workaround]]
- [[DMA Hang IC Analysis]]

## Related

- [[Index]]
- [[Home]]
- [[D2ID001A Reference]]
- [[D2I1002 Reference]]
