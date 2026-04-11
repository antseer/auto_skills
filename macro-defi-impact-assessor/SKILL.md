---
name: macro-defi-impact-assessor
description: 重大地缘/宏观事件发生后，自动构建从事件到链上数据的多层因果传导链，跨源交叉验证，输出带失效条件的结构化事件影响评估报告。
---

# 宏观事件→DeFi 影响传导链评估器

## 功能说明

当重大地缘政治或宏观经济事件发生后，自动从市场情绪、宏观基准、链上资金流向、市场技术指标、DeFi 生态五个层次构建**因果传导链**，对每层进行多源交叉验证，量化变化方向和幅度，**为每个核心结论标注失效条件**，在 15 分钟内输出结构化影响评估报告。替代用户在 5+ 个工具间手动切换的 1-3 小时流程。

## 用户画像

### Persona 1: Carol — DeFi 收益策略研究员
- **角色**: 全职 DeFi 收益策略分析师，为客户管理多协议收益组合
- **日常工作流**: 每天监控 Aave/Compound/Lido 的利率变化和 TVL 趋势，每周出策略报告，重大事件时需 15 分钟内评估影响
- **工具栈**: DefiLlama、Dune Analytics、TradingView、CoinGlass、手动 Google Sheets 汇总
- **痛点**: 地缘事件发生后需要在 5+ 个工具间手动切换做传导分析，耗时 1-2 小时，策略报告总是滞后于市场反应。上周霍尔木兹海峡事件后花了 2 小时手动追踪 WTI->TVL->借贷利率->stablecoin 外流的传导链。

### Persona 2: Marcus — 加密衍生品交易员
- **角色**: 全职加密衍生品交易员，主做 BTC/ETH 永续合约和 altcoin swing
- **日常工作流**: 每日盘前检查资金费率、OI 变化、宏观日历，事件驱动时快速评估方向
- **工具栈**: Binance/Bybit/OKX、CoinGlass、TradingView、Telegram 机器人
- **痛点**: 地缘事件后需要快速判断传导路径（事件→风险偏好→资金流→DeFi TVL→借贷利率），但跨源数据分散在不同平台，等手动汇总完毕最佳入场窗口已过。早期 3 月宏观-加密叙事分歧期间因无法量化传导链而损失了持仓的 40% 年度收益。

### Persona 3: Derek — 传统-加密跨界配置者
- **角色**: 兼职投资者，传统股票+加密混合持仓，月度再平衡
- **日常工作流**: 每周日晚检查宏观指标（CPI、利率、收益率曲线）和加密持仓，每月调整配置比例
- **工具栈**: Yahoo Finance、FRED、CashApp、Reddit/Substack
- **痛点**: 宏观事件对加密持仓的影响传导链完全靠直觉判断。Q3 2025 美元走强期间因无法量化宏观→加密传导而错失减仓时机，回吐了 40% 年度加密收益。

## 用户故事

1. **As Carol**, I want to input a geopolitical event keyword and get a structured 5-layer transmission chain from event to DeFi impact within 15 minutes, **so that** my strategy reports are timely rather than always lagging behind market moves by 1-2 hours.
2. **As Marcus**, I want each layer of the transmission chain to show cross-verified confidence with explicit invalidation conditions, **so that** I can size my positions based on how robust the signal actually is rather than gut-calling it.
3. **As Derek**, I want a plain-language summary with a clear "this judgment fails if..." section, **so that** I know exactly when to revisit my monthly rebalancing decision rather than holding through a regime change out of ignorance.

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
- "Fed just cut rates, what happens to my Aave yields?"
- "CLARITY Act 通过后 DeFi 会怎样？"

## 输入参数

| 参数 | 必填 | 说明 | 默认值 |
|------|------|------|--------|
| 事件关键词 | 是 | 地缘/宏观事件描述 | -- |
| protocols | 否 | 关注的 DeFi 协议列表 | 无（仅评估 BTC/ETH 层面） |
| coins | 否 | 额外关注的币种 | BTC, ETH |

