---
name: auto-req-persona
description: |
  auto_req 流水线 Stage 2。从帖子行为标签动态生成 persona roster。
  聚类标签为 3 个临时 persona，选择 2 个最相关的常驻 persona，输出 5 人 roster。
  在 /auto-req-collect 之后、/auto-req-simulate 之前运行。
  触发词: "update personas", "run persona stage", "refresh personas"
---

# auto-req-persona: 动态 Persona Roster 生成

从当天帖子数据中动态生成讨论 persona，替代固定的 4 人组。
输出 = 2 个常驻 persona（有 BDI 历史）+ 3 个临时 persona（从当天数据聚类生成）。

## Setup

获取当天日期（YYYY-MM-DD）。

检查必需的带标签用户文件：
1. `raw_users/YYYY-MM-DD-tagged.json` — **必需**，如果不存在则停止并提示先运行 `/auto-req-collect`

检查过滤后帖子（按优先级）：
1. `raw_posts/YYYY-MM-DD-filtered.json`（首选）
2. `raw_posts/YYYY-MM-DD.json`（备选）
3. `test_fixtures/raw_posts_sample.json`（离线测试）

读取 `persona_bank/` 中所有常驻 persona 文件：任何 `.json` 文件，**排除** `roster_*.json` 和 `temp_*.json`。

列出 `skill_bank/promoted/` 中所有 SKILL.md 文件（可能为空）。

打印正在使用的文件：
```
=== auto-req-persona 启动 ===
带标签用户: raw_users/YYYY-MM-DD-tagged.json（N 个用户）
过滤帖子:   raw_posts/YYYY-MM-DD-filtered.json（M 条帖子）
常驻 persona: alice.json, bob.json, carol.json, dave.json
已推广技能: K 个
```

## Phase 1: 行为标签聚类 → 3 个话题群组

使用 Agent 工具派遣一个子代理进行标签聚类。

子代理输入：
- 所有带标签用户（来自 `raw_users/YYYY-MM-DD-tagged.json`）
- 过滤帖子的前 30 条（提供上下文）

子代理提示词模板：

```
你是一个用户行为分析专家。所有输出使用中文。JSON 字段名保持英文。

输入数据：
TAGGED_USERS:
[TAGGED_USERS_JSON]

SAMPLE_POSTS (前 30 条):
[FIRST_30_POSTS_JSON]

任务：
1. 统计所有用户的 behavior_tags 频率分布
2. 分析标签之间的共现关系（哪些标签经常出现在同一用户身上）
3. 将用户聚类为 3 个话题群组，每个群组代表一种讨论视角

聚类规则：
- 每个群组至少包含 2 个用户
- 如果某个标签覆盖超过 60% 的用户，需要根据次要标签将其拆分为子群组
- 无法归类的用户（标签为 unclassified）排除在外

输出格式（仅输出 JSON，无其他文字）：
{
  "tag_distribution": {"tag_name": count, ...},
  "clusters": [
    {
      "cluster_id": "cluster_1",
      "name": "群组名称（中文，具体描述该群组特征）",
      "dominant_tags": ["主要标签1", "主要标签2"],
      "user_count": N,
      "representative_users": ["username1", "username2"],
      "need_perspective": "一句话描述该群组在讨论中可能提出的独特需求视角（中文）",
      "sample_post_themes": ["主题1", "主题2"]
    }
  ]
}
```

解析子代理返回的 JSON，保存聚类结果供后续阶段使用。

打印聚类摘要：
```
--- Phase 1: 标签聚类完成 ---
标签分布: quantitative_trader(N), defi_developer(N), ...
群组 1: [name] — [user_count] 人, 标签: [dominant_tags]
群组 2: [name] — [user_count] 人, 标签: [dominant_tags]
群组 3: [name] — [user_count] 人, 标签: [dominant_tags]
```

## Phase 2: 生成 3 个临时 Persona

使用 Agent 工具**并行派遣 3 个子代理**（每个群组一个）。

