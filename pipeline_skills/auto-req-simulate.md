---
name: auto-req-simulate
description: Stage 3 of redesigned auto_req pipeline. Runs adaptive multi-round BDI-aware discussion simulation. Dynamically loads persona roster from persona_bank/roster_YYYY-MM-DD.json (permanent + temporary). Adds conditional Round 4 for high-resonance desires (permanent only). Run after /auto-req-persona. Saves rounds to discussions/YYYY-MM-DD/. Triggers: "run simulate", "run discussion", "simulate personas".
---

# auto-req-simulate: Adaptive BDI-Aware Discussion Simulation

根据当日 roster 动态加载所有 persona（常驻 + 临时），运行多轮 BDI 感知讨论模拟。讨论深度根据共鸣度自适应调整。

## Setup

获取今天的日期（YYYY-MM-DD）。

**加载 persona roster（必需）：**
1. 读取 `persona_bank/roster_YYYY-MM-DD.json`
2. 如果文件不存在，停止执行并输出：`"请先运行 /auto-req-persona 生成今日 roster。"`
3. 解析 roster JSON，对每条 entry 根据 `source` 路径加载对应 persona 文件
4. 打印 roster 摘要：`"Roster: N 个 persona（P 常驻 + T 临时）"`，逐行列出每个 persona 的 `persona_id`、`type`（permanent/temporary）和 `archetype`

**加载今日帖子（按优先级）：**
1. `raw_posts/YYYY-MM-DD-filtered.json`（过滤后的帖子优先）
2. `raw_posts/YYYY-MM-DD.json`（原始帖子）
3. `test_fixtures/raw_posts_sample.json`（离线测试）

打印使用的文件路径。

创建目录 `discussions/YYYY-MM-DD/`。

**Pre-flight check：** 对 roster 中每个 persona 检查：
- `unresolved_desires`：`resolved=false` 的 desires
- `active_intentions`：`intentions` 数组
- `high_resonance_desires`：`resolved=false AND resonance >= 3` 的 desires
- 注意：临时 persona 可能有空或极少的 BDI 状态，这属于正常情况

如果某主题的相关 desires 在所有 persona 中均已 resolved，标记为跳过候选（打印提示，不做模拟）。

## Round 1: First Reactions

对 roster 中每个 persona 各派遣一个子代理（并行），使用 Agent tool。每个子代理接收该 persona 的完整 BDI 状态和所有帖子。对每个 persona 使用以下 prompt（填入 [PERSONA_JSON], [UNRESOLVED_DESIRES], [INTENTIONS], [POSTS_JSON]）：

```
ROUND 1 - PERSONA SIMULATION

所有回答和描述必须使用中文。JSON 字段名保持英文，字段值使用中文。

You are roleplaying as the following person. Stay completely in character.

YOUR PERSONA:
[PERSONA_JSON — full persona including archetype, tech_stack, daily_workflow, behavior_signature]

YOUR CURRENT MINDSET (use this to guide what you focus on):
- What you believe about the market right now: [persona.beliefs filtered to stale=false, as bullet list]
- What you're still trying to solve: [UNRESOLVED_DESIRES — desires where resolved=false, sorted by resonance desc, as bullet list]
- What you came here to explore today: [INTENTIONS — persona.intentions as bullet list]

You are scrolling through X/Twitter and see these posts:

POSTS:
[POSTS_JSON]

As this specific person — with your current beliefs and open pain points — respond to these posts. Let your existing frustrations and today's intentions shape what you notice and what you wish existed.

⚠️ SCOPE CONSTRAINT: 你提出的 skill 必须服务于金融/加密货币/投资分析工作流。具体来说：
- ✅ 市场数据分析（价格、持仓、资金流、链上指标）
- ✅ 交易决策辅助（信号验证、风险评估、回测）
- ✅ 宏观/叙事分析（利率影响、政策解读、市场情绪）
- ✅ DeFi/代币分析（收益率对比、协议健康度、代币经济学）
- ✅ 投资组合管理（仓位监控、风险敞口、相关性分析）
- ✅ 地缘政治对市场的影响分析（关税→供应链股、制裁→资金流、地缘冲突→避险资产）
- ✅ AI/科技对投资的影响（AI 概念股/代币联动、算力成本→挖矿、GPU 供应链→相关标的）
- ❌ 不要提出通用开发工具（代码安全扫描、CI/CD、通用自动化）
- ❌ 不要提出非金融领域的工具（SaaS运营、内容创作、项目管理）

Output a JSON object (no other text):
{
  "persona_name": "name from persona",
  "most_resonant_post": "post id that hit hardest",
  "why_it_resonates": "specific reason tied to YOUR workflow and current frustrations",
  "pain_point_recognized": "the underlying FINANCIAL/CRYPTO problem you see, in your own words",
  "wish_existed": "a specific Claude Code skill for financial/crypto analysis — be concrete about what market data it uses, what analysis it performs, and when you'd invoke it",
  "desire_triggered": "which of your existing unresolved desire IDs (e.g. d-001) does this connect to? Or 'new' if this is a fresh pain point",
  "side_thought": "a financial/market insight the post didn't say directly"
}
```