## 执行步骤

### Step 1: 识别事件并获取市场情绪上下文
1. 调用 `ant_market_sentiment(query_type='topics_list')` 获取当前热门话题列表。
2. 从中筛选与用户输入的地缘/宏观事件关键词相关的话题。
3. 对匹配的话题调用 `ant_market_sentiment(query_type='topic_detail', topic_id=<id>)` 获取话题详情和情绪评分。
4. 调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 获取相关帖子作为市场叙事上下文。

**因果推理要求**: 不仅记录情绪方向，还要解释 **为什么** 市场情绪如此反应。例如：
- 霍尔木兹海峡管控 → 市场恐慌（为什么？→ 能源供应中断预期 → 通胀上行压力 → 美联储推迟降息预期）
- 必须写出至少 2 层 "为什么" 推理，不能直接从事件跳到情绪结论。

### Step 2: 获取宏观经济基准数据
1. 调用 `ant_macro_economics(query_type='cpi')` 获取最新 CPI 数据。
2. 调用 `ant_macro_economics(query_type='federal_funds_rate')` 获取当前联邦基金利率。
3. 调用 `ant_macro_economics(query_type='treasury_yield')` 获取国债收益率曲线。
4. 调用 `ant_macro_economics(query_type='inflation')` 获取通胀数据。

**因果推理要求**: 对宏观数据与事件之间的因果关系进行多层解释：
- 第 1 层 "为什么"：事件如何影响通胀预期/利率路径？（例：能源价格上涨 → 核心 CPI 滞后上行 → 美联储维持高利率更长时间）
- 第 2 层 "为什么"：利率路径变化如何影响风险资产配置？（例：高利率维持更久 → 国债收益率上升 → 风险溢价压缩 → 资金从高风险资产撤出）
- 禁止直接说 "CPI 偏高，对加密不利" 而不解释传导机制。

**跨源验证**: 将 `ant_macro_economics` 的 CPI/利率数据与 Step 1 中市场情绪帖子中引用的宏观数据进行对比。如果帖子中引用的数据与 MCP 返回数据存在矛盾，标注为 "叙事偏差"。

### Step 3: 获取链上资金流向数据
1. 调用 `ant_fund_flow(query_type='exchange_netflow', coin='BTC')` 获取 BTC 交易所净流入/流出。
2. 调用 `ant_fund_flow(query_type='exchange_netflow', coin='ETH')` 获取 ETH 交易所净流入/流出。
3. 调用 `ant_fund_flow(query_type='exchange_reserve', coin='BTC')` 获取 BTC 交易所储备量变化。
4. 调用 `ant_fund_flow(query_type='exchange_reserve', coin='ETH')` 获取 ETH 交易所储备量变化。
5. 调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDT')` 获取 USDT 链上流向（**必须执行**：stablecoin 流向是 DeFi TVL 变化的最佳领先指标）。
6. 调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDC')` 获取 USDC 链上流向（**必须执行**）。
7. 调用 `ant_stablecoin(query_type='overview')` 获取 stablecoin 整体市值和流通量变化。

**因果推理要求**: 解释资金流动的 "为什么"：
- 第 1 层：资金流向方向是什么？（例：BTC 净流入交易所 +$180M）
- 第 2 层：为什么资金在这个方向流动？（例：地缘恐慌 → 持有者将 BTC 转入交易所准备抛售 → 同时 stablecoin 从交易所流出 → 表明资金在离场而非换仓）
- 第 3 层：这对 DeFi TVL 意味着什么？（例：stablecoin 外流 → DeFi 协议可用流动性下降 → 借贷利用率被动上升 → 借贷利率短期走高）

**跨源验证 (关键)**: BTC/ETH 的 netflow 和 reserve 数据必须相互验证：
- netflow 显示流入但 reserve 未增加 → 可能是 OTC 交易或数据延迟，置信度降低一级
- USDT 和 USDC 流向方向不一致 → 标注 "stablecoin 信号分裂"，不可直接得出高置信度结论
- 必须同时参考 `ant_stablecoin` 整体数据验证单币流向是否反映整体趋势

