---
name: auto-req-evaluate
description: Stage 4 of redesigned auto_req pipeline. Two-layer evaluation: (1) scores candidate requirements using skill_bank/rubric.md, (2) generates SKILL.md drafts for passing requirements and scores them. Requires discussions/YYYY-MM-DD/ from /auto-req-simulate. Triggers: "run evaluate", "evaluate requirements", "score requirements", "generate skill drafts".
---

# auto-req-evaluate: Requirement Scoring + Skill Generation

**语言要求：所有生成的 skill 文档、需求报告、反馈文件必须使用中文。MCP 工具名、参数名、JSON 字段名保持英文。**

Two-layer quality gate: first score the requirements, then generate and score the skill drafts. Only promote skills that pass both layers.

## Setup

Get today's date (YYYY-MM-DD).

Check for today's discussions, in this order:
1. `discussions/YYYY-MM-DD/round3.json` (required minimum)
2. `discussions/YYYY-MM-DD/round4.json` (use if exists — higher quality signal)
3. If not found: `test_fixtures/` equivalents for offline testing

Read `skill_bank/rubric.md`. If it does not exist, print:
```
rubric.md not found. Run /auto-req-rubric first.
```
And stop.

List all SKILL.md files in `skill_bank/promoted/` (to check novelty in Layer 1 scoring).

## Step 1: Extract Candidate Requirements

Dispatch a single subagent to extract candidate requirements from the discussion files:

```
REQUIREMENT EXTRACTION

You are a product researcher extracting actionable skill requirements from a multi-round discussion between 4 personas.

ROUND 3 DATA:
[round3.json contents]

ROUND 4 DATA (if exists):
[round4.json contents, or "None"]

Extract all distinct candidate requirements. A requirement is something specific enough to become a Claude Code skill — a tool or workflow that would address a real pain point expressed in the discussion.

For each candidate requirement, capture:
- Which personas mentioned it (and in which round)
- The concrete scenario where it would be used
- What the skill would do when invoked

输出中所有描述性文字使用中文

Output ONLY a JSON array:
[
  {
    "id": "req-001",
    "title": "short descriptive title",
    "description": "what this skill does and when you'd use it",
    "source_personas": ["alice", "bob"],
    "source_rounds": [1, 3],
    "concrete_scenario": "specific situation from the discussion",
    "proposed_invocation": "/skill-name [example context]",
    "related_desire_ids": ["d-001", "d-002"]
  }
]
```

### Step 1b: Dedup Candidate Requirements

Before scoring, dispatch a single subagent to check for semantic overlap:

```
REQUIREMENT DEDUP

You have a list of candidate requirements. Check for semantic overlap.

REQUIREMENTS:
[candidate requirements JSON array]

输出中所有描述性文字使用中文

For each pair of requirements, assess whether they describe essentially the same capability for the same audience. Two requirements overlap if:
- They solve the same core problem (e.g., both are "institutional vs retail sentiment divergence")
- AND they target the same user type at the same depth level

Two requirements are DISTINCT if they:
- Solve the same problem BUT for clearly different audiences (e.g., one is a quantitative spread for traders, another is a plain-English dashboard for beginners)
- OR solve genuinely different problems that merely share a data source

For each overlapping pair, merge them: keep the one with higher multi-persona resonance, absorb the other's unique details (scenarios, personas) into it. Update source_personas to be the union.

Output ONLY the deduplicated JSON array (same schema as input). If no merges were needed, output the original array unchanged.
```

Replace the candidate requirements list with the deduplicated output.

Print: "Dedup: N candidates → M after merging (N-M merged)"

## Step 2: Score Each Requirement (Layer 1)

For each candidate requirement, dispatch a scoring subagent using the rubric:

```
REQUIREMENT SCORING

Read the rubric criteria below and score this requirement.

RUBRIC — REQUIREMENT SCORING CRITERIA:
[skill_bank/rubric.md — "Requirement Scoring Criteria" section only]

EXISTING PROMOTED SKILLS (for novelty check):
[list of filenames + one-line descriptions from skill_bank/promoted/]

CANDIDATE REQUIREMENT:
[requirement JSON]

Score each criterion as 1 (yes) or 0 (no). Then give the total.

Output ONLY JSON:
{
  "req_id": "req-001",
  "scores": {
    "[criterion name]": 1 or 0
  },
  "total": N,
  "max": N,
  "pass": true or false,
  "improvement_suggestions": "If total < max, what specific information or refinement would raise the score? Be concrete. If total >= max, write null."
}

improvement_suggestions 和 failed_criteria 中的描述使用中文
```

**Route each requirement:**
- Pass (`pass: true`) → add to `passing_requirements` list
- Low score (total > 0 but `pass: false`) → write to `skill_bank/candidates/<req-id>-feedback.md`:
  ```markdown
  # Requirement Feedback: [title]
  
  **Score:** N/max
  **Date:** YYYY-MM-DD
  
  ## Original Requirement
  [description]
  
  ## Scoring
  [criterion-by-criterion breakdown]
  
  ## Improvement Suggestions
  [improvement_suggestions text]
  ```
- Zero score → log discarded requirement with reason, skip

Print after Layer 1:
```
Requirement scoring complete:
  Passed:    N → proceeding to skill generation
  Candidates: N → saved to skill_bank/candidates/
  Discarded:  N → logged
```

## Step 3: 生成 PRD + Skill 草稿（分离）

读取今日 roster：`persona_bank/roster_YYYY-MM-DD.json`。对 roster 中每个 persona，从其 `source` 路径加载完整 persona JSON。

对每个通过的需求，**派遣两个子代理**（顺序执行）：

### Step 3a: 生成需求文档（PRD）

```
PRD GENERATION

为以下需求生成需求分析文档（PRD）。

REQUIREMENT:
[requirement JSON]

来源 PERSONA 及其状态：
[对 source_personas 中的每个 persona：从 roster 加载。常驻 persona 包含完整 BDI 状态。临时 persona 包含 archetype、dominant_tags、beliefs 和 intentions。]

DISCUSSION EXCERPTS:
[Relevant quotes from source_rounds where this requirement was raised or supported]

按以下结构撰写 PRD：

1. **需求概述**：一段话描述这个 skill 解决什么问题

2. **用户画像**
   - 共鸣度："X/5 persona — [总结]"
   - 每个相关 persona 的画像：角色、日常工作流、工具栈、具体痛点（损失/错过的机会）
   - 场景表：每 persona 一行（情境 / 当时做了什么 / 代价）

3. **用户故事**
   - 每个 persona 一条："作为 [persona]，我想要 [能力]，以便 [结果]"

4. **讨论中的代表性引言**（注明出自哪个 persona）

5. **需求边界**
   - 做什么 / 不做什么
   - 数据源依赖
   - 已知限制

⚠️ PERSONA 差异化规则：
每个 persona 的痛点必须描述问题的不同侧面。
常驻 persona 基于 BDI 历史积累。临时 persona 基于聚类标签和当天帖子。
如果无法找到独特侧面，宁可省略该 persona。

输出完整 PRD 内容（不要输出其他文字）。
```

保存到 `skill_bank/requirements/<req-id>-prd.md`。

### Step 3b: 生成精简 SKILL.md

