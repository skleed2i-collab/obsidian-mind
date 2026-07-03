---
date: 2026-07-03
description: "Imported Hermes/project Markdown document from June/July work: Claude Reset-Low IO OSC PPT Work Instruction"
tags:
  - reference
  - rtl
  - d2i1002
  - logic-scan-top
  - hermes-import
status: active
quarter: Q3-2026
project: D2I1002
source_path: /home/tim/pjt_work/D2I1002/top/claude_reset_low_io_osc_ppt_work_instruction.md
imported_at: 2026-07-03 10:19:28 KST
---

# Claude Work Instruction: reset-low IO/OSC discharge hypothesis justification and PPT creation

## Role

You are Claude acting as a senior digital/AMS bring-up debug engineer and customer-facing technical presentation author.
Your task is to review the existing logic_scan_top reset-low analysis, justify why RTL cannot/should not be changed for this issue, and create a PPT deck explaining a practical workaround (W/A) and why that workaround is technically valid for the customer.

Hard constraint:
- RTL cannot be modified.
- Do not propose RTL timeout/retry/FSM/CDC/clock/reset changes.
- The deliverable is customer-facing material to justify a workaround, not an internal RTL fix proposal.

The goal is to show:
1. the observed failure is consistent with reset-low IO/OSC discharge/restore timing,
2. the final digital stop point is VROM/DMA/AXI read wait,
3. since RTL has no safe field-change path, the valid mitigation is to avoid the bad reset-low/startup condition externally,
4. the W/A is technically reasonable because it controls the physical startup conditions before allowing normal operation.

## Working directory

```bash
cd /home/tim/pjt_work/D2I1002/top
```

## Important source documents

Read these first:

```text
docs/plans/README_logic_scan_top_debug.md
docs/plans/logic_scan_top_reset_low_io_osc_discharge_hypothesis.md
docs/plans/logic_scan_top_reset_low_followup_execution_plan.md
docs/plans/logic_scan_top_top10_candidates_reset_800us_1p3ms.md
docs/plans/logic_scan_top_possible_cause_analysis.md
docs/plans/logic_scan_top_fsm_debug_plan.md
docs/plans/logic_scan_top_rt_readback_checklist.md
```

Relevant RTL evidence to inspect if needed:

```text
source/rtl/peri/sys/sys_clk.v
source/rtl/peri/sys/sys_clk_osc_overshoot_protect.v
source/rtl/peri/sys/sys_reset.v
```

Already established context:

```text
Top: logic_scan_top
Symptom window: reset-low 800us~1.3ms only
Temperature note: hot/cold do not reproduce; mid/room condition may reproduce
OSC note: TO_RESX_SPROUT low turns OSC off
OSC startup: IP_OSC_BIAS_EN/IP_OSC_CLK_EN to actual TO_OSC_CLK output can take about 3~5us
Pad note: *_Y pad-derived inputs may settle with several-ns skew around reset release
Clock clarification: clk_sys and clk_axi_sys/clk_axi_sys_lp share the same clock root
Current digital stop point: VROM_UCS__9E -> DMA EXE3 -> AXI read-data wait
```

## Core analysis and W/A justification to present

The central customer-facing message is:

```text
The reset-low dependency is more likely an IO/OSC discharge and relative restore-timing problem than a same-root clk_sys/clk_axi_sys skew problem. Because RTL cannot be changed, the valid mitigation is to prevent entry into the bad reset-low/startup condition by controlling reset-low duration and startup sequencing externally.
```

Recommended W/A principle:

```text
Do not operate through the marginal reset-low window.
Guarantee reset-low is either shorter than the failing partial-discharge window or long enough to force a defined cold-start state, then allow enough OSC/pad/clock-settle guard time before normal access/operation.
```

Potential customer W/A options to explain, depending on product feasibility:

```text
Option A. Avoid reset-low 800us~1.3ms.
  - Keep reset-low below the failing window when doing a warm reset, or extend reset-low beyond the window for a cold/forced reset.
  - Preferred when reset controller/PMIC/FW can control reset pulse width.

Option B. If reset-low may fall in the window, force a recovery reset with a long reset-low duration.
  - If boot does not complete or expected status is not reached, assert reset again with a long-good duration.
  - Preferred when the system can detect boot-time failure or timeout externally.

Option C. Add startup guard time after reset release before issuing dependent commands or judging failure.
  - Wait for OSC/pad/clock settling guard time after TO_RESX_SPROUT release.
  - Poll available status/readback if accessible before starting normal operation.
  - This does not fix a transaction already stuck, but reduces false judgement and avoids premature access.

Option D. Use external bring-up sequence checks.
  - After reset release, check RT readback/status if available.
  - Confirm expected clock/OSC/mode states before proceeding.
  - On mismatch, perform long reset recovery rather than continuing.
```