### Step 4: 获取市场技术指标
对用户关注的币种（默认 BTC, ETH），调用:
1. `ant_market_indicators(query_type='rsi', coin=<coin>)` 获取 RSI
2. `ant_market_indicators(query_type='macd', coin=<coin>)` 获取 MACD
3. `ant_market_indicators(query_type='boll', coin=<coin>)` 获取布林带
4. `ant_market_indicators(query_type='atr', coin=<coin>)` 获取 ATR（波动率指标）
5. `ant_market_indicators(query_type='funding_rate', coin=<coin>)` 获取资金费率

**因果推理要求**:
- 第 1 层：指标当前状态（例：RSI 42 处于中性偏空区间，资金费率转负）
- 第 2 层：为什么指标处于这个状态？（例：资金费率转负 → 说明空头头寸增加/多头在平仓 → 这与 Step 3 中 BTC 交易所净流入一致，验证了 "恐慌抛售" 而非 "有序减仓" 的叙事）
- 第 3 层：如果当前趋势持续，后果是什么？（例：如果空头继续堆积且资金费率跌破 -0.03% → 可能触发空头挤压反弹 → 这与看跌传导链的方向相反）

**跨源验证**: 资金费率信号必须与 Step 3 的链上资金流向交叉验证：
- 资金费率看空 + 链上净流入交易所 → 方向一致 → 置信度维持
- 资金费率看空 + 链上净流出交易所 → 方向矛盾 → 置信度降低一级，标注 "技术面与链上面信号冲突"

### Step 5: 获取 DeFi 协议数据（如用户指定协议）
1. 调用 `ant_protocol_tvl_yields_revenue(query_type='tvl', protocol=<protocol>)` 获取协议 TVL 数据。
2. 调用 `ant_protocol_tvl_yields_revenue(query_type='yields', protocol=<protocol>)` 获取协议收益率数据。
3. 调用 `ant_protocol_tvl_yields_revenue(query_type='revenue', protocol=<protocol>)` 获取协议收入数据。
4. 调用 `ant_market_sentiment(query_type='topics_list')` 搜索用户指定的 DeFi 协议相关话题。
5. 对匹配话题调用 `ant_market_sentiment(query_type='topic_time_series', topic_id=<id>)` 获取话题热度时间序列。

**因果推理要求**:
- 第 1 层：TVL 和收益率的当前变化方向
- 第 2 层：为什么 TVL 在变化？（例：stablecoin 从 DeFi 流出 → 为什么？→ 用户将资金转入交易所卖出 → 或者 CeFi 替代品收益率更有竞争力 → 需要用 Step 3 的 stablecoin 流向数据区分这两种解释）
- 第 3 层：TVL 变化对借贷利率和收益策略的影响（例：TVL 下降 → 借贷利用率被动上升 → 借贷利率上升 → 但如果同时借款需求也下降则利用率可能不变）

**跨源验证**: DeFi TVL 变化必须与 Step 3 的 stablecoin 流向交叉验证：
- TVL 下降 + stablecoin 净流出 → 方向一致 → 高置信度
- TVL 下降 + stablecoin 净流入 → 矛盾 → 可能是 TVL 计价币种贬值导致的 "假性 TVL 下降"，不是真正的资金外流

### Step 6: 构建因果传导链并生成报告

**传导链逻辑（每层必须包含 "为什么" 推理）**:
```
地缘/宏观事件
  → [为什么?] → 市场情绪变化（Step 1）
  → [为什么?] → 宏观指标反应（Step 2）
  → [为什么?] → 链上资金流动（Step 3）
  → [为什么?] → 市场价格/波动率（Step 4）
  → [为什么?] → DeFi 生态影响（Step 5）
```

