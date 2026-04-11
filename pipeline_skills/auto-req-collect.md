---
name: auto-req-collect
description: auto_req 流水线 Stage 1。从 ant_market_sentiment MCP 获取热门金融话题，抓取帖子，过滤无关内容，分层抽样用户并标注行为标签。输出 raw_posts/、raw_users/。运行后执行 /auto-req-persona。
---

# auto-req-collect: 数据收集

从 X/Twitter 收集热门金融/加密帖子，过滤无关内容，分层抽样用户并标注行为标签。

输出中所有描述性文字使用中文，JSON 字段名保持英文。

## Step 1a: 收集热门金融话题 + 帖子

### 1a-1: 获取热门金融话题

调用：
```
ant_market_sentiment(query_type="topics_list")
```

从返回列表中筛选金融相关 topic，分为两个优先级：

**Tier 1（加密/DeFi 核心，优先选取）**：名称包含以下关键词：
`bitcoin, btc, eth, crypto, defi, nft, token, coin, solana, stablecoin, liquidat, perp, altcoin, meme, blockchain, web3, dex, layer, protocol, prediction market`

**Tier 2（传统金融/宏观）**：名称包含以下关键词但不属于 Tier 1：
`stock, trade, invest, fund, forex, macro, inflation, fed, rate, future, option, market`

**Tier 3（地缘政治/AI科技 — 对市场有影响的泛金融话题）**：名称包含以下关键词但不属于 Tier 1/2：
`oil, gold, commodit, geopolit, tariff, sanction, chip, semiconductor, artificial intelligence, anthropic, open ai, energy`

注意：`ai` 作为关键词太短会误匹配（如 dubai, spain），必须使用**完全匹配**（话题名称 == "ai"）或检查名称以 "ai" 开头/结尾且前后是空格或边界。

**选取规则：**
- 先从 Tier 1 按 `interactions_24h` 降序取，最多取 6 个
- 再从 Tier 2 补充，最多取 2 个
- 再从 Tier 3 补充，最多取 2 个（总共 10 个）
- 如果某 Tier 不足配额，剩余名额按 Tier 1 → Tier 2 → Tier 3 顺序补充

**排除** 名称包含以下的 topic（太泛，噪声多）：
`stocks consumer cyclical, stocks consumer defensive, stocks communication services`

打印最终 10 个话题名称、interactions_24h 和所属 Tier。

### 1a-2: 抓取每个话题下的帖子

对每个选出的话题，调用：
```
ant_market_sentiment(query_type="topic_posts", topic="<topic名称>")
```

从每个话题取前 100 条帖子（按 post_created 降序）。**只保留 post_type 为 "tweet" 的帖子**。跨话题去重（同一 id 只保留一次），合并后最多保留 500 条。

获取当天日期（YYYY-MM-DD），保存至 `raw_posts/YYYY-MM-DD.json`：

```json
[
  {
    "id": "原始 id 字段",
    "text": "post_title 字段",
    "author": "creator_name 字段",
    "author_id": "creator_id 字段",
    "url": "post_link 字段",
    "topic": "来源话题名称",
    "followers": "creator_followers 字段（数字）",
    "sentiment": "post_sentiment 字段（数字）",
    "collected_at": "ISO 8601 时间戳（由 post_created Unix 时间戳转换）"
  }
]
```

打印: "已收集 N 条帖子，来自 M 个话题 → raw_posts/YYYY-MM-DD.json"

## Step 1b: 内容相关性过滤

对 `raw_posts/YYYY-MM-DD.json` 中的帖子进行加密/金融相关性过滤，剔除无关内容。

### 1b-1: 派遣子代理执行过滤

使用 Agent 工具派遣一个子代理，传入所有帖子的 `id` 和 `text` 字段。子代理按以下规则判定每条帖子是否相关：

**保留标准（必须同时满足两条）：**

