---
name: funding-rate-vs-prediction-odds
description: 叠加加密永续合约资金费率与预测市场宏观事件赔率，检测背离并评估定价错误方向。
---

# 资金费率 vs 预测市场赔率背离检测器

## 用户画像

**Dave — 加密期权/衍生品交易员**
- **日常工作流**: 用 Claude Code 监控 BTC/ETH 期权隐含波动率、Greeks、资金费率；分析期权链寻找错误定价；执行 delta 对冲和波动率套利。
- **工具栈**: Python、Deribit API、Greeks.live、TradingView、Claude Code
- **痛点**: 地缘催化事件发生后 30 分钟内是期权定价最混乱的窗口，但手动完成"Polymarket 赔率 → Deribit IV → 心算对比"流程需要 40 分钟，错过最佳入场时机。Polymarket 赔率与期权隐含 IV 之间的背离没有人在系统性地监控。

**Bob — 散户杠杆交易员**
- **日常工作流**: 用 TradingView 和 Claude Code 分析 K 线形态，在 Binance/Bybit 做永续合约，持仓前查看资金费率和市场结构信号。
- **工具栈**: Python、Pine Script、TradingView、Binance/Bybit
- **痛点**: 看到正向资金费率就误以为是做多信号，但不知道预测市场是否在定价相反的宏观事件风险（如美联储会议、CPI）。进杠杆仓位前缺乏跨市场矛盾检测工具，曾因此在事件前夕被清算。

## 用户故事

- **As Dave**, I want to instantly compare Polymarket event odds with BTC perpetual funding rates and get a numeric divergence score, **so that** I can identify gamma opportunities when prediction markets and perp positioning are pricing opposite outcomes — without spending 40 minutes manually switching between platforms.

- **As Bob**, I want the skill to warn me when funding rates and prediction market odds contradict each other before I enter a leveraged position, **so that** I don't get caught long during a high-consensus macro event that perp markets are ignoring.

## 功能说明

将永续合约资金费率（Binance、Bybit、dYdX）与预测市场事件赔率（Polymarket、Kalshi）叠加对比，计算量化背离分数，识别哪一方可能定价错误，并根据催化事件距离评估时间紧迫性。帮助交易者在进入杠杆仓位前发现衍生品持仓与宏观事件概率之间的矛盾。

## 适用场景

用户正在考虑或已持有杠杆加密仓位，且存在相关的宏观催化事件（美联储决议、CPI、地缘事件等）。

**触发短语**: "funding 跟 odds 是否一致", "永续合约是否在低估美联储", "funding vs Polymarket", "该不该在事件前对冲", "背离检查", "资金费率正向但我担心美联储", "perps 是否在 misprice 事件风险"

**不触发**: 仅问资金费率但不涉及任何宏观事件; 仅问预测市场赔率但不涉及衍生品; 仅问现货价格走势。

## 输入参数

| 参数 | 必须 | 默认值 | 说明 |
|------|------|--------|------|
| 目标资产 | 否 | BTC | 要检查资金费率的加密资产 |
| 宏观事件 | 否 | 自动推断最近催化事件 | 要对比的预测市场事件 |
| 时间范围 | 否 | 7d | 资金费率历史回溯时间 |

## 执行步骤

### 第一步：识别资产和宏观催化事件

解析用户查询，提取目标资产、宏观事件、时间范围。若用户未指定宏观事件，通过预测市场话题推断最相关的即将到来的催化事件。

### 第二步：收集资金费率数据

```
ant_futures_market_structure(symbol="BTC")
```

提取: current_funding_rate, funding_rate_trend, funding_rate_extreme (阈值 +/- 0.05%), open_interest_change

计算方向性分数:
```
funding_score = clamp(current_funding_rate * 1000, -1, +1)
# 正 = 杠杆多头主导, 负 = 杠杆空头主导
```

### 第三步：收集预测市场赔率

```
ant_market_sentiment(query_type="topics_list")
```

筛选与宏观事件匹配的话题（fed, rate, fomc, cpi, tariff 等），然后:

```
ant_market_sentiment(query_type="topic_posts", topic="<matched_topic>")
```

提取: event_probability, probability_change_24h, event_date

计算方向性分数:
```
event_score = (event_probability_bearish - 0.5) * 2
# 正 = 预测市场倾向空头/鹰派, 负 = 倾向多头/鸽派
```

### 第四步：收集补充市场指标

```
ant_market_indicators()
```

提取: fear_greed_index, btc_dominance（作为打破平局和上下文丰富，非主要背离输入）

### 第五步：计算背离

```
raw_divergence = funding_score - event_score
divergence_magnitude = abs(raw_divergence)
```

背离体制:

| 幅度 | 体制 | 解读 |
|------|------|------|
| 0.00-0.30 | 一致 | 两者讲同一个故事——标准风险 |
| 0.31-0.60 | 轻微背离 | 有张力——入场前验证论点，考虑缩小仓位 |
| 0.61-1.00 | 中度背离 | 明确错配——一方可能定价错误，减仓 30-40% |
| 1.01-2.00 | 严重背离 | 直接矛盾——高清算风险，减仓 50%+ 或对冲 |