每个子代理输入：
- 对应群组的聚类信息（cluster_id, name, dominant_tags, need_perspective, sample_post_themes）
- 该群组 representative_users 的详细信息（从 tagged users 中提取）
- 这些用户的 sample_posts

子代理提示词模板（为每个群组填入 [CLUSTER_INFO], [REPRESENTATIVE_USERS_DETAIL], [SAMPLE_POSTS]）：

```
你是一个 persona 设计专家。所有输出使用中文。JSON 字段名保持英文。

群组信息:
[CLUSTER_INFO]

代表用户详情:
[REPRESENTATIVE_USERS_DETAIL]

用户帖子样本:
[SAMPLE_POSTS]

任务：
根据以上群组特征，生成一个代表该群组的临时 persona。

规则：
- persona 必须是"普通 Claude Code 用户"，不是名人或公众人物
- tech_stack 和 daily_workflow 从帖子内容推断
- behavior_signature 反映该群组的核心行为特征
- beliefs 从当天帖子中推导 2-3 条初始信念
- intentions 从 need_perspective 推导 1-2 条意图
- influence_weight 范围 0.5-0.8

输出格式（仅输出 JSON，无其他文字）：
{
  "id": "temp_[CLUSTER_ID]_[TODAY]",
  "archetype": "一句话描述（中文）",
  "tech_stack": ["tool1", "tool2"],
  "daily_workflow": "日常工作描述（中文）",
  "behavior_signature": "行为特征（中文）",
  "twitter_style": "社交媒体风格（中文）",
  "influence_weight": 0.6,
  "source_cluster": "[CLUSTER_ID]",
  "dominant_tags": ["tag1", "tag2"],
  "beliefs": [
    {"content": "从当天帖子推导的具体信念（中文）", "source": "[TODAY]", "stale": false}
  ],
  "desires": [],
  "intentions": ["从 need_perspective 推导的具体意图（中文）"],
  "is_temporary": true
}
```

将每个临时 persona 保存至 `persona_bank/temp_cluster_N_YYYY-MM-DD.json`。

打印：
```
--- Phase 2: 临时 Persona 生成完成 ---
temp_cluster_1_YYYY-MM-DD: [archetype]
temp_cluster_2_YYYY-MM-DD: [archetype]
temp_cluster_3_YYYY-MM-DD: [archetype]
```

## Phase 3: 选择 2 个最相关的常驻 Persona

使用 Agent 工具派遣一个子代理进行选择。

子代理输入：
- 今天的 3 个群组信息（name + need_perspective）
- 3 个临时 persona 摘要（id + archetype）
- 所有常驻 persona 的完整 JSON

子代理提示词模板：

```
你是一个讨论小组组建专家。所有输出使用中文。JSON 字段名保持英文。

今日话题群组:
[CLUSTERS — 每个群组的 name + need_perspective]

已生成的临时 Persona:
[TEMP_PERSONAS — 每个的 id + archetype + dominant_tags]

可选常驻 Persona:
[ALL_PERMANENT_PERSONAS — 完整 JSON]

任务：
从常驻 persona 中选择 2 个加入今天的讨论 roster。

选择规则（按优先级排序）：
1. **金融相关性前提**：persona 的 archetype 和 daily_workflow 必须与金融/加密货币分析直接相关（交易、投资、链上分析、DeFi、衍生品、宏观分析等）。非金融角色（SaaS 开发、产品管理、通用学习等）不应入选
2. 必须带来临时 persona 尚未覆盖的讨论视角
3. 优先选择有未解决需求（desires 中 resolved=false）的 persona
4. 优先选择 intentions 与今天话题有交集的 persona
5. 不要选择与某个临时 persona 高度重叠的常驻 persona

输出格式（仅输出 JSON，无其他文字）：
{
  "selected": [
    {
      "persona_id": "选中的 persona id",
      "rationale": "选择原因（中文，具体说明为什么该 persona 能补充临时 persona 的不足）"
    }
  ],
  "excluded": [
    {
      "persona_id": "未选中的 persona id",
      "reason": "排除原因（中文）"
    }
  ]
}
```

解析子代理返回结果，保存选择理由和排除原因。

