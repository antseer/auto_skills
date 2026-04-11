---
name: auto-req-rubric
description: Tool skill for auto_req pipeline. Derives and maintains skill_bank/rubric.md from seed_skills/ examples + user feedback. Run when: initializing for the first time, after adding new seed skills, or when promoted skills receive user feedback. Triggers: "update rubric", "initialize rubric", "rubric needs update", "seed skills changed".
---

# auto-req-rubric: Rubric Derivation and Maintenance

Derive and maintain `skill_bank/rubric.md` — the scoring criteria used by `/auto-req-evaluate` to filter requirements and skill drafts. The rubric is learned from real exemplars, not hardcoded rules.

## Setup

Check what mode to run in:
- If `skill_bank/rubric.md` does not exist → **initialize mode**
- If user mentions feedback on a specific promoted skill → **feedback mode**
- If new files were added to `seed_skills/` → **re-derive mode**

## Initialize Mode

Run this when rubric.md does not yet exist.

### Step A: Check for seed skills

List all subdirectories in `seed_skills/`. Each subdirectory is one skill — look for `SKILL.md` inside each (e.g., `seed_skills/airdrop-hunter-pro/SKILL.md`). Also read any supporting files in the same folder (`_meta.json`, `references/`, scripts) for additional context.

If no subdirectories with `SKILL.md` exist, print:

```
No seed skills found in seed_skills/.
Add exemplary skill folders (each containing SKILL.md) to seed_skills/ then re-run /auto-req-rubric.
Generating a minimal starter rubric instead.
```

Then write a minimal starter `skill_bank/rubric.md`:

```markdown
---
version: 1.0
updated: YYYY-MM-DD
seed_count: 0
note: Starter rubric — no seeds provided. Add seed skills and re-run /auto-req-rubric to improve.
---

## Requirement Scoring Criteria

Rate each candidate requirement on these 4 binary checks (yes=1, no=0):

1. **Specificity**: Can you draw a clear boundary around what this skill does vs. does not do? A requirement that could mean 5 different things scores 0.
2. **Novelty**: Is this NOT already covered by any skill in skill_bank/promoted/? If a very similar skill already exists, score 0.
3. **Resonance**: Did 2 or more personas independently surface this desire? Score 1 if yes.
4. **Actionability**: Is there enough concrete detail to write a skill without more research? Score 1 if yes.

Pass threshold: 3/4 or higher.
Low score (1-2): generate improvement suggestion, save to candidates/ with -feedback.md suffix.
Zero score: discard, log reason.

## Skill Quality Criteria

Rate each generated skill draft on these 5 binary checks:

1. **Completeness**: Are all steps present with no obvious gaps? Would an agent running this know what to do at every point?
2. **Executability**: Can Claude Code execute this skill without needing to ask clarifying questions mid-run?
3. **Maintainability**: Is the skill scoped tightly enough that future edits won't cascade unexpectedly?
4. **Cost-Awareness**: Does the skill avoid unnecessary LLM calls, large context loads, or redundant subagent dispatches?
5. **Safety**: Does the skill avoid irreversible operations without explicit user confirmation?

Pass threshold: 4/5 or higher.
Failed dimensions get written as improvement notes in candidates/<name>-feedback.md.

## Version Log

- v1.0 (YYYY-MM-DD): Starter rubric, no seeds
```

Skip to Done.

### Step B: Analyze seed skills (if seeds exist)

For each subdirectory in `seed_skills/`, read `SKILL.md` plus any supporting files (`_meta.json`, `references/` contents, scripts). Dispatch a single subagent with this prompt (replace [SEED_SKILLS] with full contents of all SKILL.md files and their supporting files, labeled by folder name):