收集所有 persona 的回复。保存至 `discussions/YYYY-MM-DD/round1.json`：

```json
{
  "date": "YYYY-MM-DD",
  "round": 1,
  "posts_used": ["post_001", "post_002", "..."],
  "responses": [
    { "persona_name": "...", "most_resonant_post": "...", "why_it_resonates": "...", "pain_point_recognized": "...", "wish_existed": "...", "desire_triggered": "...", "side_thought": "..." }
  ]
}
```

Print: "Round 1 complete → discussions/YYYY-MM-DD/round1.json"

## Round 1 后更新共鸣度（Desire Resonance）

对每个 persona 的 Round 1 回复，按类型分别处理：

### 常驻 persona（type=permanent）
- 如果 `desire_triggered` 匹配已有 desire id（如 "d-001"）→ 在 `persona_bank/<id>.json` 中将该 desire 的 `resonance` +1
- 如果 `desire_triggered` 为 "new" → 添加新 desire 条目：
  ```json
  {
    "id": "d-NNN",
    "content": "[pain_point_recognized from Round 1 response]",
    "resonance": 1,
    "resolved": false,
    "resolved_by": null,
    "first_seen": "YYYY-MM-DD"
  }
  ```
- 将更新后的 persona 文件写回 `persona_bank/`

### 临时 persona（type=temporary）
- 同样记录共鸣度变化，但**仅保存在内存中**，不写回文件
- 临时 persona 的新 desires 用于后续轮次参考，但不持久化

重新检查 `high_resonance_desires`（更新后 resonance >= 3）。注意：**仅常驻 persona** 的高共鸣 desires 用于判断是否触发 Round 4。

## Round 2: Cross-Reactions

对 roster 中每个 persona 各派遣一个子代理（并行）。每个 persona 现在可以看到所有 persona 的 Round 1 回复及原始帖子。对每个 persona 使用以下 prompt（填入 [PERSONA_JSON], [POSTS_JSON], [ROUND1_RESPONSES_JSON]）：

```
ROUND 2 - CROSS-REACTION

所有回答和描述必须使用中文。JSON 字段名保持英文，字段值使用中文。

You are roleplaying as the following person. Stay completely in character.

YOUR PERSONA:
[PERSONA_JSON]

You read these posts earlier:
POSTS: [POSTS_JSON]

And now you see how the group (including yourself) reacted:
ROUND 1 RESPONSES: [ROUND1_RESPONSES_JSON]

As this specific person, engage with what others said. Don't just summarize — react from your perspective.

Output a JSON object (no other text):
{
  "persona_name": "name from persona",
  "strong_agreement": {
    "with_persona": "name of persona you agree with most",
    "their_point": "exact point you're agreeing with",
    "why_from_your_view": "why THIS matters from YOUR specific role and experience"
  },
  "disagreement": {
    "with_persona": "name of persona you'd push back on, or null if none",
    "their_point": "what they said",
    "your_counter": "your specific counterpoint or different framing"
  },
  "new_realization": "something they mentioned that you hadn't thought of before, but now recognize as a real need for you too",
  "emerging_theme": "a pattern you see across the group's reactions — something bigger than any single post"
}
```

收集所有 persona 的回复。保存至 `discussions/YYYY-MM-DD/round2.json`：

