---
name: prediction-market-divergence-detector
description: 对比预测市场赔率与传统市场隐含概率及链上资金流向，检测背离并评估操纵迹象。
---

# 预测市场赔率背离检测器

## 功能说明

对比预测市场赔率（Polymarket、Kalshi）与传统市场隐含概率（联邦基金期货、国债收益率）及链上资金流向数据，检测显著背离、评估赔率变动是信息驱动还是流动性驱动、判断是否存在操纵迹象，并输出综合可信度评分（1-10 分）。

## 用户画像与使用场景

### 用户画像 1：加密衍生品交易员（Dave 型）

**角色**: 全职加密期权 / 永续合约交易员，有 2-5 年衍生品经验

**日常工作流**: 每天用 Claude Code 监控 Deribit 期权链（IV term structure、put/call skew）、Binance/Bybit 资金费率、以及 Polymarket 重大宏观事件赔率。遇到地缘事件或 FOMC 日，会手动对比 Polymarket 赔率和 Deribit 短期期权 IV 隐含的概率分布，寻找跨市场错误定价机会。

**工具栈**: Python、Deribit API、Greeks.live、Polymarket、Claude Code

**痛点**: 每次需要在地缘事件发生后的关键 30 分钟窗口内完成"Polymarket 赔率 → Deribit IV → 是否有 mispricing"的三步分析，但手动流程需要 40 分钟，等分析完市场已重新定价。更危险的是，无法快速判断 Polymarket 赔率的快速跳动是真实信息流入还是大单操纵，曾经差点基于一笔 $150K 大单推动的假信号做出错误调仓。

**用户故事**: As a derivatives trader, I want to instantly cross-check a Polymarket odds spike against CME implied probability and on-chain flow data, so that I can determine whether the move represents genuine information or low-liquidity manipulation before committing to a position.

---

### 用户画像 2：加密-股票跨市场散户交易者（Bob 型）

**角色**: 有一定金融知识的散户投资者，同时持有加密资产和 A 股/美股，用宏观事件赔率辅助仓位调整决策

**日常工作流**: 每逢 FOMC、非农、关税新闻等宏观事件，会参考 Polymarket 赔率和 CME FedWatch 概率来决定是否调仓。已经开始把 Polymarket 赔率当作"快速锚点"，但对数据可靠性存疑——不知道某个赔率背后的流动性有多薄、是否有人在操纵。

**工具栈**: CoinGlass、TradingView、Polymarket、Claude Code（入门使用者）

**痛点**: 上个月看到 Polymarket "美联储 6 月维持利率不变" 赔率从 62% 跳到 78%，差点立刻减仓。事后发现是一笔 $15 万大单单方面推动，CME 期货同期只从 65% 变到 68%。如果当时只看 Polymarket 就行动，会基于噪音信号做出重大调仓。需要一个工具能自动对比两个来源，当出现背离时告诉他"这次跳动有没有传统市场支撑"。

**用户故事**: As a retail macro trader, I want to see a side-by-side comparison of Polymarket odds vs. CME-implied probability with a liquidity-depth warning, so that I can avoid acting on odds movements that are driven by a single large order rather than real consensus shifts.

---

## 适用场景

用户提到预测市场赔率并打算据此做交易决策，或注意到赔率出现快速变动想知道是否可信。

**触发短语**: "Polymarket 上利率不变概率87%，可信吗", "预测市场赔率跟 CME 期货差了10个百分点", "Polymarket 这个赔率是不是被操纵了", "帮我交叉验证一下这个事件的各来源概率", "Polymarket 赔率突然跳了，是真实信号还是大单操纵？", "Deribit IV 跟 Polymarket 赔率方向不一致，哪个更可信？"

## 输入参数

| 参数 | 必须 | 默认值 | 说明 |
|------|------|--------|------|
| 事件名称 | 是 | - | 要检测的预测市场事件 |
| Polymarket 赔率 | 是 | - | 用户提供的预测市场当前赔率 |
| CME/传统市场概率 | 否 | - | 用户提供的传统市场隐含概率（如 CME FedWatch） |
| 赔率近期变动 | 否 | - | 过去 24h 最高/最低值 |
| 流动性信息 | 否 | - | 订单簿深度、近期大额交易 |
| Deribit IV（加密事件适用） | 否 | - | 加密资产相关事件时，用户提供的 Deribit 短期期权隐含波动率或 IV term structure 变化 |

**注意**: Polymarket 赔率、CME 期货隐含概率和 Deribit IV 均不在 MCP 工具范围内，需用户提供。当事件涉及加密资产价格方向（如地缘冲突对 BTC 的影响），强烈建议同时提供 Deribit IV 数据作为第三个独立锚点。

## 执行步骤

### 第一步：获取传统市场隐含概率锚点

```
ant_macro_economics(query_type="federal_funds_rate")
ant_macro_economics(query_type="treasury_yield")
ant_macro_economics(query_type="expectation")
```

记录为"传统市场基准"。

### 第二步：获取链上资金流向数据

```
ant_fund_flow(query_type="exchange_whale_ratio")
ant_fund_flow(query_type="exchange_netflow")
```

```
ant_smart_money(query_type="netflows")
ant_smart_money(query_type="dex_trades")
```