```
You are analyzing exemplary Claude Code skill files to extract what makes a skill high quality.

SEED SKILLS:
[SEED_SKILLS]

Analyze these skills and identify:

1. STRUCTURAL patterns: What structural elements do all/most good skills share? (e.g., clear trigger description, step-by-step instructions, example outputs, anti-patterns section)

2. CONTENT patterns: What content qualities distinguish these skills? (e.g., atomic steps, concrete examples, explicit scope limits, clear output format)

3. FAILURE modes to avoid: Based on what's present in these good examples, what would a bad skill look like? List 4-6 specific anti-patterns.

4. REQUIREMENT signals: Based on these skills, what properties would a good skill requirement have? What would make you confident a requirement could become a skill like these?

Output a JSON object (no other text):
{
  "structural_patterns": ["pattern 1", "pattern 2", ...],
  "content_patterns": ["pattern 1", "pattern 2", ...],
  "anti_patterns": ["anti-pattern 1", ...],
  "requirement_signals": ["signal 1", "signal 2", ...],
  "requirement_scoring_criteria": [
    {"name": "criterion name", "question": "binary yes/no question", "pass_condition": "what yes looks like"},
    ...
  ],
  "skill_quality_criteria": [
    {"name": "criterion name", "question": "binary yes/no question", "pass_condition": "what yes looks like"},
    ...
  ]
}
```

### Step C: Write rubric.md from analysis

Using the subagent output, write `skill_bank/rubric.md`:

```markdown
---
version: 1.0
updated: YYYY-MM-DD
seed_count: N
---

## Requirement Scoring Criteria

Rate each candidate requirement on these binary checks (yes=1, no=0):

[For each item in requirement_scoring_criteria from subagent output:]
N. **[name]**: [question]
   Pass: [pass_condition]

Pass threshold: [ceil(N * 0.75)] or higher.
Low score: generate improvement suggestion → candidates/<name>-feedback.md
Zero score: discard, log reason.

## Skill Quality Criteria

Rate each generated skill draft on these binary checks:

[For each item in skill_quality_criteria from subagent output:]
N. **[name]**: [question]
   Pass: [pass_condition]

Pass threshold: [ceil(N * 0.8)] or higher.
Failed dimensions → improvement notes in candidates/<name>-feedback.md.

## Anti-Patterns to Flag

[anti_patterns from subagent output as bullet list]

## Version Log

- v1.0 (YYYY-MM-DD): Initial rubric derived from N seed skills
```

Print: "rubric.md initialized (v1.0) from N seed skills → skill_bank/rubric.md"

## Feedback Mode

Run this when user provides feedback on a specific promoted skill (e.g., "this skill scored high but the output was poor").

Ask user for:
1. Which skill? (path to SKILL.md in promoted/)
2. What was wrong? (free text)
3. Did it score too high or too low on any specific criterion?

Dispatch a subagent with the rubric, the skill, and the feedback:

```
The current rubric scored this skill highly, but user feedback says it has problems.

CURRENT RUBRIC:
[rubric.md contents]

SKILL THAT SCORED HIGH BUT DISAPPOINTED:
[skill contents]

USER FEEDBACK:
[feedback text]

Identify: Which rubric criterion failed to catch this problem? Propose a specific revision to that criterion's question or pass condition. Output:
{
  "failing_criterion": "name of criterion",
  "why_it_failed": "explanation",
  "revised_criterion": {"name": "...", "question": "...", "pass_condition": "..."},
  "version_note": "one sentence describing this change"
}
```

Update the relevant criterion in rubric.md. Bump version (1.0 → 1.1). Append to version log.

Print: "rubric.md updated (vX.Y) based on feedback → skill_bank/rubric.md"

## Re-derive Mode

Run this when new seed skills were added.

Re-run Steps B and C. Merge new patterns with existing rubric (keep criteria that were added via feedback, add new criteria from seeds, remove criteria that no longer appear in any seed). Bump major version (1.x → 2.0).

## Done

Print:
```
=== auto-req-rubric complete ===
Mode:         [initialize / feedback / re-derive]
Rubric path:  skill_bank/rubric.md
Version:      vX.Y
Seed count:   N
```
