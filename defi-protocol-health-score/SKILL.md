---
name: defi-protocol-health-score
description: 通过链上资金流向、协议安全基础设施（治理/预言机/清算机制）、市场结构、衍生品市场和社区情绪分析，输出 DeFi 平台级别的综合健康度评分。
version: 1.2
evolved: 2026-04-08
---

# DeFi 平台/协议健康度综合评分器

## 功能说明

当用户考虑在 DeFi 协议存入资金或在去中心化衍生品平台开仓前，自动从资金流向、协议安全基础设施、市场结构、衍生品市场、社区叙事五个维度进行量化评估，输出 0-100 的综合健康度评分和分维度明细。帮助用户在存款/开仓前完成事前平台级风险评估，尤其关注治理集中度、预言机依赖风险和清算机制健壮性。

## 适用场景

- 考虑向 DeFi 协议存入 $10K+ 资金之前
- 选择衍生品交易执行平台时（对比 Deribit vs Hyperliquid 等）
- 在新交易所注册或充值之前
- 定期审查已有持仓所在平台的健康状态
- 重大 DeFi 安全事件（如协议黑客、异常清算）发生后评估相关协议

### 触发短语
- "Aave 安全吗？我想存 $50K USDC"
- "Hyperliquid 的清算机制靠谱吗？"
- "对比一下 Deribit 和 Hyperliquid 的平台风险"
- "Chaos Labs 预言机出问题了，Aave 现在还安全吗？"
- `/defi-protocol-health aave --check governance,oracle,liquidation`

## 用户画像

### 用户一：DeFi 收益策略研究员（Carol 类型）

**角色:** DeFi yield 研究员，管理六位数规模的 DeFi 组合

**日常工作流:** 每天用 Claude Code 抓取 DefiLlama/Dune 数据对比不同协议的收益率，评估无常损失，撰写收益策略报告。每周在 Aave、Compound、Lido 等协议之间调整头寸分配。

**工具栈:** Python、Dune Analytics、DefiLlama API、Claude Code、Jupyter Notebook

**核心痛点:** 在 Aave/Chaos Labs 预言机配置纠纷（导致 $25M 错误清算）后意识到自己完全没有工具评估"我存款所在协议的治理健康度"——只能被动依赖 Twitter 上的 DeFi 治理讨论来感知风险，永远是事后知道而非事前预防。

**用户故事:** "As Carol, I want to assess Aave's governance health (oracle dependencies, voting concentration, key-person risk) before depositing $100K USDC, so that I can catch governance risks before they translate into erroneous liquidations or fund loss."

---

### 用户二：加密衍生品交易员（Dave 类型）

**角色:** 期权/永续合约交易员，专注 BTC/ETH 期权 Greeks 和资金费率套利

**日常工作流:** 用 Claude Code 监控 Deribit IV、Greeks 变化、资金费率，分析期权链数据寻找错误定价，执行 delta 对冲和波动率套利。同时考察去中心化永续合约平台（如 Hyperliquid）作为 Deribit 的补充或替代。

**工具栈:** Python、Deribit API、Greeks.live、TradingView、Claude Code

**核心痛点:** 亲眼目睹 Hyperliquid 上一名交易员因清算引擎问题导致 $100M 仓位缩水至 $900，但他没有任何工具系统性地对比去中心化 vs 中心化平台的清算机制健壮性、极端行情下的 slippage 数据、保险基金覆盖率。选择平台时完全依赖 Twitter 上的事后复盘帖。

**用户故事:** "As Dave, I want to compare Hyperliquid vs Deribit on liquidation engine stress performance, slippage in extreme conditions, and insurance fund coverage, so that I can make an informed platform choice before committing a $20K BTC leveraged position."

---

## 输入参数

| 参数 | 必填 | 说明 | 默认值 |
|------|------|------|--------|
| 协议/平台名称 | 是 | DeFi 协议或交易平台名称 | — |
| check | 否 | 指定评估维度（governance, oracle, liquidation, fund_flow, market_structure, derivatives, sentiment） | 全部 |
| compare | 否 | 对比协议名称 | 无 |

## 执行步骤