Important wording:
- Present W/A as a reset/startup operation guideline, not an RTL fix.
- Do not claim the root cause is conclusively proven unless waveform evidence exists.
- Say the W/A is justified because it avoids the partial-discharge region that creates the marginal IO/OSC restore condition.
- State that the W/A acts before the DMA/AXI hang state is entered; once RTL is already waiting for a missing response, external reset/recovery is the safe escape.

Technical rationale for W/A:

```text
If the fail mechanism is a partial IO/OSC discharge state, then avoiding that reset-low duration removes the physical precondition for failure.
If a long reset fully discharges/initializes the relevant nodes, startup becomes cold-start-like and repeatable.
If a short reset preserves the previous-good state, startup remains warm-reset-like and repeatable.
The W/A therefore converts an ambiguous partial state into a controlled known state.
```

Justification chain:

1. During TO_RESX_SPROUT low, OSC is off.
   - Therefore the chip is not counting reset-low duration with normal internal synchronous RTL clocks.
   - The duration dependence must come from physical state that changes while reset is held low.

2. clk_sys and clk_axi_sys/clk_axi_sys_lp share the same clock root.
   - Do not claim independent root clock drift/skew.
   - Any clk_sys vs clk_axi_sys issue should be framed as downstream functional availability due to reset mask, divider, gate, or LP enable, not the primary time-memory source.

3. Reset-low length can be remembered by analog/pad/IO/OSC states.
   Examples:
   - IO pad capacitance and weak pull networks
   - pad input buffer threshold crossing
   - level shifter / Schmitt trigger internal nodes
   - OSC bias/startup/amplitude node
   - POR/reset analog delay node
   - leakage-dependent weakly held nodes

4. The 800us~1.3ms fail-only window is consistent with partial discharge / partial retention.
   Model:

```text
short reset-low:
  IO and OSC nodes remain short-reset-like
  -> pass

mid reset-low, 800us~1.3ms:
  one path is partially discharged and another partially retained
  -> worst reset-release combination
  -> fail

long reset-low:
  nodes are fully discharged or enter a defined cold-start state
  -> pass
```

5. Hot/cold pass with mid-temperature fail is also consistent.

```text
cold:
  lower leakage, nodes stay retained
  -> 950us behaves short-like
  -> pass

hot:
  higher leakage, nodes discharge more completely
  -> 950us behaves long-like
  -> pass

middle temperature:
  one node retained while another partially discharged, or both near threshold
  -> worst combination
  -> fail
```

6. The most important concrete interaction is IO_DIS_EXCLK_Y plus OSC restart.

RTL evidence from sys_clk.v:

```verilog
assign w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel;
```

Design comment:

```text
i_dis_exclk = 1 -> OSC selected
i_dis_exclk = 0 -> EXCLK selected
```

Potential mechanism:

```text
reset-low 800us~1.3ms
  -> IO_DIS_EXCLK_Y pad path is near threshold or late to settle
  -> reset release temporarily selects EXCLK path
  -> EXCLK may be absent/invalid
  -> clk_sys first window is missing/glitchy/discontinuous
  -> OSC path later becomes selected and normal
  -> initial startup sequence has already diverged
  -> VROM/DMA/AXI reaches read-data wait
```

7. OSC discharge alone can also create the window.

```text
reset-low 800us~1.3ms
  -> OSC bias/startup node is in worst partial-discharge state
  -> delayed first edge, runt/missing pulses, duty distortion, unstable first cycles
  -> deglitch/overshoot-protect progression differs
  -> clk_sys startup behavior differs
  -> downstream VROM/DMA/AXI sequence diverges
```

8. AXI read wait is the final digital stop point, not necessarily the root cause.

Current chain:

```text
SYS_RST_VROM
  -> NVM_VROM_LOAD_REQ
  -> VROM_CTL_DMA_LV2
  -> VROM_UCS__9E
  -> DMA EXE3 active
  -> DMA master RADR0_DONE / RDAT0_BURST_SRT
  -> waiting for AXI R channel completion
```

Interpretation:

```text
AXI read-data wait is the endpoint where the system gets stuck.
The root trigger may be earlier reset-release IO/OSC startup behavior that caused the transaction to occur under a bad source/target readiness condition.
```

## Required PPT deliverable

Create a customer-facing PowerPoint deck under:

```text
docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx
```

Also create a short markdown summary under:

```text
docs/plans/logic_scan_top_reset_low_workaround_customer_explanation_notes.md
```