打印：
```
--- Phase 3: 常驻 Persona 选择完成 ---
选中: [id1]（原因: ...）, [id2]（原因: ...）
排除: [id3]（原因: ...）, [id4]（原因: ...）
```

## Phase 4: 更新选中常驻 Persona 的 BDI 状态

对每个选中的常驻 persona（共 2 个），**按顺序**运行 3 个子代理：

### Subagent A: 更新信念 (Beliefs)

子代理提示词（填入 [PERSONA_JSON], [RAW_POSTS_JSON], [TODAY], [CURRENT_BELIEFS]）：

```
你正在更新一个加密/金融 persona 的市场信念。
所有输出使用中文。JSON 字段名保持英文。

PERSONA:
[PERSONA_JSON]

TODAY'S DATE: [TODAY]

TODAY'S MARKET POSTS:
[RAW_POSTS_JSON]

CURRENT BELIEFS (首次运行可能为空):
[CURRENT_BELIEFS]

任务：
1. 以该 persona 的视角（archetype、tech_stack、daily_workflow）审视今天的帖子
2. 添加新信念：该 persona 会关注但之前未记录的市场动态（最多 5 条）
3. 标记过期信念：底层情况已明显改变或距今超过 14 天的信念

规则：
- 信念必须具体，不能笼统（"ETH gas 费在 NFT 发售期间飙升至 50 gwei 以上" 而不是 "gas 费很贵"）
- 每次运行最多添加 5 条新信念
- 情况已解决或反转的信念标记为 stale

输出格式（仅输出 JSON 数组，无其他文字）：
[
  {
    "content": "具体的市场观察（中文）",
    "source": "[TODAY]",
    "stale": false
  }
]

包含所有现有未过期信念加上新增的。将过期信念标记为 "stale": true（保留在数组中）。
```

保存输出为新的 `beliefs` 数组。

### Subagent B: 标记已解决的需求 (Desires)

子代理提示词（填入 [PERSONA_JSON], [DESIRES_JSON], [PROMOTED_SKILLS]）：

```
你正在审查一个 persona 的痛点（desires）是否已被已有技能覆盖。
所有输出使用中文。JSON 字段名保持英文。

PERSONA:
[PERSONA_JSON]

CURRENT DESIRES（痛点列表）:
[DESIRES_JSON]

AVAILABLE PROMOTED SKILLS:
[对 skill_bank/promoted/ 中每个 SKILL.md，包含文件名 + 前 20 行]

任务：
对每个当前未解决的 desire，判断：
- 是否有已推广的技能直接解决了该需求？
- 只有技能覆盖了核心用例才算解决，仅仅相关不算

输出格式（仅输出完整 desires 数组 JSON，无其他文字）：
[
  {
    "id": "d-001",
    "content": "原始需求文本",
    "resonance": N,
    "resolved": true 或 false,
    "resolved_by": "skill_bank/promoted/filename.md 或 null",
    "first_seen": "YYYY-MM-DD"
  }
]

如果还没有 desires，输出空数组: []
```

保存输出为新的 `desires` 数组。

### Subagent C: 生成今日意图 (Intentions)

子代理提示词（填入 [PERSONA_JSON], [UNRESOLVED_DESIRES], [NEW_BELIEFS], [TODAY], [CLUSTER_INFO]）：

```
你正在为即将参加今天小组讨论的 persona 设定讨论重点。
所有输出使用中文。JSON 字段名保持英文。

PERSONA:
[PERSONA_JSON]

TODAY: [TODAY]

UNRESOLVED DESIRES（按共鸣度从高到低排序）:
[UNRESOLVED_DESIRES — desires 中 resolved=false 的，按 resonance 降序]

TODAY'S NEW BELIEFS（本次运行新增的）:
[NEW_BELIEFS — beliefs 中 source=[TODAY] 的]

TODAY'S DISCUSSION CLUSTERS（今天的话题群组）:
[CLUSTER_INFO — 3 个群组的 name + need_perspective]

任务：
生成 1-3 个意图 — 该 persona 在今天讨论中想要探索或提出的具体方向。意图应该：
- 将至少一个未解决需求与今天的新市场背景（包括群组信息）关联起来
- 足够具体，能指导 persona 在讨论中会提出什么话题
- 听起来像是这个特定的人会真正想到的，符合其 archetype

如果没有未解决需求，纯粹从新信念生成意图。
如果既没有新信念也没有未解决需求，生成一个通用的探索性意图。

输出格式（仅输出 JSON 字符串数组，无其他文字）：
["意图1", "意图2"]
```