### Step 1: 识别目标协议/平台并获取基本信息
调用 `ant_market_sentiment(query_type='topics_list')` 搜索用户指定的协议/平台相关话题。
对匹配话题调用 `ant_market_sentiment(query_type='topic_detail', topic_id=<id>)` 获取话题详情、情绪评分。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 获取最近帖子，识别安全事件或治理争议。
调用 `ant_market_sentiment(query_type='topic_news', topic_id=<id>)` 获取近期新闻，筛选安全/黑客/治理类新闻。

### Step 2: 评估代币集中度和资金流向（大户风险）
调用 `ant_market_sentiment(query_type='coins_list')` 搜索协议治理代币。
调用 `ant_market_sentiment(query_type='coin_detail', coin_id=<id>)` 获取代币基本面。
调用 `ant_fund_flow(query_type='exchange_whale_ratio', coin=<代币>)` 获取鲸鱼交易占比。
调用 `ant_fund_flow(query_type='exchange_netflow', coin=<代币>)` 获取资金流向。
调用 `ant_smart_money(query_type='holdings', coin=<代币>)` 获取聪明钱持仓分布。

### Step 2.5: 评估协议安全基础设施（治理、预言机、清算机制）

**这是核心步骤，解答"这个协议/平台本身的运作机制是否安全可靠"。**

**治理集中度评估:**
调用 `ant_protocol_ecosystem(query_type='protocol_info', protocol=<名称>)` 获取协议治理架构信息（多签配置、核心开发团队、关键服务商依赖）。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 搜索关键词 "governance", "multisig", "admin key", "upgradeable proxy", "DAO vote" 的相关讨论，识别治理争议和集中度风险。
从 Step 1 的新闻数据中筛选治理提案、关键服务商变更、多签签名人变动相关新闻。

评估指标（基于可获取信号，输出判断而非精确数字）：
- **关键服务商依赖度**: 协议是否重度依赖单一风险/预言机服务商（如 Chaos Labs）？（高/中/低）
- **治理提案争议度**: 近期是否存在有争议的治理提案或治理紧急操作？（有/无）
- **多签集中度信号**: 社区讨论中是否出现对多签配置或管理员权限的质疑？（有/无）

**预言机依赖风险评估:**
调用 `ant_market_sentiment(query_type='topic_news', topic_id=<id>)` 筛选 "oracle", "price feed", "Chainlink", "Pyth", "manipulation" 关键词相关新闻。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 识别预言机相关安全讨论或历史事件提及。

评估指标：
- **预言机多样性**: 协议是否依赖单一预言机来源？（单一/多源/不明）
- **历史预言机异常事件**: 近12个月内是否发生过预言机配置错误或喂价异常导致的资金损失？（有/无/不明）

**清算机制健壮性评估（适用于借贷协议和衍生品平台）:**
调用 `ant_futures_liquidation(query_type='liq_heatmap', coin=<相关代币>)` 查看清算热力图，了解大额清算聚集区域。
调用 `ant_futures_market_structure(query_type='futures_open_interest', coin=<代币>)` 获取当前未平仓合约规模（评估清算风险暴露）。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 搜索 "liquidation cascade", "insurance fund", "bad debt", "socialized loss" 关键词，识别历史清算事件讨论。

评估指标：
- **历史异常清算事件频率**: 近期是否发生过非正常清算（如 $25M 错误清算、$100M 单仓位被清算至 $900）？（高/低/无）
- **保险基金/坏账覆盖**: 从社区讨论和新闻推断协议的损失吸收机制是否健全？（健全/有缺口/不明）
- **清算引擎集中度**: 平台清算是否过度依赖单一市场做市商或引擎？（高集中/分散/不明）

### Step 3: 评估市场微观结构健康度
调用 `ant_market_indicators(query_type='orderbook_ask_bids', coin=<代币>)` 获取订单簿深度。
调用 `ant_market_indicators(query_type='coinbase_premium', coin=<代币>)` 获取 Coinbase 溢价（机构兴趣代理）。
调用 `ant_market_indicators(query_type='atr', coin=<代币>)` 获取波动率。