**置信度判断准则（必须基于跨源验证）**:
- **高置信度**: >=3 层数据方向一致 **且** 每层内 >=2 个独立 MCP 源交叉验证通过（例：链上层中 netflow + reserve + stablecoin 三组数据方向一致）
- **中置信度**: 2 层数据方向一致，或 3 层一致但层内存在单源未验证
- **低置信度**: 仅 1 层支持，或层间方向相互矛盾，或层内数据源信号分裂

**失效条件生成规则**: 对每个核心结论，必须生成至少 1 个失效条件：
- 格式: "此判断在以下条件下失效: [具体条件]"
- 失效条件必须是可观测、可量化的（不接受 "如果市场环境变化" 这种模糊表述）
- 示例: "此看空判断在以下条件下失效: BTC 在 24 小时内站稳 $70,000 上方 + 资金费率转正 + stablecoin 净流入 DeFi 协议"

## 输出格式

```markdown
# 宏观事件影响评估报告

## 事件: [事件名称]
## 评估时间: [YYYY-MM-DD HH:MM UTC]
## 整体影响等级: [高/中/低]
## 整体置信度: [高/中/低] (基于 [N] 层交叉验证)

---

## 因果传导链分析

### Layer 1: 市场情绪
- 相关话题热度: [热度值] ([变化方向] [变化幅度])
- 市场情绪倾向: [看涨/看跌/中性]
- 关键帖子摘要: [1-2句]
- **因果推理**: [事件] → [为什么?] → [情绪反应] → [为什么?] → [预期行为]
- **跨源一致性**: [情绪帖子引用的数据] vs [MCP 宏观数据] = [一致/存在叙事偏差]
- **失效条件**: 此情绪判断在以下条件下失效: [具体条件]

### Layer 2: 宏观基准
- CPI (最新): [值] | 联邦基金利率: [值]
- 国债收益率 (10Y): [值] ([变化])
- 通胀预期: [值]
- 宏观环境判断: [紧缩/宽松/中性]
- **因果推理**: [事件] → [第1层为什么: 如何影响通胀/利率路径] → [第2层为什么: 利率变化如何影响风险偏好]
- **失效条件**: 此宏观判断在以下条件下失效: [具体条件]

### Layer 3: 链上资金流动
| 币种 | 交易所净流入(24h) | 交易所储备变化(24h) | 信号 | 内部交叉验证 |
|------|-------------------|---------------------|------|-------------|
| BTC  | [值]              | [值]                | [流入/流出/平稳] | [netflow vs reserve 一致/矛盾] |
| ETH  | [值]              | [值]                | [流入/流出/平稳] | [netflow vs reserve 一致/矛盾] |
| USDT | [值]              | N/A                 | [流入/流出/平稳] | [与 USDC 方向 一致/分裂] |
| USDC | [值]              | N/A                 | [流入/流出/平稳] | [与 USDT 方向 一致/分裂] |

- **Stablecoin 整体**: [ant_stablecoin 市值变化] → [与单币 netflow 一致/矛盾]
- **因果推理**: [资金流向方向] → [为什么: 驱动力分析] → [为什么: 对 DeFi 流动性的影响机制]
- **失效条件**: 此链上判断在以下条件下失效: [具体条件]

### Layer 4: 市场微观结构
| 币种 | RSI | MACD信号 | ATR(波动率) | 资金费率 | 综合判断 |
|------|-----|---------|------------|---------|---------|
| BTC  | [值]| [值]    | [值]       | [值]    | [判断]  |
| ETH  | [值]| [值]    | [值]       | [值]    | [判断]  |

- **跨源验证**: 资金费率信号 vs 链上 netflow 信号 = [一致/冲突]
- **因果推理**: [指标状态] → [为什么处于此状态] → [如果趋势持续的后果]
- **失效条件**: 此技术判断在以下条件下失效: [具体条件，例如 "RSI 跌破 30 且资金费率跌破 -0.05% 则空头挤压概率上升"]

### Layer 5: DeFi 生态影响推断
- 协议 TVL: [值] ([变化方向] [变化幅度])
- 协议收益率: [值] ([变化方向])
- Stablecoin 净流向: [方向] ([量级])
- DeFi TVL 变化推断: [方向] ([置信度])
- 借贷利率压力: [上行/下行/中性]
- **跨源验证**: TVL 变化方向 vs stablecoin 流向方向 = [一致/矛盾 → 如矛盾说明可能是计价币种贬值导致假性下降]
- **因果推理**: [TVL 变化] → [为什么: stablecoin 外流 or 计价贬值?] → [对借贷利率的影响机制] → [对收益策略的影响]
- **失效条件**: 此 DeFi 判断在以下条件下失效: [具体条件]

---

## 综合评估
- **对 DeFi 收益策略的影响**: [1-2句结论]
  - 失效条件: [具体可观测条件]
- **对加密持仓的影响**: [1-2句结论]
  - 失效条件: [具体可观测条件]
- **建议关注**: [2-3个关键监控点]

## 跨源验证摘要
| 验证对 | 数据源 A | 数据源 B | 结果 |
|--------|---------|---------|------|
| 链上内部 | BTC netflow | BTC reserve | [一致/矛盾] |
| Stablecoin | USDT flow | USDC flow | [一致/分裂] |
| Stablecoin 整体 | 单币 netflow | ant_stablecoin 整体 | [一致/矛盾] |
| 技术面 vs 链上 | 资金费率方向 | 链上 netflow 方向 | [一致/冲突] |
| DeFi vs 链上 | TVL 变化方向 | stablecoin 流向 | [一致/矛盾] |
| 叙事 vs 数据 | 市场情绪帖子 | MCP 宏观数据 | [一致/叙事偏差] |

## 置信度声明
本评估基于 [N] 个独立 MCP 数据源的交叉验证分析。传导链共 5 层，每层的因果推理明确标注了推理依据和失效条件。
- 跨源验证通过率: [X/Y] 对
- 信号冲突: [列出存在冲突的验证对]
- 数据时间戳: [各 MCP 调用的时间戳]
地缘事件的市场影响存在高度不确定性，本报告不构成投资建议。每个结论的失效条件提供了判断何时应重新评估的具体指引。
```

