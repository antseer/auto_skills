# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

`auto_req` is a self-improving five-stage pipeline that mines Claude Code skill requirements from real X/Twitter financial/crypto discussions. It uses persistent BDI (Belief-Desire-Intention) personas, seed-derived quality scoring, and a replay-driven skill evolution loop to continuously improve both the requirements it surfaces and the skills it generates.

**Pipeline stages:**
1. `/auto-req-collect` — fetch top-10 hot finance/crypto topics via `ant_market_sentiment` MCP, collect posts, filter for relevance, stratified user sampling with behavior tags
2. `/auto-req-persona` — cluster behavior tags from posts → generate 3 temporary personas → select 2 permanent personas → output 5-person roster to `persona_bank/roster_YYYY-MM-DD.json`
3. `/auto-req-simulate` — run adaptive multi-round discussion with dynamic roster (BDI-aware, Round 4 triggers when permanent persona desire resonance ≥ 3)
4. `/auto-req-evaluate` — two-layer quality gate: score requirements via rubric, generate skill drafts (with embedded user personas + usage scenarios), score drafts
5. `/auto-req-evolve` — replay-driven mutation loop: improve promoted skills using historical discussion history

**Tool skill:**
- `/auto-req-rubric` — derive and maintain `skill_bank/rubric.md` from `seed_skills/` examples + user feedback

## Running the Pipeline

Each stage is a Claude Code skill, triggered manually in sequence:

```
/auto-req-collect     # Stage 1: collect + filter posts, stratified user sampling + behavior tags
/auto-req-persona     # Stage 2: cluster tags → 3 temp personas + 2 permanent → 5-person roster
/auto-req-simulate    # Stage 3: run adaptive discussion with dynamic roster
/auto-req-evaluate    # Stage 4: score requirements + generate + score skill drafts
/auto-req-evolve      # Stage 5: replay-driven skill improvement (run periodically)
/auto-req-rubric      # Tool: maintain skill_bank/rubric.md from seeds + feedback
```

**First run:** populate `seed_skills/` with exemplary SKILL.md files, then run `/auto-req-rubric` before Stage 4.

Each stage can be re-run independently. Files from the previous stage are read from disk at the start of the next stage.

## Directory Layout

```
raw_posts/YYYY-MM-DD.json       # Stage 1a: collected X posts (all)
raw_posts/YYYY-MM-DD-filtered.json  # Stage 1b: relevance-filtered posts
raw_users/YYYY-MM-DD.json       # Stage 1c: user profiles (old format, backward compat)
raw_users/YYYY-MM-DD-tagged.json    # Stage 1c: stratified users with behavior tags
personas/YYYY-MM-DD.json        # Stage 1: legacy static personas (superseded by persona_bank/)

persona_bank/
  alice.json                    # BDI state: DeFi developer (permanent, cross-run)
  bob.json                      # BDI state: retail trader (permanent)
  carol.json                    # BDI state: full-stack SaaS dev (permanent)
  dave.json                     # BDI state: beginner (permanent)
  temp_cluster_N_YYYY-MM-DD.json  # Temporary persona (auto-cleaned after 7 days)
  roster_YYYY-MM-DD.json        # Today's active 5-person roster (2 permanent + 3 temporary)

discussions/YYYY-MM-DD/
  round1.json                   # Stage 3: first persona reactions (BDI-aware)
  round2.json                   # Stage 3: cross-reactions
  round3.json                   # Stage 3: facilitator themes + probe responses
  round4.json                   # Stage 3: deep probe (only when resonance ≥ 3)

seed_skills/                    # User-provided exemplary skill folders (each contains SKILL.md + supporting files)

skill_bank/
  rubric.md                     # Scoring criteria derived from seeds (versioned)
  candidates/                   # Low-scoring requirements + skill drafts with feedback
  promoted/                     # Skills that passed both quality layers
  CHANGELOG.md                  # Full skill evolution history

test_fixtures/                  # Offline testing: sample posts, users, personas
.claude/skills/                 # Local skill definitions for this project
```

## Offline Testing