条件 A — 帖子主题属于以下之一：
- 加密货币交易（BTC/ETH/山寨币价格、技术分析、交易策略、持仓变化）
- DeFi 协议（收益率、TVL、流动性、DEX 交易量、协议收入）
- 链上数据（鲸鱼动向、交易所资金流、持仓地址变化、UTXO 分析）
- 衍生品市场（期货持仓、资金费率、清算数据、期权 Greeks）
- 宏观经济**对币圈/股市的直接影响**（美联储利率决议→BTC 价格影响、CPI→风险资产影响）
- 地缘政治**对市场的影响**（关税政策→供应链股、制裁→加密资产流动、战争→避险资产）
- AI/科技**对投资和市场的影响**（AI 概念股/代币走势、算力成本→挖矿收益、AI 公司估值分析、GPU 供应链对相关标的的影响）
- ETF/机构动向（BTC ETF 资金流、企业 BTC 国库、灰度持仓）
- 代币经济学（供应量变化、解锁时间表、通缩机制、staking 收益）
- 量化/算法交易（策略回测、信号系统、套利机会）
- MEV/安全（闪电贷攻击、合约漏洞、审计结果）

条件 B — 帖子包含**分析性内容**（至少满足一项）：
- 包含数据/数字（价格、百分比、金额、倍数）
- 包含因果推理（"因为...所以..."、"如果...则..."）
- 包含策略建议（"应该做多/做空"、"关注这个指标"）
- 包含对比分析（"A vs B"、"历史类比"）
- 包含风险提示或反面观点

**排除（即使主题相关也排除）：**
- 纯新闻标题无分析（"X 公司股价跌了" — 除非附带分析为什么跌、意味着什么）
- 产品发布/更新公告（"Tesla 提高了 Model S 售价" — 除非讨论对股价/市场的影响）
- 纯体育/娱乐/政治（无论是否提及金融平台）
- 广告/推广/空投诈骗
- 无实质内容的短句（"看涨"、"HODL"、"LFG" 等无分析价值的帖子）
- 纯科技产品新闻（SaaS 产品发布、手机评测 — 除非讨论对相关股票/代币/市场的投资影响）

子代理返回所有相关帖子的 `id` 列表。

### 1b-2: 保存过滤结果

根据子代理返回的 id 列表，从 `raw_posts/YYYY-MM-DD.json` 中提取对应帖子，保存至 `raw_posts/YYYY-MM-DD-filtered.json`（schema 与原始文件完全相同）。

打印过滤统计：
```
=== 内容过滤统计 ===
原始帖子数:   N
过滤后帖子数: M
过滤率:       X%
已保存 → raw_posts/YYYY-MM-DD-filtered.json
```

## Step 1c: 分层用户抽样 + 行为标签

本步骤取代旧的"按粉丝数取前 30"方式，改为分层抽样 + 行为标注。

### 1c-0: 兼容性保留 — 旧格式用户文件

对每个选出的话题，调用：
```
ant_market_sentiment(query_type="topic_creators", topic="<topic名称>")
```

合并所有话题的创作者，按 followers 降序去重，取前 30 个。保存至 `raw_users/YYYY-MM-DD.json`（旧格式，向后兼容）：

```json
[
  {
    "username": "creator_name 字段",
    "creator_id": "creator_id 字段",
    "display_name": "creator_display_name 字段（如有）",
    "bio": "creator_bio 或 description 字段（如有）",
    "followers": "creator_followers 字段",
    "topics": ["该创作者出现在哪些话题下"],
    "influence_level": "high（followers > 10000）/ medium（1000-10000）/ low（< 1000）",
    "source_post_ids": ["从 raw_posts 中找到该 creator 的帖子 id"]
  }
]
```

打印: "已保存 N 个用户资料（旧格式）→ raw_users/YYYY-MM-DD.json"

### 1c-1: 从过滤帖子中提取作者并分层抽样

读取 `raw_posts/YYYY-MM-DD-filtered.json`，提取所有唯一作者（基于 `author` + `author_id` 去重）。每个作者收集其 `followers` 数（取该作者所有帖子中的最大值）。

