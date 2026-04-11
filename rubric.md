# Skill Bank Scoring Rubric

> **Version:** 1.0.0
> **Derived from:** 3 seed skills (`crypto-whale-monitor`, `airdrop-hunter-pro`, `polymarket-emerging-tech-trader`) + 1 high-quality reference (`narrative-alignment-score`)
> **Date:** 2026-04-07

This rubric is used by Stage 4 (`/auto-req-evaluate`) to gate both requirements and skill drafts. Each criterion is binary (1 = yes, 0 = no). Items are evaluated independently.

---

## Requirement Scoring Criteria

**Pass threshold: >= 4 out of 6**

These criteria evaluate whether a surfaced requirement is worth turning into a Claude Code skill.

| # | Criterion | Question | Score |
|---|-----------|----------|-------|
| R1 | **Specificity** | Is the requirement specific enough to implement as a single, focused skill? It should describe one well-bounded capability, not a vague category ("improve my trading") or an umbrella of unrelated features. | 0 or 1 |
| R2 | **Actionability** | Does the requirement describe a concrete action that Claude Code can perform? It must be something Claude Code can execute end-to-end (fetch data, compute a result, produce structured output) -- not a passive observation, opinion, or manual-only workflow. | 0 or 1 |
| R3 | **User grounding** | Is the requirement grounded in a real user pain point with a concrete scenario? There should be at least one specific situation where a real user (retail trader, DeFi dev, solo builder, beginner) lost money, wasted time, or missed an opportunity due to the absence of this capability. Vague "it would be nice" requirements score 0. | 0 or 1 |
| R4 | **Multi-persona resonance** | Did multiple personas (>= 2 out of 4) independently raise, support, or validate this need during simulation? Requirements that resonate with only one persona may be too niche. Higher resonance (3-4 personas) indicates broader value. | 0 or 1 |
| R5 | **Novelty** | Is the requirement distinct from existing promoted skills in `skill_bank/promoted/`? Check for semantic overlap -- a requirement that duplicates or trivially extends an already-promoted skill scores 0. Minor variations on existing skills also score 0 unless they address a clearly different user scenario. | 0 or 1 |
| R6 | **MCP feasibility** | Can the resulting skill be implemented using available MCP tools and Claude Code capabilities? Available MCP tools include: `ant_market_sentiment`, `ant_macro_economics`, `ant_market_indicators`, `ant_fund_flow`, `ant_etf_fund_flow`, `ant_futures_market_structure`, `ant_spot_market_structure`, `ant_futures_liquidation`, `ant_smart_money`, `ant_stablecoin`, `ant_protocol_tvl_yields_revenue`, `ant_protocol_ecosystem`, `ant_token_analytics`, `ant_meme`, `ant_perp_dex`, `ant_binance_spot`, `ant_binance_futures`, `ant_bridge_fund_flow`, `ant_precious_metal_tokens`, `ant_us_stock_tokens`, `ant_address_profile`. Requirements that need unavailable external APIs, private data sources, or real-time execution infrastructure that Claude Code cannot provide score 0. | 0 or 1 |

### Scoring guidance

- **Score 6/6:** Strong candidate -- proceed to skill draft generation immediately.
- **Score 5/6:** Good candidate -- proceed, but note the missing criterion for the skill author to address.
- **Score 4/6:** Borderline -- proceed with caution; the skill draft must compensate for weak criteria.
- **Score 0-3/6:** Reject -- do not generate a skill draft. Log the requirement in `skill_bank/candidates/` with the score and feedback for potential future reconsideration.

---

## Skill Quality Criteria

**Pass threshold: >= 6 out of 10**

These criteria evaluate whether a generated SKILL.md draft is ready for promotion to `skill_bank/promoted/`. The criteria are derived from analyzing what distinguishes the best seed skills (especially `narrative-alignment-score`) from weaker ones.

