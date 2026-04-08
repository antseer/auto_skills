---
name: macro-defi-impact-assessor
description: 重大地缘/宏观事件发生后，自动构建从事件到链上数据的多层传导链，输出结构化的事件影响评估报告。
---

# 宏观事件→DeFi 影响传导链评估器

## 功能说明

当重大地缘政治或宏观经济事件发生后，自动从市场情绪、宏观基准、链上资金流向、市场技术指标、DeFi 生态五个层次构建传导链，量化每层的变化方向和幅度，在 15 分钟内输出结构化影响评估报告。替代用户在 5+ 个工具间手动切换的 1-3 小时流程。

## 适用场景

- 重大地缘政治新闻发布后（军事冲突、外交谈判破裂、贸易制裁）
- 央行利率决议或通胀数据发布后
- 监管法案通过或重大执法行动后
- 需要快速评估宏观事件对 DeFi 策略或加密持仓的影响时

### 触发短语
- "伊朗霍尔木兹海峡管控对 DeFi 有什么影响？"
- "今天的 CPI 数据出来了，我的 DeFi 持仓安全吗？"
- "evaluate macro impact of Iran tensions on aave and compound"
- `/macro-defi-impact '霍尔木兹海峡管控' --protocols aave,compound,lido`

## 输入参数

| 参数 | 必填 | 说明 | 默认值 |
|------|------|------|--------|
| 事件关键词 | 是 | 地缘/宏观事件描述 | — |
| protocols | 否 | 关注的 DeFi 协议列表 | 无（仅评估 BTC/ETH 层面） |
| coins | 否 | 额外关注的币种 | BTC, ETH |

## 执行步骤

### Step 1: 识别事件并获取市场情绪上下文
调用 `ant_market_sentiment(query_type='topics_list')` 获取当前热门话题列表。
从中筛选与用户输入的地缘/宏观事件关键词相关的话题。
对匹配的话题调用 `ant_market_sentiment(query_type='topic_detail', topic_id=<id>)` 获取话题详情和情绪评分。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 获取相关帖子作为市场叙事上下文。

### Step 2: 获取宏观经济基准数据
调用 `ant_macro_economics(query_type='cpi')` 获取最新 CPI 数据。
调用 `ant_macro_economics(query_type='federal_funds_rate')` 获取当前联邦基金利率。
调用 `ant_macro_economics(query_type='treasury_yield')` 获取国债收益率曲线。
调用 `ant_macro_economics(query_type='inflation')` 获取通胀数据。
这些数据构成传导链的"宏观基准层"。

### Step 3: 获取链上资金流向数据
调用 `ant_fund_flow(query_type='exchange_netflow', coin='BTC')` 获取 BTC 交易所净流入/流出。
调用 `ant_fund_flow(query_type='exchange_netflow', coin='ETH')` 获取 ETH 交易所净流入/流出。
调用 `ant_fund_flow(query_type='exchange_reserve', coin='BTC')` 获取 BTC 交易所储备量变化。
调用 `ant_fund_flow(query_type='exchange_reserve', coin='ETH')` 获取 ETH 交易所储备量变化。
调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDT')` 获取 USDT 链上流向（必须执行：stablecoin 流向是 DeFi TVL 变化的最佳领先指标）。
调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDC')` 获取 USDC 链上流向（必须执行）。
这些数据构成传导链的"链上资金流动层"。

### Step 4: 获取市场技术指标
对用户关注的币种（默认 BTC, ETH），调用:
- `ant_market_indicators(query_type='rsi', coin=<coin>)` 获取 RSI
- `ant_market_indicators(query_type='macd', coin=<coin>)` 获取 MACD
- `ant_market_indicators(query_type='boll', coin=<coin>)` 获取布林带
- `ant_market_indicators(query_type='atr', coin=<coin>)` 获取 ATR（波动率指标）
- `ant_market_indicators(query_type='funding_rate', coin=<coin>)` 获取资金费率
这些数据构成传导链的"市场微观结构层"。

### Step 5: 获取 DeFi 协议 TVL 和收益率数据（如用户指定协议）
调用 `ant_market_sentiment(query_type='topics_list')` 搜索用户指定的 DeFi 协议相关话题。
对匹配话题调用 `ant_market_sentiment(query_type='topic_time_series', topic_id=<id>)` 获取话题热度时间序列。
结合 Step 3 的 stablecoin 资金流向，推断 DeFi TVL 变化方向。

