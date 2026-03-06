# Eval Spec

Use this reference when creating or updating automated skill evaluation data.

## Required Directory Layout

```text
.agents/evals/<skill-name>/
  contract.v1.json
  prompts.csv
  checks.v1.json
  style-schema.v1.json            # optional
  results.latest.json
  results.history.csv
  artifacts/
    <run-id>/
      <id>.jsonl
      <id>.last.txt               # optional
      <id>.style.json             # optional
```

Use ISO-like run IDs, for example:

`2026-03-06T14-25-00Z`

## `contract.v1.json` Shape

```json
{
  "schema_version": "1.0",
  "skill_name": "my-skill",
  "updated_at": "2026-03-06T14:00:00Z",
  "weights": {
    "outcome": 0.4,
    "process": 0.3,
    "style": 0.2,
    "efficiency": 0.1
  },
  "pass_threshold": 0.8,
  "hard_fail_checks": [
    "required_artifact_missing",
    "forbidden_command_used"
  ],
  "user_validation": {
    "goals_confirmed": true,
    "goals_confirmed_at": "2026-03-06T14:05:00Z",
    "prompts_confirmed": true,
    "prompts_confirmed_at": "2026-03-06T14:08:00Z",
    "notes": "Added two negative-control prompts."
  },
  "goals": {
    "outcome": [
      "Produce all required artifacts without errors."
    ],
    "process": [
      "Use expected tools and avoid forbidden actions."
    ],
    "style": [
      "Return output in required format and tone."
    ],
    "efficiency": [
      "Avoid unnecessary commands and retries."
    ]
  },
  "efficiency_policy": {
    "max_commands_per_case": 15,
    "max_retries_per_case": 2
  }
}
```

The evaluation run must not start until both are true:

- `user_validation.goals_confirmed`
- `user_validation.prompts_confirmed`

## `prompts.csv` Columns

Use these columns exactly (OpenAI eval-skills pattern):

`id,should_trigger,prompt`

Column guidance:

- `id`: unique and stable test-case identifier.
- `should_trigger`: `true` or `false`.
- `prompt`: user prompt to run with `codex exec`.

## `checks.v1.json` Shape

```json
{
  "required_actions": [
    "read_skill_file",
    "emit_structured_scorecard"
  ],
  "forbidden_actions": [
    "git reset --hard",
    "rm -rf"
  ],
  "required_artifacts": [
    "results.latest.json"
  ],
  "efficiency": {
    "max_commands_per_case": 15,
    "max_retries_per_case": 2
  }
}
```

## `results.latest.json` Canonical Shape

```json
{
  "schema_version": "1.0",
  "run_id": "2026-03-06T15:10:45Z",
  "skill_name": "my-skill",
  "pass_threshold": 0.8,
  "summary": {
    "final_score": 0.82,
    "verdict": "PASS",
    "hard_fail_count": 0,
    "total_cases": 12,
    "passed_cases": 10,
    "failed_cases": 2
  },
  "validation": {
    "goals_confirmed": true,
    "prompts_confirmed": true
  },
  "cases": [
    {
      "id": "test_01",
      "should_trigger": true,
      "skill_invoked": true,
      "dimension_scores": {
        "outcome": 1.0,
        "process": 0.8,
        "style": 0.7,
        "efficiency": 0.6
      },
      "weighted_score": 0.84,
      "checks": [
        {
          "id": "render_succeeded",
          "type": "deterministic",
          "status": "pass",
          "score": 1.0,
          "hard_fail": true,
          "evidence": "artifacts/2026-03-06T15-10-45Z/test_01.jsonl"
        }
      ],
      "efficiency_metrics": {
        "command_count": 7,
        "retry_count": 1,
        "token_count": 1900
      },
      "final": {
        "score": 0.84,
        "verdict": "PASS"
      }
    }
  ]
}
```

## `results.history.csv` Columns

Use one row per run:

`run_id,skill_name,final_score,verdict,hard_fail_count,total_cases,passed_cases,failed_cases,avg_command_count,avg_retry_count`

## Scoring Rules

Apply this decision order:

1. Confirm user validation gates are complete.
2. Evaluate hard-fail checks.
3. If any hard-fail fails, set `verdict=FAIL`.
4. If no hard-fail fails, compute weighted dimension score.
5. If score meets threshold, set `verdict=PASS`; otherwise `FAIL`.

## Evidence Requirements

Require each scored check to reference evidence from run artifacts:

- Trace path (`artifacts/<run-id>/<id>.jsonl`).
- Log line or error signature when relevant.

Reject final scoring if evidence pointers are missing.

## Output UX Requirements

When presenting results to the user:

- Prefer a concise in-chat scorecard over multiple files.
- Show overall verdict, score, pass rate, and top 3 failures.
- Include one evidence path per failure.
- Use optional ASCII trend from `results.history.csv` when available.
