# D2ID001A/D001 검증환경 개선 Agent Loop

작업 위치:

```text
/home/tim/pjt_work/D2ID001A/D001
```

목적:
- `tbench/tb_top.sv`를 TB top으로 보고, DUT는 `D2ID001A_TOP`으로 취급한다.
- 일반 scenario (`tbench/scenario/`)와 EDS 환경 (`tbench/eds_scen/`, `sim_vec/*/eds_run`, `analog_vector/perl/eds_*`)을 함께 분석한다.
- Claude Code와 Codex 두 agent 모두 같은 규칙으로 반복 개선 작업을 수행한다.

핵심 규칙:
- 기존 파일 수정 전 반드시 백업 helper를 실행한다.

```bash
./scripts/d2id_backup_for_edit.sh relative/path/to/file
```

이 helper는 원본을 `_back/<timestamp>/...`로 이동한 뒤 작업용 copy를 원래 위치에 복원한다.

주요 command:

```bash
cd /home/tim/pjt_work/D2ID001A/D001
python3 scripts/d2id_verif_env_audit.py

FOCUS=structure MAX_ITERS=1 ./scripts/d2id_claude_verif_loop.sh
FOCUS=structure MAX_ITERS=1 ./scripts/d2id_codex_verif_loop.sh

FOCUS=compile TOP_SCENARIO=TOP RUN_XRUN=1 MAX_ITERS=3 ./scripts/d2id_claude_verif_loop.sh
FOCUS=compile TOP_SCENARIO=TOP RUN_XRUN=1 MAX_ITERS=3 ./scripts/d2id_codex_verif_loop.sh

FOCUS=eds EDS_SCENARIO=reset RUN_XRUN=0 MAX_ITERS=1 ./scripts/d2id_claude_verif_loop.sh
FOCUS=eds EDS_SCENARIO=osc RUN_XRUN=0 MAX_ITERS=1 ./scripts/d2id_codex_verif_loop.sh
```

생성된 project 문서:

```text
/home/tim/pjt_work/D2ID001A/D001/doc/verif_env_agent_loop.md
```

생성된 핵심 script:

```text
scripts/d2id_backup_for_edit.sh
scripts/d2id_verif_env_audit.py
scripts/agent_loop_prompt.md
scripts/d2id_claude_verif_loop.sh
scripts/d2id_codex_verif_loop.sh
```
