---
name: smart-money-spread
description: 计算机构信念 vs 散户恐惧的实时背离价差，返回量化分数、体制标签和分层解读。
---

# Smart Money 价差

## 功能说明

计算机构买家（鲸鱼钱包、ETF 资金流、企业国库购入）与散户情绪（社交情绪、恐惧贪婪指数、资金费率）之间的实时背离价差。产出 -100 到 +100 的量化分数、体制标签、以及针对不同认知水平的分层解读。当两者分歧时（机构积累 + 散户恐慌，或机构分销 + 散户贪婪）价值最大。

## 适用场景

用户想知道机构买家和散户情绪是否朝同一方向移动。适用于: 重大机构 BTC 购入被披露; 散户情绪急剧下降; 即将调整加密或 BTC 关联股票仓位; 恐慌标题出现时做理性检查。

**触发短语**: "鲸鱼在买还是在卖", "smart money 在这次下跌中积累吗", "机构 vs 散户价差是多少", "该不该担心这次暴跌", "Saylor 刚买了——散户怎么看", "入场前查一下 smart money spread", "恐惧有根据还是机构在买", "smart money vs scared money"

## 输入参数

| 参数 | 必须 | 默认值 | 说明 |
|------|------|--------|------|
| 资产 | 否 | BTC | 要分析的资产 |
| 时间窗口 | 否 | 7d | 回溯期 |

## 执行步骤

### 第一步：收集机构信念信号

```
ant_smart_money(query_type="whale_movements", asset="BTC", timeframe="7d")
```
记录: whale_net_flow, whale_buy_count, whale_total_count
→ whale_conviction_ratio = whale_buy_count / whale_total_count

```
ant_fund_flow(query_type="etf_flows", asset="BTC", timeframe="7d")
```
记录: etf_net_flow_usd, etf_flow_trend

```
ant_macro_economics(query_type="institutional_activity", asset="BTC")
```
记录: treasury_purchases (实体、BTC量、USD量、成本基础), treasury_vs_market_price

计算 **institutional_conviction_score** (0-100):
```
whale_component = whale_conviction_ratio * 40          # 最高 40 分
etf_component = normalize(etf_net_flow_usd, -500M..+500M) * 30   # 最高 30 分
treasury_component = (30 if 7天内有折价购入 else 15 if 溢价购入 else 0)  # 最高 30 分
institutional_conviction_score = round(whale_component + etf_component + treasury_component)
```

### 第二步：收集散户恐惧信号

```
ant_market_sentiment(query_type="topics_list")
```
筛选加密相关话题，前 5 个:
```
ant_market_sentiment(query_type="topic_posts", topic="<topic>")
```
计算: social_sentiment_mean (-1..+1), social_bearish_pct

```
ant_market_indicators(query_type="fear_greed_index")
```
记录: fear_greed_score (0-100), fear_greed_label

```
ant_futures_market_structure(query_type="funding_rates", asset="BTC", exchanges=["binance","bybit","dydx"])
```
记录: avg_funding_rate, funding_rate_percentile (vs 90天)

计算 **retail_fear_score** (0-100, 100=最大恐惧):
```
sentiment_component = (1 - (social_sentiment_mean + 1) / 2) * 30  # 最高 30
fear_greed_component = (100 - fear_greed_score) * 0.4             # 最高 40
funding_component = normalize(-avg_funding_rate, -0.05..+0.05) * 30  # 最高 30
retail_fear_score = round(sentiment_component + fear_greed_component + funding_component)
```

### 第三步：计算背离价差

```
raw_spread = institutional_conviction_score - (100 - retail_fear_score)
normalized_spread = round(raw_spread)  # 范围约 -100 到 +100
```

体制标签:

| 价差 | 体制 | 解读 |
|------|------|------|
| +60 到 +100 | 强机构积累 | 机构大举买入 + 散户恐慌。历史上利于中期多头。 |
| +30 到 +59 | 机构积累 | 机构净买入，散户恐惧。背离明显但不极端。 |
| -29 到 +29 | 中性/一致 | 双方大致方向一致。无背离信号。 |
| -59 到 -30 | 机构分销 | 机构净卖出 + 散户自满/贪婪。需谨慎。 |
| -100 到 -60 | 强机构分销 | 机构大举分销 + 散户追涨。历史上先于回调。 |

### 第四步：生成分层解读