| # | Criterion | Question | Score |
|---|-----------|----------|-------|
| S1 | **Valid YAML frontmatter** | Does the skill have valid YAML frontmatter (between `---` delimiters) containing at minimum `name` and `description` fields? The description should be a single sentence explaining when to invoke the skill. | 0 or 1 |
| S2 | **User persona profiles** | Does the skill include at least 2 user persona profiles, each with: (a) a name and role, (b) a behavioral description of their daily workflow, (c) their tool stack, and (d) a specific pain point that this skill addresses? Personas must represent ordinary Claude Code users (retail traders, DeFi developers, full-stack solo devs, beginners) -- not celebrities or influencers. | 0 or 1 |
| S3 | **User stories** | Does the skill include at least 2 user stories in the format "As [persona], I want [capability], so that [benefit]"? Each story must reference a specific persona from the profiles and articulate a concrete benefit tied to their pain point. | 0 or 1 |
| S4 | **Trigger conditions** | Does the skill have a clear "When This Skill Applies" section (or equivalent) that specifies: (a) the situation in which the skill should be invoked, and (b) example trigger phrases a user might say? The triggers must be specific enough that Claude Code can reliably decide when to activate this skill. | 0 or 1 |
| S5 | **Numbered executable steps** | Does the skill have a "What To Do" section (or equivalent) with numbered, sequential steps that Claude Code can execute? Each step must specify: what to call (MCP tool name or computation), what parameters to pass, and what to do with the result. Steps must not be vague ("analyze the data") -- they must be concrete ("Call `ant_market_sentiment(query_type='topics_list')` and filter to crypto topics"). | 0 or 1 |
| S6 | **Specific output format** | Does the skill define a specific output template or format that Claude Code should produce? The format must show the exact structure (field names, layout, sections) of the final output -- not just "return a summary." A code-fenced template with placeholder values is ideal. | 0 or 1 |
| S7 | **Concrete example** | Does the skill include at least one concrete example showing: (a) an example user request/trigger, and (b) what the output would look like (with realistic sample values, not just the template)? The example must be specific enough that a reader can verify the skill works correctly. | 0 or 1 |
| S8 | **Anti-patterns or caveats** | Does the skill include an anti-patterns section, caveats, or explicit warnings about what NOT to do? This includes: common misuse patterns, data interpretation pitfalls, or scope boundaries (what the skill does NOT cover). A skill with no guardrails scores 0. | 0 or 1 |
| S9 | **Correct MCP tool usage** | Does the skill reference MCP tools by their correct names and use valid parameters? Every MCP call must use a tool that actually exists in the available MCP tool set (see R6 list above). Invented tool names, incorrect parameter formats, or references to unavailable APIs score 0. Skills that require no MCP calls (pure computation) get a free pass on this criterion. | 0 or 1 |
| S10 | **Immediately executable** | Is the skill complete and immediately executable by Claude Code without further development? There must be no TBD sections, placeholder logic, "coming soon" features, or references to scripts/files that do not exist. Every step must be actionable right now. A skill that requires external setup (installing packages, configuring API keys, running scripts) that Claude Code cannot perform scores 0. | 0 or 1 |
| S11 | **Multi-layer causal chain** | Does the skill require at least 2 layers of "why" reasoning in its analysis steps? Example: funding rate high → why (crowded longs) → consequence (liquidation risk) → reversal condition. A skill that only does "data → conclusion" with no intermediate causal reasoning scores 0. | 0 or 1 |
| S12 | **Cross-source verification** | Does the skill require ≥2 independent MCP data sources to cross-verify the same conclusion before outputting high confidence? Example: stablecoin inflow + ETF fund flow + on-chain whale behavior must align for a "high confidence" verdict. A skill that draws conclusions from a single data source scores 0. | 0 or 1 |
| S13 | **Invalidation conditions** | Does the skill output include explicit "this judgment fails if..." conditions? Every core conclusion must state at least 1 condition under which the conclusion would be wrong or reversed. A skill that only presents bullish/bearish arguments without stating failure conditions scores 0. | 0 or 1 |

### Scoring guidance

- **Score 12-13/13:** Promote immediately to `skill_bank/promoted/`.
- **Score 10-11/13:** Promote with minor revision notes attached. Log what is missing.
- **Score 8-9/13:** Borderline promote -- add to `skill_bank/promoted/` with a `needs_improvement: true` flag and specific feedback. Eligible for `/auto-req-evolve` improvement.
- **Score 0-7/13:** Reject -- move to `skill_bank/candidates/` with detailed feedback. May be retried after revision.

### Reference scores for seed skills

These scores illustrate how the rubric applies to known skills, calibrating evaluators:

| Skill | S1 | S2 | S3 | S4 | S5 | S6 | S7 | S8 | S9 | S10 | S11 | S12 | S13 | Total | Verdict |
|-------|----|----|----|----|----|----|----|----|----|----|-----|-----|-----|-------|---------|
| `narrative-alignment-score` | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 0 | 1 | 1 | 1 | 1 | 0 | **11/13** | Promote |
| `polymarket-emerging-tech-trader` | 1 | 0 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | **2/13** | Reject |
| `crypto-whale-monitor` | 1 | 0 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | **2/13** | Reject |
| `airdrop-hunter-pro` | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | **1/13** | Reject |

---

## How This Rubric Is Used

1. **Stage 4 (`/auto-req-evaluate`)** scores each surfaced requirement against the Requirement Scoring Criteria. Requirements scoring >= 4/6 proceed to skill draft generation.

2. **Stage 4** then scores each generated skill draft against the Skill Quality Criteria. Skills scoring >= 6/10 are promoted to `skill_bank/promoted/`.

3. **Stage 5 (`/auto-req-evolve`)** uses this rubric to evaluate whether evolved skill versions improve on their predecessors. An evolved skill must score equal to or higher than the original on every criterion.

4. **`/auto-req-rubric`** updates this file when new seed skills are added to `seed_skills/` or when user feedback indicates criteria need adjustment. Changes are logged in `skill_bank/CHANGELOG.md`.

---

## Changelog

| Date | Version | Change |
|------|---------|--------|
| 2026-04-07 | 1.0.0 | Initial rubric derived from 3 seed skills + 1 reference skill |
| 2026-04-09 | 1.1.0 | Added S11 (multi-layer causal chain), S12 (cross-source verification), S13 (invalidation conditions) for analysis depth. Scoring thresholds updated from /10 to /13. |