```json
{
  "date": "YYYY-MM-DD",
  "round": 2,
  "responses": [
    { "persona_name": "...", "strong_agreement": { "..." }, "disagreement": { "..." }, "new_realization": "...", "emerging_theme": "..." }
  ]
}
```

Print: "Round 2 complete → discussions/YYYY-MM-DD/round2.json"

## Round 3: Facilitator + Targeted Probe

### Step 3a — Facilitator Analysis

Dispatch a single subagent with this prompt (fill in [ROUND1_JSON] and [ROUND2_JSON]):

```
FACILITATOR ANALYSIS

所有回答和描述必须使用中文。JSON 字段名保持英文，字段值使用中文。

You are a product researcher facilitating a focus group of developers discussing Claude Code.

You have observed two rounds of discussion:

ROUND 1 (first reactions to posts):
[ROUND1_JSON]

ROUND 2 (cross-reactions):
[ROUND2_JSON]

执行两层分析：

**第一层：显性主题提取**
Identify the 3 most significant themes — these could be:
- Points of strong convergence (multiple personas independently naming the same need)
- Points of productive tension (personas who disagree in ways that reveal a deeper unresolved problem)
- Surprising realizations that appeared in Round 2 but not Round 1

**第二层：隐含需求推理**
在显性主题之外，分析讨论中未被直接说出但可以推导出的潜在需求：

1. **行为推理**：从帖子和讨论中用户"在做什么"推导"缺什么工具"
   - 例：用户手动对比5个交易所价格 → 缺跨交易所价差实时监控
   - 例：用户反复提到"我查了X又查了Y然后算了Z" → 缺一键聚合分析工具

2. **负空间检测**：今天的帖子和讨论中大家集体忽略了什么本应关注的金融/crypto 维度？
   - 例：大家讨论 BTC ETF 资金流但没人看期权隐含波动率 → 可能是工具缺失导致的认知盲区
   - 例：讨论 DeFi 收益率但没人提无常损失 → 风险评估工具可能不够直观

3. **提问模式聚类**：帖子和讨论中出现的问句或疑惑，反复出现的问题类型 = 系统性信息缺口
   - 例：多人问"这个数据在哪里看" → 数据源发现工具需求
   - 例：多人问"历史上类似情况怎样" → 历史类比/回测工具需求

Output a JSON object (no other text):
{
  "themes": [
    {
      "id": "theme_1",
      "title": "short theme title",
      "description": "what this theme is about in 2-3 sentences",
      "evidence": ["quote or reference from round1/round2", "another quote"],
      "tension_type": "convergence / productive_tension / surprise",
      "probe_question": "a sharp, specific question to ask all personas to deepen this theme"
    },
    { "id": "theme_2", "..." },
    { "id": "theme_3", "..." }
  ],
  "latent_needs": [
    {
      "id": "latent_1",
      "type": "behavior_inference / negative_space / question_pattern",
      "observation": "观察到的现象（用户在做什么/讨论中缺什么/反复出现什么问题）",
      "inferred_need": "推导出的潜在工具需求",
      "evidence": ["支持这个推断的具体帖子内容或讨论片段"],
      "confidence": "high / medium / low（基于证据强度）",
      "probe_question": "验证这个隐含需求的探测问题"
    }
  ]
}
```

### Step 3b — Persona Responses to Themes

对 roster 中每个 persona 各派遣一个子代理（并行）。每个子代理接收 3 个主题及该 persona 的 BDI 状态。对每个 persona 使用以下 prompt（填入 [PERSONA_JSON] 和 [THEMES_JSON]）：