### Step 4: 评估期货/衍生品市场健康度（如协议有衍生品）
调用 `ant_futures_market_structure(query_type='futures_open_interest', coin=<代币>)` 获取未平仓合约。
调用 `ant_futures_market_structure(query_type='futures_funding_rate', coin=<代币>)` 获取资金费率。
调用 `ant_futures_market_structure(query_type='futures_long_short_ratio', coin=<代币>)` 获取多空比。

**纯借贷协议（无衍生品数据）**: 标注此维度"不适用"，调整总分权重（资金流向 +7分 → 32分，市场结构 +8分 → 33分，协议安全 +10分 → 35分）。

### Step 5: 社区情绪辅助分析
调用 `ant_market_sentiment(query_type='topic_creators', topic_id=<id>)` 获取话题主要创作者。
基于 Step 1 的帖子和新闻，提取补充信号（协议安全基础设施维度的二手数据来源）:
- 安全事件提及频率（已在 Step 2.5 中用于协议安全维度）
- 治理争议提及频率（已在 Step 2.5 中使用）
- "rug pull"/"scam"/"hack"/"oracle attack" 等风险关键词出现次数
- 正面评价 vs 负面评价比例（排除已被 Step 2.5 分类的安全事件讨论后的纯社区情绪）

### Step 6: 综合评分
将各维度评分汇总为综合健康度评分:

**标准评分（协议有衍生品数据）:**
- **资金流向健康度** (0-20分): 基于鲸鱼占比、资金净流向、聪明钱持仓
- **协议安全基础设施** (0-30分): 基于治理集中度、预言机依赖风险、清算机制健壮性 ← **核心维度**
- **市场结构健康度** (0-20分): 基于订单簿深度、波动率、Coinbase 溢价
- **衍生品市场健康度** (0-20分): 基于 OI 集中度、资金费率极端性、多空比偏离
- **社区/叙事情绪** (0-10分): 基于纯社区情绪（排除安全事件）、讨论多样性

**调整评分（纯借贷协议，无衍生品数据）:**
- **资金流向健康度** (0-25分)
- **协议安全基础设施** (0-40分) ← 权重上调，因为借贷协议治理/预言机风险更集中
- **市场结构健康度** (0-25分)
- **社区/叙事情绪** (0-10分)

总分 = 各维度之和 (0-100)。

**字母评级:**
- A: 80-100 (低风险)
- B: 65-79 (中低风险)
- C: 50-64 (中等风险，谨慎)
- D: 35-49 (高风险)
- F: 0-34 (极高风险，不建议)

## 输出格式

```markdown
# DeFi 协议/平台健康度评估

## 目标: [协议/平台名称]
## 评估时间: [YYYY-MM-DD]
## 综合健康度评分: [X]/100 [评级: A/B/C/D/F]

---

## 评分明细

| 维度 | 得分 | 等级 | 关键发现 |
|------|------|------|---------|
| 资金流向健康度 | [X]/20 | [等级] | [1句话总结] |
| 协议安全基础设施 | [X]/30 | [等级] | [1句话总结，含治理/预言机/清算风险] |
| 市场结构健康度 | [X]/20 | [等级] | [1句话总结] |
| 衍生品市场健康度 | [X]/20 | [等级，或"不适用（已调整权重）"] | [1句话总结] |
| 社区/叙事情绪 | [X]/10 | [等级] | [1句话总结] |

---

## 协议安全基础设施详情

### 治理集中度
[评估结论：高/中/低集中度，1-2句理由，包含近期治理事件]

### 预言机依赖风险
[评估结论：单一/多源/不明，1-2句理由，包含历史异常事件]

### 清算机制健壮性
[评估结论：健全/有缺口/不明，1-2句理由，包含历史异常清算事件]

---

## 其他维度详情
[各维度 1-2 句分析]

## 风险警告
[0-3 个最重要的风险项，优先列出协议安全基础设施相关风险]

## 与同类协议对比
[对比表格，含协议安全基础设施维度]

## 适用性建议
- 大额存款 (>$50K): [适合/谨慎/不建议] — [1句理由]
- 杠杆交易: [适合/谨慎/不建议] — [1句理由]
- 长期持仓: [适合/谨慎/不建议] — [1句理由]

## 局限性声明
本评估基于链上数据、市场指标和社区情绪的量化/定性分析。协议安全基础设施维度的治理和预言机评估主要依赖社区讨论和新闻信号，无法替代智能合约代码审计、多签签名人背景调查、预言机节点运行状态的直接核查。本报告不构成投资建议。
```