## 使用示例

**输入:**
```
/macro-defi-impact '霍尔木兹海峡管控升级' --protocols aave,compound
```

**输出 (简化摘要):**

```
# 宏观事件影响评估报告

## 事件: 霍尔木兹海峡管控升级
## 评估时间: 2026-04-09 14:30 UTC
## 整体影响等级: 中
## 整体置信度: 中 (基于 3/5 层交叉验证通过)

## 因果传导链分析

### Layer 1: 市场情绪
- 话题热度: 87/100 (↑ +32)
- 情绪: 看跌
- 因果推理: 海峡管控 → 为什么恐慌？→ 全球 20% 石油运输受威胁 → 能源价格飙升预期 → 为什么影响加密？→ 通胀上行 → 美联储推迟降息 → risk-off
- 失效条件: 此恐慌情绪判断在以下条件下失效: 伊朗在 48 小时内撤回管控措施，或 OPEC 宣布增产弥补供应缺口

### Layer 2: 宏观基准
- CPI: 3.2% | 利率: 4.75% | 10Y: 4.52% (↑ +0.08)
- 因果推理: 能源价格 → 第1层: 运输成本上升推高 PPI → 第2层: 3-6 个月后传导至核心 CPI → 美联储维持 4.75% 更久 → 国债收益率吸引力上升 → 风险资产配置下降
- 失效条件: 此判断在以下条件下失效: 原油价格 48h 内回落至事件前水平，或美联储明确表态 "energy shock 不改变降息路径"

### Layer 3: 链上资金流动
| 币种 | 交易所净流入(24h) | 储备变化 | 信号 | 内部验证 |
|------|-------------------|---------|------|---------|
| BTC  | +$180M           | +0.8%   | 流入 | 一致    |
| ETH  | +$95M            | +0.5%   | 流入 | 一致    |
| USDT | -$370M (链下)    | N/A     | 流出 | 与USDC一致 |
| USDC | -$210M (链下)    | N/A     | 流出 | 与USDT一致 |

- 因果推理: BTC/ETH 流入交易所 → 为什么？→ 持有者准备抛售避险 → stablecoin 链下流出 → 为什么？→ 资金在离场而非换仓 → DeFi 可用流动性将减少
- 失效条件: 此判断在以下条件下失效: BTC 交易所储备在 24h 内回落至事件前水平 + stablecoin 流出逆转为净流入

### Layer 4: 市场微观结构
| 币种 | RSI | MACD | ATR | 资金费率 | 综合 |
|------|-----|------|-----|---------|------|
| BTC  | 42  | 空头 | 偏高 | -0.01% | 偏空 |

- 跨源验证: 资金费率看空 + 链上净流入交易所 = 方向一致
- 因果推理: 资金费率转负 → 空头增加/多头平仓 → 如果继续恶化至 -0.03% → 可能触发空头挤压反弹 → 与看跌链条方向相反
- 失效条件: BTC RSI 跌破 30 + 资金费率跌破 -0.03% 则空头挤压概率上升至高水平

### Layer 5: DeFi 生态影响
- Stablecoin 流出: -$580M (USDT+USDC 合计)
- DeFi TVL 推断: ↓ (中置信度)
- 跨源验证: stablecoin 流出方向 vs 协议 TVL 变化 = 一致
- 因果推理: stablecoin 外流 → DeFi 可用流动性减少 → 借贷利用率被动上升 → 短期借贷利率上行 → 但如果同时借款需求也下降（因恐慌）→ 利用率可能持平
- 失效条件: 此 TVL 下行判断在以下条件下失效: USDT/USDC 在 24h 内出现 >$200M 的 DeFi 协议净流入

## 综合评估
- 对 DeFi 收益策略: 短期借贷利率可能上行，但高波动率下无常损失风险增大，建议降低 LP 仓位。
  - 失效条件: 原油 48h 内回落至事件前 + stablecoin 回流 DeFi >$200M
- 对加密持仓: 中期偏空，但空头挤压风险存在。
  - 失效条件: BTC 站稳 $70,000 + 资金费率转正 + stablecoin 止流出

## 跨源验证摘要
| 验证对 | 源 A | 源 B | 结果 |
|--------|-----|------|------|
| 链上内部 | BTC netflow | BTC reserve | 一致 |
| Stablecoin | USDT | USDC | 一致 |
| 技术 vs 链上 | 资金费率 | netflow | 一致 |
| DeFi vs 链上 | TVL 推断 | stablecoin 流向 | 一致 |
| 叙事 vs 数据 | 帖子情绪 | MCP 宏观 | 一致 |
```