记录最近 24h 鲸鱼净流入/流出和大额 DEX 交易。

### 第三步：获取市场情绪与技术指标

```
ant_market_sentiment(query_type="topics_list")
ant_market_sentiment(query_type="topic_detail", topic="[相关事件话题]")
ant_market_sentiment(query_type="topic_posts", topic="[相关事件话题]")
```

```
ant_market_indicators(query_type="rsi")
ant_market_indicators(query_type="boll")
```

### 第四步：获取期货市场结构数据

```
ant_futures_market_structure(query_type="futures_funding_rate_history")
ant_futures_market_structure(query_type="futures_oi_aggregated")
ant_futures_market_structure(query_type="futures_long_short_global")
```

### 第五步：整合用户提供的预测市场数据

将用户提供的 Polymarket 赔率、CME 概率、流动性信息整合到分析中。

### 第六步：计算背离分析

1. **概率偏差幅度** = |预测市场赔率 - 传统市场隐含概率|
2. **方向一致性检查**: 期货资金费率方向 vs 预测市场赔率方向 vs 宏观数据趋势
3. **鲸鱼活动相关性**: 净流入/流出时间窗口是否与赔率变动吻合
4. **流动性调整评分**: 赔率变动幅度 / 推动该变动所需资金量
5. **衍生品IV交叉验证**（仅加密事件适用）: 若用户提供了 Deribit IV 数据，计算 IV 隐含的尾部概率分布与 Polymarket 赔率的一致性。IV 和 Polymarket 同向变化（如 IV 急升 + 负面事件赔率上升）支持赔率可信；若 IV 稳定但 Polymarket 赔率大幅跳动，倾向于将跳动归因于流动性驱动而非信息驱动。
6. **综合可信度评分**: 基于以上四项（或五项，如有 IV 数据），1-10 分。每个一致信号加分，每个背离信号减分。

### 第七步：生成历史校准参考

检索同类历史事件中预测市场赔率 vs 实际结果的偏差统计。

## 输出格式

```markdown
# 预测市场赔率背离检测报告

**事件:** [事件名称]
**报告时间:** [YYYY-MM-DD HH:MM UTC]

## 多源概率对比矩阵

| 来源 | 概率/方向 | 更新时间 | 备注 |
|------|----------|----------|------|
| Polymarket | [X]% | [时间] | [用户提供] |
| CME 期货隐含概率 | [Y]% | [时间] | [用户提供] |
| Deribit IV 隐含方向 | [偏多/偏空/中性] | [时间] | [用户提供，加密事件适用] |
| 期货资金费率方向 | [正/负/中性] | [时间] | [MCP数据] |
| 期货多空比 | [多头占比]% | [时间] | [MCP数据] |
| 鲸鱼净流向 | [流入/流出] [金额] | [时间] | [MCP数据] |
| 宏观基准 | [联邦基金利率等] | [时间] | [MCP数据] |

## 背离分析

**概率偏差幅度:** [X] 个百分点
**方向一致性:** [一致 / 部分分歧 / 显著分歧]
**综合可信度评分:** [N]/10

### 背离归因

- **信息驱动 vs 流动性驱动:** [分析]
- **鲸鱼活动关联:** [是否检测到异常大额交易]
- **传统市场确认度:** [CME 期货/利率数据是否支撑赔率变动方向]
- **衍生品 IV 确认度:** [如有 Deribit IV 数据：IV term structure 变化是否与 Polymarket 赔率方向一致]

## 风险提示

- [如检测到操纵迹象:] 警告：赔率变动可能由低流动性下的大单驱动
- [如检测到反身性风险:] 注意：赔率可能已被自身影响的交易行为推高/拉低
- **不确定性声明:** 预测市场赔率和 CME 数据由用户提供，未经独立验证

## 历史校准参考

过去 [N] 次同类事件:
- 平均绝对偏差: [X] 个百分点
- 赔率 > 80% 的事件最终发生率: [Y]%
- 最大校准失败案例: [描述]

## 行动建议

- **偏差 < 5%:** 基本一致，赔率可作为参考锚点
- **偏差 5-15%:** 值得关注的分歧，建议深入调查后再决策
- **偏差 > 15%:** 显著背离，不建议依赖单一来源
```

## 使用示例

**用户**: "Polymarket 上'美联储6月维持利率不变'赔率87%，CME FedWatch 显示78%，差了9个百分点。帮我验证一下这个偏差是否正常。"

→ 执行全部步骤。获取宏观基准和链上数据，整合用户提供的赔率数据，计算背离，输出可信度评分和历史校准参考。

## 反模式

1. **不要把预测市场赔率当作唯一真相。** 赔率是信号之一——与传统市场一致时可信度更高，背离时需要更多调查。
2. **不要忽略流动性深度。** 87% 赔率 + $50K 流动性的信息价值远低于 82% 赔率 + $5M 流动性。
3. **不要假设历史校准能预测未来。** 预测市场在常规事件上校准良好，但黑天鹅事件上可能严重失准。
4. **不要把背离等同于套利机会。** 背离可能源于合理的信息差、流动性差异或时间窗口不同步。
5. **不要用此技能替代对具体事件的深度研究。** 本技能提供"信号可信度评估"，不提供"事件预测"。