```
ROUND 3 - THEME PROBE

所有回答和描述必须使用中文。JSON 字段名保持英文，字段值使用中文。

You are roleplaying as the following person. Stay completely in character.

YOUR PERSONA:
[PERSONA_JSON]

A facilitator has identified 3 themes and several latent (hidden) needs from your group's discussion. Respond to both.

THEMES:
[THEMES_JSON — themes 数组部分]

LATENT NEEDS:
[LATENT_NEEDS_JSON — latent_needs 数组部分]

For each theme AND each latent need, answer the facilitator's probe question from YOUR specific perspective. Be concrete — give a real scenario from your daily work.

Output a JSON object (no other text):
{
  "persona_name": "name from persona",
  "theme_responses": [
    {
      "theme_id": "theme_1",
      "probe_answer": "your specific answer to the probe question",
      "concrete_scenario": "a real situation from your daily work where this theme shows up",
      "ideal_skill": {
        "what_it_does": "describe a FINANCIAL/CRYPTO analysis skill in one sentence — must involve market data, trading decisions, or investment analysis",
        "trigger": "when would you invoke it (must be a financial/crypto scenario)",
        "example_invocation": "show an example: /skill-name [financial context]"
      }
    },
    { "theme_id": "theme_2", "..." },
    { "theme_id": "theme_3", "..." }
  ],
  "latent_need_responses": [
    {
      "latent_id": "latent_1",
      "is_relevant_to_me": true,
      "probe_answer": "从你的视角回答探测问题",
      "my_workaround": "你目前怎么解决这个问题（或者根本没意识到这是个问题）",
      "ideal_skill": {
        "what_it_does": "describe a FINANCIAL/CRYPTO analysis skill in one sentence",
        "trigger": "when would you invoke it (must be a financial/crypto scenario)",
        "example_invocation": "/skill-name [financial context]"
      }
    }
  ]
}
```

收集 facilitator 输出及所有 persona 的回复。保存至 `discussions/YYYY-MM-DD/round3.json`：

```json
{
  "date": "YYYY-MM-DD",
  "round": 3,
  "facilitator": { "themes": [...], "latent_needs": [...] },
  "persona_responses": [
    { "persona_name": "...", "theme_responses": [...], "latent_need_responses": [...] }
  ]
}
```

Print: "Round 3 complete → discussions/YYYY-MM-DD/round3.json"

## Round 4: Deep Probe (Conditional)

仅当任一**常驻 persona**（type=permanent）存在 `resonance >= 3` 且 `resolved=false` 的 desire 时才执行。临时 persona 的高共鸣 desires 不触发 Round 4，但临时 persona 仍参与回答 probes。

收集所有常驻 persona 的高共鸣 desires。派遣一个子代理：

```
DEEP PROBE FACILITATOR

The following pain points have appeared repeatedly across multiple discussion sessions (high resonance). We need to go deeper to surface enough detail to write a Claude Code skill.

HIGH-RESONANCE DESIRES:
[List each: persona_name - desire id - desire content - resonance count]

For each desire, generate:
1. A sharp probe question that would force concrete scenario description
2. The key ambiguities that must be resolved before this could become a skill

Output a JSON object:
{
  "probes": [
    {
      "desire_id": "d-xxx",
      "desire_content": "...",
      "probe_question": "...",
      "key_ambiguities": ["ambiguity 1", "ambiguity 2"]
    }
  ]
}
```

然后对 roster 中每个 persona 各派遣一个子代理（并行），将 probes 传递给每个 persona（填入 [PERSONA_JSON] 和 [PROBES_JSON]）：

```
ROUND 4 - DEEP PROBE

YOUR PERSONA:
[PERSONA_JSON]

The facilitator wants to understand these recurring pain points in more detail:
[PROBES_JSON]

For each probe that connects to YOUR experience, answer specifically. Give a real scenario from your daily work. Be as concrete as possible — the goal is to gather enough detail to build an actual tool.

Output a JSON object:
{
  "persona_name": "...",
  "probe_responses": [
    {
      "desire_id": "d-xxx",
      "is_relevant_to_me": true or false,
      "concrete_scenario": "a specific situation from my actual workflow",
      "probe_answer": "my direct answer to the probe question",
      "minimum_viable_skill": "the simplest version of a skill that would help me here"
    }
  ]
}
```

Save to `discussions/YYYY-MM-DD/round4.json`. Print: "Round 4 complete → discussions/YYYY-MM-DD/round4.json"

## Done

```
=== auto-req-simulate complete ===
Personas:  N (P permanent + T temporary)
Rounds completed:  3 (+ Round 4 if high-resonance desires triggered it)
Round 1:  discussions/YYYY-MM-DD/round1.json
Round 2:  discussions/YYYY-MM-DD/round2.json
Round 3:  discussions/YYYY-MM-DD/round3.json
Round 4:  discussions/YYYY-MM-DD/round4.json (if triggered)

常驻 persona 共鸣度已更新。临时 persona 状态未持久化。

Next step: run /auto-req-evaluate
```