If today's `raw_posts/` or `personas/` files are missing, the simulate and output stages automatically fall back to `test_fixtures/`:
- `test_fixtures/raw_posts_sample.json`
- `test_fixtures/personas_sample.json`
- `test_fixtures/raw_users_sample.json`

Use fixtures to test pipeline logic without needing a live MCP call.

## Persona Design Constraint

Personas must represent **ordinary Claude Code users** — not celebrities ranked by follower count.

**Permanent personas** (in `persona_bank/`) are hand-designed behavioral archetypes that accumulate BDI state across runs. They provide continuity and catch long-tail needs.

**Temporary personas** are generated each run from behavior tag clustering of real post data. They capture the day's specific audience composition and adapt automatically as topics shift.

The roster for each run is 2 permanent + 3 temporary, selected for maximum coverage diversity.

## MCP Dependency

Stage 1 requires the `ant_market_sentiment` MCP tool (configured in `.claude/settings.local.json`). All other stages use only the filesystem and Claude's Agent tool for subagent dispatch. If `ant_market_sentiment` is unavailable, use test fixtures.

## Skills in This Repo

`.claude/skills/` contains the full redesigned pipeline:

- `auto-req-collect.md` — Stage 1 (data collection + relevance filter + stratified user sampling + behavior tags)
- `auto-req-persona.md` — Stage 2 (dynamic persona roster: tag clustering + temp personas + permanent selection)
- `auto-req-simulate.md` — Stage 3 (adaptive discussion with dynamic roster, supports N personas)
- `auto-req-evaluate.md` — Stage 4 (two-layer requirement + skill quality gate)
- `auto-req-evolve.md` — Stage 5 (replay-driven skill evolution)
- `auto-req-rubric.md` — Tool (rubric derivation + feedback-driven updates)
- `auto-req-output.md` — Stage 3 legacy (superseded by auto-req-evaluate)
- `narrative-alignment-score.md` — crypto/macro narrative alignment scoring skill (output of a prior pipeline run)

`skills/drafts/` contains skill drafts generated by the pipeline, intended for review and potential promotion to `~/.claude/skills/`. Current drafts from 2026-04-06:
- `narrative-alignment-score.md` — scores how well a thesis aligns with macro/crypto narratives
- `signal-base-rates.md` — base rate calibration for market signals
- `alert-enrichment.md` — enriching trade alerts with market context

## Autoresearch Skill

`autoresearch-skill/SKILL.md` autonomously optimizes any Claude Code skill via prompt mutation loops (Karpathy-style autoresearch). It runs the target skill repeatedly, scores outputs against binary evals, mutates the prompt one change at a time, and keeps only improvements — producing an improved SKILL.md, a `results.tsv` score log, and a `changelog.md` of every mutation tried.

**How to use it:** Run `/autoresearch` (or invoke `autoresearch-skill/SKILL.md` directly) and provide:
1. Path to the target skill (e.g. `.claude/skills/auto-req-simulate.md`)
2. 3–5 test inputs covering different use cases
3. 3–6 binary yes/no eval criteria defining a good output
4. Optional: runs per experiment (default 5), budget cap (default: none)

The skill runs fully autonomously once started. It also generates a live HTML dashboard (`autoresearch-[skill-name]/dashboard.html`) showing score progression in real time.

Use this to improve pipeline skills when they produce inconsistent results.

## Research References

The following directories contain academic/research projects that inform this pipeline's design. They are not part of the pipeline itself.

- `AutoSkill/` — experience-driven lifelong skill learning via self-evolution (ECNU-ICALK, arXiv 2603.01145); see [`AutoSkill/CLAUDE.md`](AutoSkill/CLAUDE.md) for setup, architecture, and CLI usage
- `SkillCraft/` — benchmark for LLM agents learning to compose and cache tool-use skills (arXiv 2603.00718)
- `SkillNet/` — open infrastructure for creating, evaluating, and sharing AI agent skills (arXiv 2603.04448)
- `lifesim/` — LifeSim: long-horizon user life simulator for personalized assistant evaluation (BDI model); see [`lifesim/CLAUDE.md`](lifesim/CLAUDE.md) for setup, simulation commands, and evaluation pipeline