保存输出为新的 `intentions` 数组。

### 写回 Persona

将更新后的 beliefs、desires、intentions 合并到 persona JSON 中，写回 `persona_bank/<id>.json`。

打印: "已更新 [persona.id]: [N_new] 条新信念, [N_resolved] 个需求已解决, [N] 个意图已设定"

## Phase 5: 保存 Roster

组装今日 roster 并保存至 `persona_bank/roster_YYYY-MM-DD.json`：

```json
{
  "date": "YYYY-MM-DD",
  "roster": [
    {
      "persona_id": "alice",
      "type": "permanent",
      "source": "persona_bank/alice.json"
    },
    {
      "persona_id": "bob",
      "type": "permanent",
      "source": "persona_bank/bob.json"
    },
    {
      "persona_id": "temp_cluster_1_YYYY-MM-DD",
      "type": "temporary",
      "source": "persona_bank/temp_cluster_1_YYYY-MM-DD.json"
    },
    {
      "persona_id": "temp_cluster_2_YYYY-MM-DD",
      "type": "temporary",
      "source": "persona_bank/temp_cluster_2_YYYY-MM-DD.json"
    },
    {
      "persona_id": "temp_cluster_3_YYYY-MM-DD",
      "type": "temporary",
      "source": "persona_bank/temp_cluster_3_YYYY-MM-DD.json"
    }
  ],
  "cluster_summary": [
    {
      "cluster_id": "cluster_1",
      "name": "群组名称",
      "dominant_tags": ["tag1", "tag2"],
      "user_count": N,
      "need_perspective": "需求视角描述"
    }
  ],
  "selection_rationale": {
    "selected": [
      {"persona_id": "id", "rationale": "选择原因"}
    ],
    "excluded": [
      {"persona_id": "id", "reason": "排除原因"}
    ]
  }
}
```

roster 数组中总是包含 5 个条目：2 个 permanent + 3 个 temporary。permanent 条目对应 Phase 3 选出的 2 个常驻 persona，temporary 条目对应 Phase 2 生成的 3 个临时 persona。

## 清理过期临时 Persona

列出 `persona_bank/` 中所有 `temp_*.json` 文件。从文件名解析日期（格式：`temp_cluster_N_YYYY-MM-DD.json`）。删除日期早于 7 天前的文件。

打印清理统计：
```
--- 临时 Persona 清理 ---
检查文件: N 个
已删除:   M 个（超过 7 天）
保留:     K 个
```

## Done

打印完整摘要：
```
=== auto-req-persona 完成 ===
数据来源:
  带标签用户: raw_users/YYYY-MM-DD-tagged.json（N 个用户）
  过滤帖子:   raw_posts/YYYY-MM-DD-filtered.json（M 条帖子）

标签聚类（3 个群组）:
  群组 1: [name] — [user_count] 人, 标签: [dominant_tags]
  群组 2: [name] — [user_count] 人, 标签: [dominant_tags]
  群组 3: [name] — [user_count] 人, 标签: [dominant_tags]

今日 Roster（5 人）:
  [permanent] alice  — [N] 信念, [N] 未解决需求, [N] 意图
  [permanent] bob    — [N] 信念, [N] 未解决需求, [N] 意图
  [temporary] temp_cluster_1_YYYY-MM-DD — archetype: [archetype]
  [temporary] temp_cluster_2_YYYY-MM-DD — archetype: [archetype]
  [temporary] temp_cluster_3_YYYY-MM-DD — archetype: [archetype]

已保存 → persona_bank/roster_YYYY-MM-DD.json

下一步: 运行 /auto-req-simulate
```
