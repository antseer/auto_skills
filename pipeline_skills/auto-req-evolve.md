---
name: auto-req-evolve
description: Stage 5 of redesigned auto_req pipeline. Replay-driven mutation loop for promoted skills. Builds replay pool from discussions/ history, establishes baseline, mutates one change at a time, keeps improvements. Updates persona_bank desire resonance when coverage improves. Triggers: "run evolve", "evolve skills", "improve promoted skills", "run evolution loop".
---

# auto-req-evolve: Replay-Driven Skill Evolution

Improve promoted skills by replaying historical discussions, measuring performance against auto-derived evals, and mutating one change at a time. Keeps only improvements.

## When to Run

Check preconditions before starting:
1. Count SKILL.md files in `skill_bank/promoted/` — need at least 3
2. Count subdirectories in `discussions/` — need at least 2

If either condition is unmet, print which condition failed and stop:
```
Cannot run evolution: need at least 3 promoted skills (found N) and 2 days of discussion history (found N).
Run more pipeline cycles first.
```

## Setup

List all SKILL.md files in `skill_bank/promoted/`. 

Ask the user: "Which skill would you like to evolve? (or press Enter to default to the oldest modified)"

If no response or Enter: use the skill with the oldest last-modified timestamp.

Read the target skill fully.

Read `skill_bank/rubric.md`.

Read all `persona_bank/*.json` — find which desires have `resolved_by` pointing to this skill.

## Step 1: Build Replay Pool

Scan all subdirectories of `discussions/` with YYYY-MM-DD names. For each, check if `round3.json` or `round4.json` exists.

Dispatch one relevance-check subagent per discussion folder (can run in parallel, batches of 5):

```
REPLAY POOL FILTER

TARGET SKILL NAME AND DESCRIPTION:
[skill name] — [skill description from frontmatter]

DISCUSSION DATA:
[round3.json contents + round4.json if exists]

Does this discussion contain scenarios where the target skill would be relevant?
Look for: mentions of the same pain point, similar desired tools, related workflows.

Output ONLY JSON:
{
  "relevant": true or false,
  "relevance_reason": "one sentence",
  "relevant_scenarios": [
    "brief description of specific scenario from this discussion that relates to the skill"
  ]
}
```

Collect all relevant scenarios from relevant discussions. Target 5-15 scenarios total.

If fewer than 3 relevant scenarios found, print:
```
Insufficient replay pool for [skill name] — only N relevant scenarios found.
Run more pipeline cycles to build discussion history, then retry.
```
And stop.

Print: "Replay pool: N scenarios from N discussions"

## Step 2: Derive Binary Evals

Dispatch a single subagent:

```
EVAL DERIVATION

Generate 4-5 binary yes/no eval criteria for testing a Claude Code skill.

TARGET SKILL:
[full skill content]

RUBRIC SKILL QUALITY CRITERIA:
[skill_bank/rubric.md — "Skill Quality Criteria" section]

REPLAY SCENARIOS (what the skill will be tested against):
[relevant_scenarios list]

Generate evals that are:
1. Specific to THIS skill (not generic)
2. Answerable yes/no from reading the skill content against a scenario
3. Covering different failure modes
4. Strict enough to catch bad outputs, not so strict they reject good ones

Output ONLY JSON:
{
  "evals": [
    {
      "name": "short name",
      "question": "yes/no question about whether the skill handles this scenario well",
      "pass_condition": "what yes looks like — be specific",
      "fail_condition": "what no looks like"
    }
  ]
}
```

## Step 3: Establish Baseline

Score the current promoted skill against each eval for each scenario in the replay pool.

For each (scenario, eval) pair, dispatch a scoring subagent:

```
SKILL EVAL SCORING

SCENARIO:
[scenario description]

SKILL CONTENT:
[full SKILL.md content]

EVAL:
Name: [eval name]
Question: [eval question]
Pass condition: [pass condition]
Fail condition: [fail condition]

Does this skill pass this eval for this scenario?
Output ONLY JSON: {"pass": true or false, "reason": "one sentence"}
```

Calculate:
- `baseline_score` = (total passes / total pairs) × 100, as a percentage
- Per-eval pass rates across all scenarios

Record in `skill_bank/CHANGELOG.md`:

```markdown
## Evolution: [skill name] — Experiment 0 (Baseline)

**Date:** YYYY-MM-DD
**Skill:** [skill filename]
**Score:** X% (N/M passes)
**Status:** baseline
**Replay pool:** N scenarios

**Per-eval breakdown:**
- [eval name]: X% (N/M)
- ...
```

Print: "Baseline established: X% (N/M passes)"

## Step 4: Mutation Loop

Run autonomously. Stop when: score ≥ 95% for 2 consecutive experiments, OR 10 experiments completed.

### Each Experiment:

**4a. Analyze failures**

Look at which evals have the lowest pass rates. Read the specific scenarios that failed. Find the pattern — is it a formatting issue? A missing instruction? An ambiguous step?

**4b. Form one hypothesis**

Choose ONE change. Good mutations:
- Add a specific instruction addressing the most common failure pattern
- Reword an ambiguous step to be more explicit
- Add a concrete example covering a failing scenario type
- Add an anti-pattern entry for a recurring mistake
- Remove a step that over-constrains the skill

Bad mutations: rewriting everything, changing multiple things at once, adding vague instructions like "be better".

**4c. Make the change**

Edit the skill content in memory (keep previous version to restore if needed).

**4d. Re-score against replay pool**

Run the same (scenario, eval) scoring as Step 3 with the mutated skill.

Calculate new score and per-eval breakdown.

**4e. Keep or discard**

- Score improved → **KEEP**: this is the new current version
- Score same or worse → **DISCARD**: restore previous version

**4f. Log to CHANGELOG.md**

```markdown
## Evolution: [skill name] — Experiment N ([keep/discard])

**Date:** YYYY-MM-DD
**Score:** X% → Y%
**Status:** [keep/discard]
**Change:** [one sentence describing what changed]
**Hypothesis:** [why this was expected to help]
**Failing evals before change:** [which evals were failing most]
**Result:** [what actually happened — which evals improved or didn't]
```

### After Loop Ends

Write the final (best) skill version back to `skill_bank/promoted/<skill-name>.md`.

Print: "Evolution complete: X% → Y% over N experiments (K kept, M discarded)"

## Step 5: Update Persona Desires

If final score > baseline score:

For each persona in `persona_bank/` where any desire has `resolved_by` matching this skill's path:
- Decrement `resonance` by 1 (minimum 0)
- If `resonance` reaches 0: the desire is well-served and will deprioritize naturally in future simulations

Write updated persona files to `persona_bank/`.

Print: "Updated N persona desires (resonance decremented)"

## Done

Print:
```
=== auto-req-evolve complete ===
Skill:          [skill filename]
Baseline score: X%
Final score:    Y%
Improvement:    +Z%
Experiments:    N total (K kept, M discarded)
Replay pool:    N scenarios from N discussions
Desires updated: N desires deprioritized in persona_bank/

Full evolution log: skill_bank/CHANGELOG.md
```
