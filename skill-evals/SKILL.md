---
name: skill-evals
description: "Evaluate whether a skill works reliably using the OpenAI eval-skills workflow: define outcome/process/style/efficiency goals, run a small prompt set through codex exec, grade traces with deterministic checks, and return a concise scorecard. Use when validating a skill before iteration or release."
---

# Skill Evals

Run this workflow to evaluate a target skill in a repeatable, low-overhead way.

Keep the evaluation automated and deterministic first.

## Step 1: Define Goals and Pass Criteria

Inspect the target skill files first (`SKILL.md`, scripts, references, assets) and infer candidate goals.

Ask the user to validate or edit the inferred goals before running evals.

Before proceeding, present a draft success definition in chat using this structure:

- Outcome goals.
- Process goals.
- Style goals.
- Efficiency goals.
- Pass threshold and hard-fail checks.

Then ask for explicit confirmation. Do not run evals until the user confirms or edits this draft.

Capture the approved goals in `.agents/evals/<skill-name>/contract.v1.json`:

- Outcome goals: define expected end results and required artifacts.
- Process goals: define required and forbidden tool/command patterns that can be checked in trace logs.
- Style goals: define expected format and quality criteria for deliverables.
- Efficiency goals: define command budget, retry limits, and avoidable-action penalties.

Require explicit values for:

- Goal weights (default: outcome 0.40, process 0.30, style 0.20, efficiency 0.10).
- Pass threshold (default: 0.80).
- Hard-fail checks that always block PASS.

Record confirmation status and timestamp in the contract.

Read `references/eval-spec.md` for required fields and examples.

## Step 2: Build a Small Prompt Set

Create or update `.agents/evals/<skill-name>/prompts.csv` with 10 to 20 prompts.

Use the exact columns:

`id,should_trigger,prompt`

Include:

- Direct success prompts.
- Edge cases.
- Negative controls (`should_trigger=false`).
- At least a few prompts that test style and efficiency expectations.

Add failed cases from previous runs as regressions.

Before executing prompts, show the draft prompt set in chat as a compact table:

- `id`
- `should_trigger`
- short prompt preview

Ask the user to confirm or edit this set (add/remove/change cases). Do not run prompt execution until confirmed.

## Step 3: Run Prompts and Capture Traces

For each prompt, run `codex exec --json` and save one trace file:

`.agents/evals/<skill-name>/artifacts/<run-id>/<id>.jsonl`

If you need final plain-text output for a style check, also save:

`.agents/evals/<skill-name>/artifacts/<run-id>/<id>.last.txt`

Keep traces as the source of truth. Do not generate extra logs unless the user asks for debug mode.

Only execute this step after both confirmations are complete:

1. Success-definition confirmation.
2. Prompt-set confirmation.

## Step 4: Grade with Deterministic Checks First

Use deterministic checks for objective correctness:

- Required files/artifacts exist.
- Builds/tests/renders complete.
- No forbidden errors.
- Required process/tool steps appear in traces.
- Negative controls do not trigger the skill.
- Efficiency counters stay within contract thresholds.

Use trace events (`command_execution`, final message, and artifact outputs) to score process and efficiency.

## Step 5: Run Optional Structured Style Check

When style is hard to grade deterministically, run a lightweight model check with structured output:

- Define a small JSON schema for style criteria in `style-schema.v1.json`.
- Score style against that schema and save one result per case in:
  - `.agents/evals/<skill-name>/artifacts/<run-id>/<id>.style.json`

Do not add human review in default workflow.

## Step 6: Compute Final Scores and Verdict

Evaluate hard-fail checks first. If any hard-fail check fails, set verdict to `FAIL`.

Otherwise compute weighted score by dimension:

`final_score = sum(weight_dimension * dimension_score)`

Persist only concise outputs:

- `results.latest.json` (canonical per-run output).
- `results.history.csv` (one row per run).

Do not create `report.md`, `summary.csv`, `command-log.txt`, or run subfolders beyond `artifacts/<run-id>/` unless the user explicitly requests detailed debug artifacts.

## Step 7: Present a Concise Scorecard

Always return a compact summary in chat:

- Overall verdict and score.
- Pass rate.
- Top 3 failing cases with one-line reason and evidence path.
- One short "what to fix next" list.

Optionally include a tiny ASCII trend line from `results.history.csv` if multiple runs exist.

## Step 8: Iterate

After each run:

- Add regressions to `prompts.csv`.
- Tighten deterministic checks.
- Keep the eval set small and high-signal.