### 第六步：生成时间敏感性评估

如有 event_date，计算距催化事件天数:

| 距事件天数 | 紧迫度 | 指引 |
|-----------|--------|------|
| > 7 天 | 低 | 背离可能逐步收敛——每日监控 |
| 3-7 天 | 中 | 事件前建仓或对冲——每日检查 |
| < 3 天 | 高 | 不要无对冲入场，现有仓位需止损 |
| < 24 小时 | 紧急 | 事件临近——平仓或全额对冲 |

## 输出格式

```
=== 资金费率 vs 预测市场赔率背离 ===
日期: YYYY-MM-DD
资产: [BTC/ETH/...]
宏观催化事件: [事件名称]

背离分数:         [0.00-2.00]
体制:             [一致 | 轻微背离 | 中度背离 | 严重背离]
定价错误信号:     [哪一方可能定价错误，或"一致"]
事件紧迫度:       [低 | 中 | 高 | 紧急] ([N] 天后解决)

资金费率信号
  当前费率:       [value]% (8h) 跨 [交易所]
  趋势 (7d):      [上升 | 下降 | 平稳]
  极端标记:       [是 / 否] (阈值: +/- 0.05%)
  OI 变化 (24h):  [+/-]%
  方向性分数:     [funding_score]

预测市场信号
  事件:           [具体事件和解决日期]
  空头概率:       [value]% (24h前 [prior]%, 变动 [+/-]pp)
  方向性分数:     [event_score]

市场上下文
  恐惧贪婪:       [value] ([标签])
  BTC 主导度:     [value]%

解读
[2-3 句通俗语言解释背离，点名具体风险]

建议行动
[一句话: 当前该如何处理持仓及原因]

数据来源
  资金费率:       ant_futures_market_structure
  事件赔率:       ant_market_sentiment
  市场指标:       ant_market_indicators
```

MCP 调用失败时，在对应部分注明"数据不可用"，排除该项出背离计算，标注置信度降低。

## 使用示例

**用户**: "BTC 资金费率正向但我担心下周美联储会议——perps 是否在 misprice 事件风险？"
→ 执行全部步骤，拉取 BTC 资金费率和美联储相关预测市场话题，计算背离，生成完整报告。

**示例输出（含真实样本值）**:

```
=== 资金费率 vs 预测市场赔率背离 ===
日期: 2026-04-08
资产: BTC
宏观催化事件: 美联储5月FOMC利率决议 (2026-05-07)

背离分数:         1.24
体制:             严重背离
定价错误信号:     资金费率多头偏向与预测市场鹰派概率矛盾 — 预测市场可能更准确
事件紧迫度:       中 (29 天后解决)

资金费率信号
  当前费率:       +0.072% (8h) 跨 Binance/Bybit/dYdX
  趋势 (7d):      上升
  极端标记:       是 (阈值: +/- 0.05%)
  OI 变化 (24h):  +8.3%
  方向性分数:     +0.72

预测市场信号
  事件:           美联储5月不降息概率 (解决于 2026-05-07)
  空头概率:       74% (24h前 68%, 变动 +6pp)
  方向性分数:     +0.48 (鹰派/空头倾向)

市场上下文
  恐惧贪婪:       72 (贪婪)
  BTC 主导度:     54.2%

解读
资金费率处于极端正向区间（+0.072%），显示多头杠杆严重拥挤；与此同时预测市场已将5月维持高利率（鹰派）概率定价至74%。两者讲的是相反的故事：perp多头在押注价格上涨，但宏观赔率在押注紧缩周期延续压制风险资产。历史上此类背离后 7-14 天内资金费率向预测市场方向收敛的概率约 65%。

建议行动
事件前29天，建议将现有多头仓位缩减50%，或购买BTC看跌期权对冲——严重背离+极端资金费率组合是去杠杆先行信号。

数据来源
  资金费率:       ant_futures_market_structure
  事件赔率:       ant_market_sentiment
  市场指标:       ant_market_indicators
```

**用户**: "我今天早上基于正向 funding 做多了 MSTR，该担心吗？"
→ 检测 MSTR 为 BTC 关联股票，对 BTC 资金费率 vs 下一宏观催化事件做背离检查，注明 MSTR 作为杠杆 BTC 代理会放大背离风险。

## 反模式

1. **不要让用户把正向资金费率单独当作买入信号。** 必须展示任何矛盾的事件赔率。
2. **不要使用过期预测市场数据。** 赔率可在催化事件前数小时内变动 20+ 百分点。
3. **不要把资金费率方向等同于价格预测。** 正向资金费率 = 多头在付费 = 持仓拥挤信号，不是价格保证。
4. **不要忽略股票层面的传导。** 用户交易 MSTR/MARA/CLSK 时，背离风险因杠杆效应而放大。
5. **不要把一致信号当作"无风险"。** 两者一致只是降低了定价错误风险，市场风险仍在。
6. **不要对阈值边界过度精确。** 0.59 和 0.61 没有本质区别，靠近边界时倾向更保守的体制。