## 反模式

1. **不要做方向性交易建议**: 此 skill 评估影响，不预测价格。传导链分析不等于交易信号。
2. **不要忽视置信度标注**: 传导链越长，每层误差累积越大。必须对每层标注置信度。
3. **不要将相关性当因果性**: 交易所资金流入与价格下跌的时间重叠不等于因果关系。每层传导必须解释 "为什么" 而非仅呈现相关数据。
4. **不要遗漏 stablecoin 数据**: Stablecoin 链上流向是 DeFi TVL 变化的最佳领先指标。
5. **数据延迟风险**: MCP 数据可能有 15-60 分钟延迟，报告应标注数据时间戳。
6. **不覆盖**: 期权定价、杠杆倍数建议、具体协议智能合约风险评估。
7. **禁止单源结论**: 任何标注为 "高置信度" 的结论必须有 >=2 个独立 MCP 源交叉验证。如果只有 1 个数据源支持，最高只能标注 "中置信度"。
8. **禁止无失效条件的结论**: 综合评估中的每个核心结论必须附带至少 1 个具体、可观测的失效条件。"如果市场变化" 不是有效的失效条件。
9. **禁止跳过因果层**: 不能从 "事件发生" 直接跳到 "DeFi TVL 下降"。必须逐层推理并标注每层的 "为什么"。