The PPT should be 8~12 slides.
Use a technical executive style: clear, structured, evidence-driven, and customer-friendly.
The deck is not just a hypothesis deck; it must justify the W/A and explain why it is technically reasonable when RTL cannot be modified.
Avoid dense text-only slides.
Use diagrams, timelines, comparison tables, and signal-flow blocks.

## Recommended slide outline

### Slide 1. Title / Customer-facing key message

Title:
```text
logic_scan_top Reset-Low Window Issue: Workaround Rationale
```

Key message:
```text
Because RTL cannot be modified, the practical mitigation is to avoid the marginal reset-low/startup condition and force a known-good startup state.
```

Show as a simple chain:
```text
Observed window -> Physical startup hypothesis -> W/A principle -> Customer operation guideline
```

### Slide 2. Observed symptom and current boundary

Include:
- Failure appears only for reset-low 800us~1.3ms.
- Hot/cold conditions do not reproduce.
- Current digital stop point is VROM/DMA/AXI read-data wait.

Diagram:
```text
SYS_RST_VROM -> VROM_UCS__9E -> DMA EXE3 -> RADR0_DONE/RDAT0_BURST_SRT
```

Customer-facing conclusion:
```text
The observed stop point is AXI read wait, but the reset-low and temperature selectivity point to an upstream startup condition.
```

### Slide 3. Why RTL change is not the proposed path

Purpose: explicitly frame the constraint.

Content:
```text
RTL cannot be modified for this customer response.
Therefore the mitigation must act before the system enters the no-completion wait state.
```

Use two columns:
- Not available: RTL timeout, retry, FSM change, CDC change, clock/reset RTL change
- Available: reset pulse-width control, startup guard time, status polling, external recovery reset

Key phrase:
```text
The W/A prevents entry into the marginal condition; it is not trying to repair the FSM after it is already stuck.
```

### Slide 4. Corrected technical framing

Purpose: avoid an incorrect clock-root argument.

Include:
```text
clk_sys and clk_axi_sys/clk_axi_sys_lp share the same root.
The reset-low dependency should not be explained as independent clock-root skew/drift.
```

Better framing:
```text
reset-low duration is remembered by IO/OSC analog states, then appears downstream as startup/availability sequencing.
```

### Slide 5. Why reset-low duration points to IO/OSC discharge

Show a conceptual discharge/retention curve.

Content:
```text
OSC is off during TO_RESX_SPROUT low.
No internal synchronous counter is measuring 800us~1.3ms.
The duration must be stored in slowly changing physical states.
```

List physical states:
- IO pad capacitance / threshold / weak pull
- OSC bias/startup node
- OSC output threshold / first-cycle quality
- POR/reset analog delay node

### Slide 6. Short / mid / long reset-low model and W/A target

Use three-column comparison:

```text
Short reset-low: retained warm-reset-like state -> pass candidate
Mid reset-low: partial discharge mismatch -> fail window to avoid
Long reset-low: defined cold-start-like state -> pass candidate
```

W/A message:
```text
Choose a controlled reset-low duration outside the marginal mid-window.
```

Visual: three vertical cards with the middle card highlighted as “Do Not Operate Here”.

### Slide 7. Temperature behavior supports W/A logic

Explain hot/cold pass and mid-temperature fail.

Use table:

```text
Cold: lower leakage -> short-like retained state -> pass
Hot: higher leakage -> long-like discharged state -> pass
Mid: mixed retained/discharged states -> fail
```

Customer-facing phrase:
```text
The W/A converts an ambiguous partial-discharge startup into a controlled known startup state.
```

### Slide 8. Concrete mechanism: IO_DIS_EXCLK_Y + OSC restart

Must include RTL snippet:

```verilog
assign w_sys_exclk_sel = (~i_dis_exclk) ^ rm_sys_exclk_sel;
```

And design comment:
```text
i_dis_exclk = 1: OSC, 0: EXCLK
```

Show mechanism:
```text
IO_DIS_EXCLK_Y late/threshold
  -> transient EXCLK select or unstable clock-source selection
  -> invalid first clk_sys window
  -> later OSC recovery
  -> startup sequence already diverged
```

Tie to W/A:
```text
Avoiding the partial-discharge reset-low window reduces the probability of this transient startup combination.
```

### Slide 9. Concrete mechanism: OSC restart / deglitch path

Show chain:

```text
IP_OSC_CLK_EN -> TO_OSC_CLK first edge -> first-cycle quality -> deglitch mask -> clk_sys
```

Include RTL-derived point:
```text
r_clk_i_osc_glitch_mask shifts on div8; at 60MHz, 9 div8 shifts are about 1.2us plus edge alignment.
```