## 使用示例

**输入:**
```
Hyperliquid 安全吗？我想在上面开一个 $20K 的 BTC 多单。
```

**输出示例:**

```
# DeFi 协议/平台健康度评估

## 目标: Hyperliquid
## 评估时间: 2026-04-08
## 综合健康度评分: 51/100 [评级: C]

## 评分明细

| 维度 | 得分 | 等级 | 关键发现 |
|------|------|------|---------|
| 资金流向健康度 | 13/20 | C | 鲸鱼交易占比偏高，近7日净流入但波动大 |
| 协议安全基础设施 | 14/30 | D | 清算引擎历史存在极端清算事件（$100M→$900），治理高度中心化，无公开多签配置 |
| 市场结构健康度 | 12/20 | C | 订单簿深度在极端行情下不足，流动性主要由少数做市商提供 |
| 衍生品市场健康度 | 9/20 | D | 资金费率偶发极端，OI 集中度高，极端行情下清算级联风险 |
| 社区/叙事情绪 | 3/10 | D | 社区对平台安全性讨论存在较多负面声音，主要围绕清算机制 |

## 协议安全基础设施详情

### 治理集中度
**高集中度**。Hyperliquid 为非 DAO 结构，核心团队控制协议升级权。未发现公开的多签配置信息，团队单方面决策能力较强。

### 预言机依赖风险
**单一/不明**。具体预言机配置信息在社区讨论中不透明。极端行情下的价格喂价准确性历史存疑。

### 清算机制健壮性
**有缺口**。有记录的 $100M 单仓位清算至 $900 事件，说明在极端行情下清算引擎存在滑点/流动性不足风险。保险基金规模和覆盖率未公开透明。

## 风险警告
1. **清算机制极端风险** (高): 历史记录证明极端行情下单仓位可能被清算至接近零值，$20K 杠杆仓位面临此风险。
2. **治理透明度不足** (中): 非 DAO 结构、无公开多签，协议变更缺乏社区监督机制。
3. **流动性深度风险** (中): 订单簿深度不足，大额仓位在极端行情下 slippage 风险高。

## 与同类协议对比

| 指标 | Hyperliquid | Deribit | dYdX |
|------|-------------|---------|------|
| 综合评分 | 51/100 | 72/100 | 63/100 |
| 协议安全基础设施 | 14/30 | 24/30 | 20/30 |
| 治理透明度 | 低 | 中 | 高（DAO） |
| 历史异常清算 | 有 | 无重大 | 有（小规模） |

## 适用性建议
- 大额存款 (>$50K): **不建议** — 清算风险和治理不透明度不适合大额长期资金
- 杠杆交易: **谨慎** — 可用于小仓位（≤$5K），必须设严格止损，避免高杠杆
- 长期持仓: **不建议** — 平台安全基础设施评分过低，不适合持续持仓
```

## 反模式

1. **不要给出"绝对安全"的结论**: 任何 DeFi 协议都有风险，评分是相对指标而非安全保证。
2. **不要将协议安全基础设施维度降级为"补充说明"**: 治理集中度、预言机依赖、清算机制是核心风险维度，权重最高（30分），必须首先评估。
3. **不要在无衍生品数据的协议上强行评分衍生品维度**: 标注"不适用"并按调整权重重新分配分值。
4. **不要将鲸鱼占比高直接等同于风险**: 新协议鲸鱼占比天然偏高，需结合行为模式综合判断。
5. **不要因为无法直接访问链上多签数据就跳过治理评估**: 利用社区讨论、新闻和 MCP 可获取的信号做定性判断，明确标注数据来源和置信度。
6. **不覆盖**: 智能合约源代码审计、多签签名人实名背景调查、预言机节点实时状态、协议内部运营风险。
7. **数据覆盖限制**: 小型/新兴协议在 MCP 中可能缺少数据，应标注数据不完整并降低评分置信度。