```
SKILL GENERATION

为以下需求生成一个精简的 Claude Code skill（纯执行指令，不含用户画像/用户故事）。

REQUIREMENT:
[requirement JSON]

CONCRETE SCENARIO:
[concrete_scenario field]

PROPOSED INVOCATION:
[proposed_invocation field]

PRD 参考（仅用于理解需求背景，不要复制到 SKILL.md）：
[刚生成的 PRD 内容摘要 — 需求概述 + 需求边界]

AVAILABLE MCP TOOLS — USE ONLY THESE (any other tool name is INVALID):

ant_market_sentiment(query_type, coin?, topic?, category?, network?, creator_id?, sort?, limit?, desc?, page?, bucket?, interval?, start?, end?)
  query_types: coins_list, coin_detail, coin_time_series, coin_meta, topics_list, topic_detail, topic_time_series, topic_posts, topic_news, topic_creators, categories_list, category_detail, category_topics, creators_list, creator_detail

ant_market_indicators(query_type, symbol="BTC", exchange?, interval?, limit?, window?, series_type?, fast_window?, slow_window?, signal_window?, mult?)
  query_types: rsi, basis, ma, ema, boll, macd, macd_list, whale_index, cgdi, cdri, atr, netflow, orderbook_ask_bids, taker_flow, taker_flow_aggregated, coinbase_premium, bitfinex_margin, borrow_rate

ant_macro_economics(query_type, interval?, maturity?)
  query_types: real_gdp, real_gdp_per_capita, cpi, inflation, expectation, federal_funds_rate, treasury_yield, unemployment, nonfarm

ant_fund_flow(query_type, asset?, exchange?, miner?, token?, symbol?, window="day", from_date?, to_date?, limit=100)
  query_types: exchange_reserve, exchange_netflow, exchange_inflow, exchange_outflow, exchange_transactions_count, miner_reserve, miner_netflow, miner_inflow, miner_outflow, exchange_whale_ratio, mpi, fund_flow_ratio, centralized_exchange_assets, centralized_exchange_balance_list, centralized_exchange_balance_chart, centralized_exchange_transfers, centralized_exchange_whale_transfer

ant_futures_market_structure(query_type, symbol="BTC", exchange?, exchange_list?, interval?, limit?, range?, model=1)
  query_types: futures_market_snapshot, futures_pairs_market, futures_price_change, futures_price_history, futures_oi_history, futures_oi_aggregated, futures_oi_stablecoin, futures_oi_coin_margin, futures_oi_exchange_list, futures_oi_exchange_history, futures_funding_rate_history, futures_funding_rate_exchange_list, futures_funding_rate_oi_weight, futures_funding_rate_vol_weight, futures_funding_rate_accumulated, futures_funding_rate_arbitrage, futures_long_short_global, futures_long_short_top_account, futures_long_short_top_position, futures_taker_ratio_exchange, futures_net_position, futures_orderbook_ask_bids, futures_orderbook_aggregated, futures_orderbook_history, futures_orderbook_large_orders, futures_orderbook_large_orders_history, futures_taker_flow, futures_taker_flow_aggregated, futures_footprint, futures_cvd, futures_cvd_aggregated, futures_netflow

ant_smart_money(query_type, chains?, filters?, order_by?, pagination?, extra?)
  query_types: netflows, holdings, historical_holdings, dex_trades, dcas, perp_trades

⚠️ MCP RULES:
- Only use exact query_types listed above. Do NOT invent.
- If data is unavailable via MCP, say "ask the user to provide [X]".

按以下结构撰写 SKILL.md（**不含用户画像、用户故事、场景表**）：

1. **YAML frontmatter**: name (kebab-case), description (一句话：触发条件 + 功能)

2. **功能说明**（2-3 句，说清楚这个 skill 做什么）

3. **适用场景**
   - 触发情境和示例短语

4. **输入参数**
   - 用户需要提供什么（如币种、事件关键词、时间范围等）
   - 可选参数和默认值

5. **执行步骤**
   - Claude Code 可直接执行的清晰编号步骤
   - 每步包含具体 MCP 调用（tool_name + query_type + params）
   - 步骤之间的数据流转逻辑

6. **输出格式**
   - 具体的输出模板（用户最终看到什么）

7. **使用示例**
   - 1-2 个具体示例（含输入和预期输出摘要）

8. **反模式**
   - 本 skill 不做什么

skill 必须可以立即执行——不允许出现"待定"或模糊指令。

输出完整 SKILL.md 内容（不要输出其他文字）。
```

保存到 `skill_bank/candidates/<req-id>-draft.md`。

## Step 4: Score Skill Drafts (Layer 2)

For each generated draft, dispatch a scoring subagent:

```
SKILL QUALITY SCORING

Score this skill draft against the rubric AND verify MCP tool usage.

RUBRIC — SKILL QUALITY CRITERIA:
[skill_bank/rubric.md — "Skill Quality Criteria" section only]

SKILL DRAFT:
[draft content]

Score each criterion as 1 (yes) or 0 (no).

⚠️ FOR CRITERION S9 (MCP tool usage), you MUST perform line-by-line verification:
1. Extract every MCP tool call from the "What To Do" section
2. For each call, check against this VALID TOOL REGISTRY:
   - ant_market_sentiment: query_types = coins_list, coin_detail, coin_time_series, coin_meta, topics_list, topic_detail, topic_time_series, topic_posts, topic_news, topic_creators, categories_list, category_detail, category_topics, creators_list, creator_detail
   - ant_market_indicators: query_types = rsi, basis, ma, ema, boll, macd, macd_list, whale_index, cgdi, cdri, atr, netflow, orderbook_ask_bids, taker_flow, taker_flow_aggregated, coinbase_premium, bitfinex_margin, borrow_rate
   - ant_macro_economics: query_types = real_gdp, real_gdp_per_capita, cpi, inflation, expectation, federal_funds_rate, treasury_yield, unemployment, nonfarm
   - ant_fund_flow: query_types = exchange_reserve, exchange_netflow, exchange_inflow, exchange_outflow, exchange_transactions_count, miner_reserve, miner_netflow, miner_inflow, miner_outflow, exchange_whale_ratio, mpi, fund_flow_ratio, centralized_exchange_assets, centralized_exchange_balance_list, centralized_exchange_balance_chart, centralized_exchange_transfers, centralized_exchange_whale_transfer
   - ant_futures_market_structure: query_types = futures_market_snapshot, futures_pairs_market, futures_price_change, futures_price_history, futures_oi_history, futures_oi_aggregated, futures_oi_stablecoin, futures_oi_coin_margin, futures_oi_exchange_list, futures_oi_exchange_history, futures_funding_rate_history, futures_funding_rate_exchange_list, futures_funding_rate_oi_weight, futures_funding_rate_vol_weight, futures_funding_rate_accumulated, futures_funding_rate_arbitrage, futures_long_short_global, futures_long_short_top_account, futures_long_short_top_position, futures_taker_ratio_exchange, futures_net_position, futures_orderbook_ask_bids, futures_orderbook_aggregated, futures_orderbook_history, futures_orderbook_large_orders, futures_orderbook_large_orders_history, futures_taker_flow, futures_taker_flow_aggregated, futures_footprint, futures_cvd, futures_cvd_aggregated, futures_netflow
   - ant_smart_money: query_types = netflows, holdings, historical_holdings, dex_trades, dcas, perp_trades
3. If ANY call uses an invalid tool name, invalid query_type, or fabricated parameter: S9 = 0
4. If the skill references data that no MCP tool can provide (e.g., stock prices, SEC filings) without explicitly asking the user to provide it: S9 = 0
5. List every invalid call found in `failed_criteria`

Output ONLY JSON:
{
  "draft_id": "req-001",
  "scores": {
    "[criterion name]": 1 or 0
  },
  "total": N,
  "max": N,
  "pass": true or false,
  "failed_criteria": [
    {"criterion": "name", "reason": "specific problem", "fix": "specific improvement"}
  ]
}

improvement_suggestions 和 failed_criteria 中的描述使用中文
```

**Route each draft:**
- Pass → determine filename from skill `name` field in frontmatter (kebab-case), copy to `skill_bank/promoted/<skill-name>.md`
- Fail → keep in `skill_bank/candidates/<req-id>-draft.md`, append scoring feedback:
  ```markdown
  ---
  ## Quality Scoring Results
  **Score:** N/max
  **Status:** Not promoted
  
  ### Failed Criteria
  [For each failed criterion: name + reason + fix]
  ```

## Done

Print:
```
=== auto-req-evaluate complete ===
Requirements scored:  N
  Passed layer 1:     N
  Candidates:         N → skill_bank/candidates/
  Discarded:          N

PRDs generated:       N → skill_bank/requirements/
Skills generated:     N
  Promoted:           N → skill_bank/promoted/
  Not promoted:       N → skill_bank/candidates/ (with feedback)

Next step: run /auto-req-evolve (when enough history exists in discussions/)
```