### Step 6: 构建传导链并生成报告
将上述数据按以下传导逻辑组织：
```
地缘/宏观事件
  → 市场情绪变化（Step 1: 话题热度、帖子情绪）
  → 宏观指标反应（Step 2: CPI/利率/国债收益率）
  → 链上资金流动（Step 3: 交易所净流入/流出、储备量变化）
  → 市场价格/波动率（Step 4: RSI/MACD/ATR/资金费率）
  → DeFi 生态影响（Step 5: 相关话题热度、stablecoin 流向推断）
```
对每层标注：变化方向（↑/↓/→）、变化幅度、置信度（高/中/低）。
置信度判断准则：≥3 层数据方向一致 → 高；1-2 层支持 → 中；仅单层或方向相互矛盾 → 低。

## 输出格式

```markdown
# 宏观事件影响评估报告

## 事件: [事件名称]
## 评估时间: [YYYY-MM-DD HH:MM UTC]
## 整体影响等级: [高/中/低]

---

## 传导链分析

### Layer 1: 市场情绪
- 相关话题热度: [热度值] ([变化方向] [变化幅度])
- 市场情绪倾向: [看涨/看跌/中性]
- 关键帖子摘要: [1-2句]

### Layer 2: 宏观基准
- CPI (最新): [值] | 联邦基金利率: [值]
- 国债收益率 (10Y): [值] ([变化])
- 通胀预期: [值]
- 宏观环境判断: [紧缩/宽松/中性]

### Layer 3: 链上资金流动
| 币种 | 交易所净流入(24h) | 交易所储备变化(24h) | 信号 |
|------|-------------------|---------------------|------|
| BTC  | [值]              | [值]                | [流入/流出/平稳] |
| ETH  | [值]              | [值]                | [流入/流出/平稳] |
| USDT | [值]              | [值]                | [流入/流出/平稳] |
| USDC | [值]              | [值]                | [流入/流出/平稳] |

### Layer 4: 市场微观结构
| 币种 | RSI | MACD信号 | ATR(波动率) | 资金费率 | 综合判断 |
|------|-----|---------|------------|---------|---------|
| BTC  | [值]| [值]    | [值]       | [值]    | [判断]  |
| ETH  | [值]| [值]    | [值]       | [值]    | [判断]  |

### Layer 5: DeFi 生态影响推断
- Stablecoin 净流向: [方向] ([量级])
- DeFi TVL 变化推断: [方向] ([置信度])
- 借贷利率压力: [上行/下行/中性]

---

## 综合评估
- **对 DeFi 收益策略的影响**: [1-2句结论]
- **对加密持仓的影响**: [1-2句结论]
- **建议关注**: [2-3个关键监控点]

## 置信度声明
本评估基于公开市场数据的量化分析，传导链每层的影响估计受数据延迟和模型简化限制。
地缘事件的市场影响存在高度不确定性，本报告不构成投资建议。
```

## 使用示例

**输入:**
```
/macro-defi-impact '霍尔木兹海峡管控升级' --protocols aave,compound
```

**输出:** 生成包含 5 层传导链分析的结构化报告，从市场情绪（话题热度 87/100）→ 宏观基准（CPI 3.2%、利率 4.75%）→ 链上资金（BTC +$180M 流入交易所）→ 市场技术（RSI 42、空头信号）→ DeFi 推断（stablecoin 链下流出 -$370M）。

## 反模式

1. **不要做方向性交易建议**: 此 skill 评估影响，不预测价格。传导链分析不等于交易信号。
2. **不要忽视置信度标注**: 传导链越长，每层误差累积越大。必须对每层标注置信度。
3. **不要将相关性当因果性**: 交易所资金流入与价格下跌的时间重叠不等于因果关系。
4. **不要遗漏 stablecoin 数据**: Stablecoin 链上流向是 DeFi TVL 变化的最佳领先指标。
5. **数据延迟风险**: MCP 数据可能有 15-60 分钟延迟，报告应标注数据时间戳。
6. **不覆盖**: 期权定价、杠杆倍数建议、具体协议智能合约风险评估。