**用户类型判断规则**（从对话上下文推断）：

| 用户类型 | 判断条件 |
|----------|---------|
| 🟢 通俗用户 | 持仓 < $5,000，或自述为新手，或提问中未使用专业术语 |
| 🔵 分析用户 | 持仓 ≥ $5,000，或主动使用"资金费率"/"IV"/"8-K"等术语，或要求详细数据 |

**通俗解读写作规则（严格执行）**：

通俗解读必须满足以下所有条件：
1. **严禁使用**以下术语（无论何种用户类型均在通俗解读中禁止）：资金费率、funding rate、ETF 资金流、whale、鲸鱼净流向、机构信念分数、散户恐惧分数、IV、期权、价差、normalization、分位数、percentile、8-K、基础差
2. **必须使用**具体数字代替术语：不写"鲸鱼净买入"，写"X 个大 BTC 持有者中有 Y 个本周加仓"
3. **必须给出一句行动方向提示**（不是买卖建议，而是情绪定性）：例如"标题在说恐慌，但大买家在加仓" 或 "大持有者本周在卖出，散户仍在追涨"

通俗用户额外输出（在完整报告前方插入）：
```
通俗解读: "[X] 个大 BTC 持有者中有 [Y] 个本周加仓。
市场情绪评级: [极度恐惧 | 恐惧 | 中性 | 贪婪 | 极度贪婪]（[fear_greed_score] 分）。
这意味着: [一句话解释当前背离对普通持有者的实际含义，不含任何专业术语]。"
```

所有用户都输出完整分析版（见输出格式）。

### 第五步：输出完整报告

## 输出格式

```
=== SMART MONEY 价差 ===
日期: YYYY-MM-DD
资产: [BTC | ETH | 指定资产]

背离价差:         [数值, -100 到 +100]
体制:             [强机构积累 | 机构积累 | 中性/一致 | 机构分销 | 强机构分销]

机构信念  [score]/100
  鲸鱼积累:       [ratio] ([buy_count]/[total_count] 钱包净买入)
  ETF 资金流 (7d): [净流入/出 USD] ([趋势])
  国库购入:       [实体: 数量 @ 价格, vs 平均成本] 或 "窗口内无"
  成本基础信号:   [买弱势 | 买强势 | 无数据]

散户恐惧  [score]/100
  社交情绪:       [mean] ([bearish_pct]% 看空)
  恐惧贪婪指数:   [score] ([标签])
  平均资金费率:   [rate] ([percentile] vs 90天)

解读
[2-4 句描述具体背离。点名机构行为者、散户信号、
 以及价差历史上意味着什么。]

通俗解读
[1-2 句面向非技术用户。]

数据新鲜度
  鲸鱼数据:    [时间戳] ([距今])
  ETF 资金流:  [时间戳] ([距今])
  资金费率:    [时间戳] ([距今])
  恐惧贪婪:    [时间戳] ([距今])
  社交:        [时间戳] ([距今])
```

## 使用示例

**用户**: "Saylor 又宣布 BTC 购入了。该担心暴跌还是机构在买低？"

→ 执行全部步骤。如果鲸鱼 8/11 净买入、ETF +$412M 加速流入、Strategy 以低于成本基础 10.5% 购入，同时恐惧贪婪 22、资金费率 -0.025%，价差 +68 = 强机构积累。通俗解读: "11 个大 BTC 持有者中有 8 个本周加仓。标题说恐慌，但最大买家把这当作买入机会。小额组合不必恐慌卖出。"

## 反模式

- **本技能不提供买卖建议。** 只计算背离价差和描述机构 vs 散户行为。
- **本技能不预测价格方向。** 高机构信念分数不保证反弹。
- **本技能不追踪分析师观点。** 只追踪机构实际资金行为（钱包流、ETF 购入、8-K 文件）。
- **本技能不要求用户懂资金费率或 8-K 文件。** 通俗解读部分翻译一切。
- **本技能不替代仓位大小工具。** 提供背离信号辅助决策，不计算仓位/止损。
- **本技能不覆盖无机构存在的资产。** 低市值山寨币/meme 币缺乏机构数据时明确说明无法计算。
- **本技能不把过期数据当新鲜数据。** 数据新鲜度部分展示每个输入的年龄。鲸鱼数据 > 4 小时或资金费率 > 15 分钟时标注置信度降低。