按粉丝数分三个层级：
- **高影响力（high）**：followers > 100,000 — 配额 10 人
- **中影响力（medium）**：1,000 ≤ followers ≤ 100,000 — 配额 10 人
- **低影响力（low）**：followers < 1,000 — 配额 10 人

**配额再分配规则：**
- 如果某层级实际人数不足配额，将剩余名额分配给相邻层级（优先分配给中层级，再分配给另一层级）
- 总人数上限 30 人
- 每个层级内部按帖子数量降序排列（帖子越多越优先），帖子数相同时按 followers 降序

### 1c-2: 行为标签标注

使用 Agent 工具派遣子代理，为每个抽样用户标注 1-3 个行为标签。

子代理接收每个用户的以下信息：
- `username`
- `followers`
- 该用户在 `raw_posts/YYYY-MM-DD-filtered.json` 中的所有帖子文本

**可用标签列表（每人选 1-3 个最匹配的）：**
- `quantitative_trader` — 讨论算法交易、量化策略、回测
- `defi_developer` — 讨论 DeFi 协议开发、智能合约、TVL
- `on_chain_analyst` — 分析链上数据、鲸鱼追踪、地址分析
- `macro_analyst` — 讨论宏观经济、利率、通胀对市场影响
- `retail_speculator` — 讨论短线交易、追涨杀跌、散户情绪
- `security_researcher` — 讨论安全漏洞、MEV、审计
- `product_builder` — 讨论 Web3/加密产品开发、用户体验
- `prediction_market_user` — 使用或讨论预测市场（Polymarket 等）
- `learner` — 提问、寻求学习资源、新手入门内容
- `content_creator` — 制作教程、分析视频、长文解读
- `institutional_trader` — 讨论机构级交易、OTC、大额头寸

如果用户帖子内容过于稀少或模糊无法判断 → 标注为 `unclassified`。

### 1c-3: 保存带标签的用户数据

将分层抽样结果 + 行为标签保存至 `raw_users/YYYY-MM-DD-tagged.json`：

```json
[
  {
    "username": "用户名",
    "author_id": "用户 ID",
    "display_name": "显示名称（如有）",
    "followers": 12345,
    "influence_level": "high/medium/low",
    "topics": ["该用户帖子涉及的话题列表"],
    "behavior_tags": ["tag1", "tag2"],
    "source_post_ids": ["该用户在过滤帖子中的帖子 id 列表"],
    "sample_posts": ["代表性帖子文本1", "代表性帖子文本2"]
  }
]
```

**字段说明：**
- `display_name`：若 MCP 返回的 topic_creators 数据中有该用户，则取 `creator_display_name`；否则使用 `username`
- `topics`：该用户帖子所属的话题名称列表（去重）
- `sample_posts`：从该用户的过滤帖子中选取最多 3 条代表性文本（优先选不同话题的帖子）

打印: "已保存 N 个带标签用户 → raw_users/YYYY-MM-DD-tagged.json"

## Done

打印完整摘要：
```
=== auto-req-collect 完成 ===
原始帖子:         N 条  →  raw_posts/YYYY-MM-DD.json
过滤后帖子:       M 条  →  raw_posts/YYYY-MM-DD-filtered.json（过滤率 X%）
用户资料（旧格式）: P 个  →  raw_users/YYYY-MM-DD.json
带标签用户:       Q 个  →  raw_users/YYYY-MM-DD-tagged.json

--- 行为标签分布 ---
quantitative_trader:    N 人
defi_developer:         N 人
on_chain_analyst:       N 人
macro_analyst:          N 人
retail_speculator:      N 人
security_researcher:    N 人
product_builder:        N 人
prediction_market_user: N 人
learner:                N 人
content_creator:        N 人
institutional_trader:   N 人
unclassified:           N 人

--- 影响力层级分布 ---
high (>100k):   N 人
medium (1k-100k): N 人
low (<1k):      N 人

下一步: 运行 /auto-req-persona
```