Tie to W/A:
```text
A controlled short-good or long-good reset avoids the marginal OSC partial-start condition and makes first-clock/deglitch behavior more repeatable.
```

### Slide 10. Recommended W/A options

Show as ranked options:

```text
W/A-1: Avoid reset-low 800us~1.3ms.
  - Use reset-low shorter than the fail window for warm reset, or longer than the fail window for cold/forced reset.

W/A-2: If boot/status timeout occurs, issue recovery reset with long reset-low.
  - Do not continue operation after detecting the stuck state.

W/A-3: Add startup guard time and status polling after reset release.
  - Wait for IO/OSC/clock settle before normal operation or before judging failure.

W/A-4: Use RT readback/status check when available.
  - If status indicates VROM/DMA wait or unexpected mode/clock state, recover by long reset.
```

Make clear:
```text
Exact reset limits must be set from characterization data; the known bad region is 800us~1.3ms.
```

### Slide 11. Why the W/A is technically valid

Use a cause/effect table:

```text
Observed property                    W/A implication
reset-low window only                avoid the window
hot/cold pass                        controlled state beats partial state
same-root clocks                     do not chase root skew; control startup precondition
AXI read wait final stop             reset/recover before or after timeout, do not rely on RTL self-timeout
```

Key statement:
```text
The W/A acts on the physical precondition of the failure, before the RTL reaches the no-completion wait state.
```

### Slide 12. Customer conclusion / action request

Conclusion bullets:
- RTL modification is not available.
- Reset-low 800us~1.3ms should be treated as a restricted operating window.
- Customer/PMIC/FW should force reset-low outside the bad window and add startup guard/status check where possible.
- If failure is detected, recover with a long reset rather than continuing.

One-line final:
```text
Avoid the 800us~1.3ms reset-low window and use controlled reset/startup sequencing because RTL cannot be modified and the W/A prevents entry into the marginal IO/OSC startup condition.
```

## PPT design requirements

Use a professional technical style.
Suggested theme:
- dark navy title/conclusion slides
- light background content slides
- accent color: amber/orange for fail window, teal/green for pass cases

Recommended visual motifs:
- signal-flow arrows
- cards for short/mid/long and cold/mid/hot comparisons
- small RTL snippet callout boxes
- timeline markers for key timestamps

Avoid:
- too many raw bullets
- claiming proven root cause
- implying clk_sys and clk_axi_sys are independent roots
- saying AXI is root cause without evidence
- any RTL modification proposal

Use careful wording:
- “working hypothesis”
- “consistent with”
- “to be proven by waveform comparison”
- “immediate digital stop point”
- “physical time-memory source”

Do not overstate:
- Do not say IO_DIS_EXCLK_Y is confirmed.
- Do not say OSC is confirmed.
- Do not say AXI is not involved.

Correct framing:
```text
AXI read-data wait is the observed digital endpoint.
The reset-low and temperature selectivity make an upstream IO/OSC discharge/restore timing trigger more plausible.
```

## Required validation before final response

After creating the PPT:

1. Verify the file exists:
```bash
ls -l docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx
```

2. Extract text and check no placeholder remains:
```bash
python -m markitdown docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx > /tmp/reset_low_wa_ppt_text.md
python - <<'PY'
from pathlib import Path
text = Path('/tmp/reset_low_wa_ppt_text.md').read_text(errors='ignore')
for bad in ['lorem', 'ipsum', 'TODO', 'XXXX', 'placeholder']:
    if bad.lower() in text.lower():
        print('FOUND_PLACEHOLDER', bad)
PY
```

3. If LibreOffice is available, render to PDF/images and inspect visually:
```bash
soffice --headless --convert-to pdf --outdir /tmp docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx
pdftoppm -jpeg -r 150 /tmp/logic_scan_top_reset_low_workaround_customer_explanation.pdf /tmp/reset_low_wa_slide
```

4. Check for:
- no overlapping text
- no clipped text
- readable RTL snippet
- clear diagrams
- no unsupported claims

## Final response expected from Claude

Report:

```text
Created:
- docs/plans/logic_scan_top_reset_low_workaround_customer_explanation.pptx
- docs/plans/logic_scan_top_reset_low_workaround_customer_explanation_notes.md

Validated:
- text extraction / placeholder check
- visual render check if available

Key message:
- RTL cannot be modified; W/A controls reset-low duration/startup sequencing to avoid the marginal IO/OSC partial-discharge condition before VROM/DMA/AXI wait is entered
```

## Related

- [[D2I1002 Reference]]
- [[Hermes June July 2026 Work Capture]]
- [[Index]]
