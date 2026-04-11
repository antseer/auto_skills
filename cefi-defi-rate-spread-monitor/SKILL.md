---
name: cefi-defi-rate-spread-monitor
description: 实时对比 DeFi 借贷利率与 CeFi/TradFi 收益率，追踪 stablecoin 链上流向，输出利差分析和套利机会评估。
---

# CeFi-DeFi 收益率利差实时监控器

## 用户画像

### Carol — DeFi 收益策略研究员
- **日常工作流**: 每天用 Claude Code 抓取 DeFi 协议数据（TVL、APY、资金池深度），对比不同链和协议的收益率，评估无常损失风险，撰写收益策略报告
- **工具栈**: Python、Dune Analytics、DefiLlama API、Claude Code、Jupyter Notebook
- **痛点**: 手动对比 Aave USDC 借贷利率和 Money Market 基金收益率每周需要 1-2 小时；CLARITY Act 之后监管驱动的 CeFi 产品入场会使这成为每天的工作，人工完全跟不上

### Dave — 加密期权/衍生品交易员
- **日常工作流**: 监控 BTC/ETH 期权 IV、资金费率，分析链上借贷利率与 CeFi 产品利率的利差，寻找 DeFi 利率衍生品（Pendle PT/YT）的套利机会
- **工具栈**: Python、Deribit API、Greeks.live、TradingView、Claude Code
- **痛点**: 没有工具能实时追踪 CeFi-DeFi 利差变化并列出可用的利率衍生品工具（Pendle）及其流动性深度，导致错失利差扩大时的套利窗口

---

## 用户故事

1. **As Carol**, I want to see a real-time side-by-side table of DeFi protocol rates vs CeFi/Money Market benchmarks, so that I can immediately assess whether CLARITY Act-driven CeFi products are eroding DeFi yield advantages without spending 1-2 hours on manual comparisons.

2. **As Dave**, I want the skill to surface available DeFi rate derivative instruments (Pendle PT/YT) with liquidity depth when the CeFi-DeFi spread exceeds 50bp, so that I can execute rate arbitrage strategies before the spread converges.

---

## 功能说明

当监管政策变化或宏观事件可能导致 CeFi-DeFi 资金重新分配时，自动对比 DeFi 借贷协议利率与传统/CeFi 同类产品收益率（基于宏观基准估算），并追踪 stablecoin 链上流向作为资金迁移领先指标，输出利差分析、资金迁移信号和套利机会评估。替代用户手动的每周/每日利率对比。

## 适用场景

- 监管法案（如 CLARITY Act）通过后评估对 DeFi 收益格局的影响
- 传统机构宣布推出加密/稳定币收益产品时
- 需要决定在 CeFi 还是 DeFi 部署稳定币资金时
- 寻找 CeFi-DeFi 利差套利机会时

### 触发短语
- "Schwab 要推出稳定币理财了，Aave 还值得存吗？"
- "CeFi 和 DeFi 的稳定币利率差多少？"
- "CLARITY Act 之后 stablecoin 有没有从链上流出？"
- `/cefi-defi-yield-spread --stablecoin USDC --defi-protocols aave,compound`

## 输入参数

| 参数 | 必填 | 说明 | 默认值 |
|------|------|------|--------|
| stablecoin | 否 | 关注的稳定币 | USDC |
| defi-protocols | 否 | 关注的 DeFi 协议列表 | Aave, Compound |
| 监管事件 | 否 | 相关监管背景描述 | 自动从话题中检测 |

## 执行步骤

### Step 1: 获取 DeFi 协议利率相关市场数据
调用 `ant_market_sentiment(query_type='topics_list')` 搜索 Aave、Compound、Lido 等 DeFi 协议话题。
对每个匹配话题调用 `ant_market_sentiment(query_type='topic_detail', topic_id=<id>)` 获取协议最新情况。
调用 `ant_market_sentiment(query_type='topic_time_series', topic_id=<id>)` 获取话题热度趋势。

### Step 2: 获取 stablecoin 链上资金流向（资金迁移领先指标）
调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDT')` 获取 USDT 交易所净流向。
调用 `ant_fund_flow(query_type='exchange_netflow', coin='USDC')` 获取 USDC 交易所净流向。
调用 `ant_fund_flow(query_type='exchange_reserve', coin='USDT')` 获取 USDT 交易所储备。
调用 `ant_fund_flow(query_type='exchange_reserve', coin='USDC')` 获取 USDC 交易所储备。
调用 `ant_fund_flow(query_type='exchange_inflow', coin='USDC')` 获取 USDC 流入交易所数据。
调用 `ant_fund_flow(query_type='exchange_outflow', coin='USDC')` 获取 USDC 流出交易所数据。
调用 `ant_stablecoin(query_type='stablecoin_supply')` 获取 stablecoin 链上总供给变化（交叉验证资金迁移方向）。

### Step 3: 获取宏观利率基准
调用 `ant_macro_economics(query_type='federal_funds_rate')` 获取联邦基金利率。
调用 `ant_macro_economics(query_type='treasury_yield')` 获取国债收益率。
调用 `ant_macro_economics(query_type='expectation')` 获取利率预期。
CeFi 稳定币收益产品利率通常锚定联邦基金利率 - 管理费。

### Step 4: 估算 CeFi vs DeFi 利率对比
**CeFi 利率估算**（基于宏观基准）:
- Money Market 基金收益率 ≈ 联邦基金利率 - 0.2% ~ 0.5%
- 机构稳定币收益产品 ≈ 联邦基金利率 - 0.5% ~ 1.0%

**DeFi 利率获取**:
调用 `ant_market_indicators(query_type='borrow_rate', coin='USDC')` 获取链上 USDC 借贷利率参考。
如无直接借贷利率数据，使用资金费率作为代理:
调用 `ant_futures_market_structure(query_type='futures_funding_rate', coin='BTC')` — 正资金费率暗示借贷利率上行。

### Step 5: 获取监管动态上下文
调用 `ant_market_sentiment(query_type='topics_list')` 搜索监管话题。
对匹配话题调用 `ant_market_sentiment(query_type='topic_news', topic_id=<id>)` 获取最新监管新闻。
调用 `ant_market_sentiment(query_type='topic_posts', topic_id=<id>)` 获取社区对监管变化的反应。

### Step 6: 多层因果链分析（Multi-Layer Causal Chain）

**第一层：利差计算（What）**
计算关键利差:
- **核心利差** = DeFi 借贷利率 - CeFi 估算收益率
- **风险调整利差** = 核心利差 - DeFi 智能合约风险溢价
- **资金迁移信号** = stablecoin 净流向方向和量级

**第二层：驱动因素分析（Why）**
对每个利差数值进行因果归因:
- 利差扩大时：是因为 DeFi 利率上升（链上借贷需求增加？杠杆多头拥挤？协议激励提升？）还是 CeFi 利率下降（降息预期？机构竞争加剧？）？
- 利差收窄时：是因为监管驱动的资金迁移（CLARITY Act 等法案使 CeFi 产品合规化吸引资金回流）？还是 DeFi 利率周期性回落（杠杆清洗后借贷需求降温）？
- stablecoin 流向异常时：是散户恐慌性提取（检查社交情绪）？还是机构级别的系统性再配置（检查大额转账地址画像）？

**第三层：传导后果推演（Consequence）**
基于驱动因素推演后续影响:
- 如果利差扩大由杠杆拥挤驱动 → 后果：清算风险累积，利差可能突然反转
- 如果利差收窄由监管资金迁移驱动 → 后果：DeFi 协议 TVL 下降 → 借贷利用率变化 → 可能触发利率机制调整
- 如果 stablecoin 大规模单向流动 → 后果：目标平台流动性深度变化 → 滑点和执行成本改变

**第四层：结构性 vs 周期性判定**
综合以上三层，判定当前利差变动属于:
- **结构性变化**（监管制度转变、新产品类别入场）→ 利差中枢可能永久性偏移，套利策略需调整基准
- **周期性波动**（杠杆周期、市场情绪摆动）→ 利差可能均值回归，可执行均值回归策略

### Step 7: 交叉验证与置信度校准（Cross-Source Verification）

在输出任何结论之前，执行以下交叉验证:

**验证矩阵（至少 2 个独立数据源必须一致才能标记"高置信度"）:**

| 结论 | 数据源 1 | 数据源 2 | 数据源 3（可选） | 置信度规则 |
|------|---------|---------|---------------|-----------|
| DeFi 利率趋势 | `ant_market_indicators` 借贷利率 | `ant_futures_market_structure` 资金费率方向 | `ant_protocol_tvl_yields_revenue` TVL 变化 | 3源一致=高, 2源一致=中, 分歧=低 |
| CeFi 利率趋势 | `ant_macro_economics` 联邦基金利率 | `ant_macro_economics` 国债收益率 | `ant_market_sentiment` 利率预期话题 | 同上 |
| 资金迁移方向 | `ant_fund_flow` 交易所净流向 | `ant_stablecoin` 链上供给变化 | `ant_fund_flow` 交易所储备 | 同上 |
| 套利可行性 | 利差数值 > 50bp | `ant_protocol_tvl_yields_revenue` Pendle 流动性 | stablecoin 流向支持方向 | 全部满足=可行, 任一不满足=标注风险 |

**置信度降级规则:**
- 如果 DeFi 利率源和资金费率方向矛盾 → 置信度强制降至"低"，并在输出中标注"信号冲突：借贷利率与资金费率方向不一致，建议观望"
- 如果 stablecoin 链上流向与交易所储备变化矛盾 → 资金迁移结论置信度降至"低"
- 如果宏观利率预期（联邦基金利率路径）与市场情绪话题（社区讨论方向）矛盾 → CeFi 利率趋势置信度降至"低"

### Step 8: 构建利差分析报告

调查可用的 DeFi 利率衍生品工具（仅当核心利差 > 50bp 且套利可行性交叉验证通过时执行）:
调用 `ant_protocol_tvl_yields_revenue(query_type='protocol_yields', protocol='pendle')` 获取 Pendle PT/YT 收益代币的当前利率锁定报价和流动性深度。
如无直接数据，记录 Pendle PT（固定收益端）和 YT（浮动收益端）作为可用的利率锁定/套利工具，并注明需用户自行查验当前流动性。
评估利率互换可行性：DeFi 利率互换平台（如 Pendle Finance）允许在锁定 DeFi 固定收益（PT）和保持 CeFi 浮动收益之间套利。

## 输出格式

```markdown
# CeFi-DeFi 收益率利差分析

## 分析时间: [YYYY-MM-DD]
## 基准 Stablecoin: [USDC/USDT]
## 监管背景: [当前监管状态简述]

---

## 利率对比

### CeFi / TradFi 收益率基准
| 产品类型 | 估算收益率 | 来源/基准 | 风险等级 |
|---------|-----------|----------|---------|

### DeFi 协议收益率
| 协议 | 稳定币产品 | 当前利率 | 趋势 | 风险等级 |
|------|-----------|---------|------|---------|

### 核心利差
| 对比 | 利差 | 风险调整后利差 | 信号 | 置信度 |
|------|------|-------------|------|--------|

---

## 利差驱动因素分析（因果链）

### 当前利差变动的"为什么"
| 利差变动方向 | 第一层原因（直接驱动） | 第二层原因（根因） | 传导后果 |
|-------------|---------------------|------------------|---------|
| [扩大/收窄/稳定] | [DeFi利率上升/CeFi利率下降/...] | [杠杆拥挤/监管迁移/降息预期/...] | [清算风险累积/TVL下降/...] |

### 结构性 vs 周期性判定
- **判定**: [结构性变化 / 周期性波动]
- **依据**: [综合因果链分析的判断理由]
- **对策略的含义**: [结构性→调整基准；周期性→均值回归策略可行]

---

## 交叉验证与置信度校准

| 结论 | 数据源 1 | 数据源 2 | 数据源 3 | 一致性 | 最终置信度 |
|------|---------|---------|---------|--------|-----------|
| DeFi利率趋势 | [借贷利率: ↑/↓] | [资金费率: ↑/↓] | [TVL变化: ↑/↓] | [一致/分歧] | [高/中/低] |
| CeFi利率趋势 | [联邦基金利率: X%] | [国债收益率: X%] | [市场利率预期: X] | [一致/分歧] | [高/中/低] |
| 资金迁移方向 | [交易所净流向: 流入/流出] | [链上供给变化: 增/减] | [交易所储备: 增/减] | [一致/分歧] | [高/中/低] |

**信号冲突警告**: [如有交叉验证不一致，在此明确标注冲突内容及建议]

---

## Stablecoin 资金流向（资金迁移信号）
[资金流向表格 + 综合判断]

## 监管动态
[最新进展 + 对利差的潜在影响]

## 套利机会评估
[可行性 + 利差大小 + 执行成本 + 净套利空间]

## 利率衍生品工具（仅当核心利差 > 50bp 且交叉验证通过时展示）
| 工具 | 类型 | 当前利率/赔率 | 流动性深度 | 适用策略 |
|------|------|------------|----------|---------|
| Pendle PT-USDC | 固定收益代币 | [当前隐含固定利率] | [TVL/深度] | 锁定 DeFi 固定收益，规避利率下行风险 |
| Pendle YT-USDC | 浮动收益代币 | [杠杆收益敞口] | [TVL/深度] | 做多 DeFi 利率，押注利差扩大 |
| 利率互换 | 固定-浮动互换 | [互换价格] | [可用平台] | CeFi-DeFi 利差收敛套利 |

---

## 失效条件（Invalidation Conditions）

每个核心结论必须附带至少一条失效条件:

| 结论 | 失效条件 | 失效时的应对 |
|------|---------|------------|
| 利差方向判断: [当前判断] | 此判断失效如果: [具体条件，如"联邦基金利率意外降息50bp以上"或"DeFi 协议突发安全事件导致利率异常飙升"] | [应对建议] |
| 资金迁移信号: [当前判断] | 此判断失效如果: [具体条件，如"stablecoin 流出被证实为 OTC 交易而非 CeFi 迁移"或"大额单一地址操作而非散户行为"] | [应对建议] |
| 套利可行性: [当前判断] | 此判断失效如果: [具体条件，如"Pendle 流动性深度不足以支撑目标仓位"或"Gas 费飙升吞掉利差"] | [应对建议] |
| 结构性/周期性判定: [当前判断] | 此判断失效如果: [具体条件，如"监管法案实施被推迟/修改"或"CeFi 产品利率远低于预期"] | [应对建议] |

---

## 建议
- 收益优先用户: [建议] — **失效如果**: [条件]
- 风险规避用户: [建议] — **失效如果**: [条件]
- 套利交易者: [建议（含利率衍生品工具建议）] — **失效如果**: [条件]

## 局限性声明
CeFi 收益率为基于宏观基准的估算值。DeFi 利率波动频繁，展示的是快照值。Stablecoin 资金流向是代理指标而非直接证据。本报告不构成投资建议。交叉验证置信度为"低"的结论应视为初步假设而非可行动信号。
```

## 使用示例

**输入:**
```
CLARITY Act 通过了，Schwab 可能要推稳定币理财。我在 Aave 存的 USDC 要不要搬出来？
```

**输出:** 生成利差分析报告:

**利率对比**: Aave USDC 5.2% vs Money Market ~4.45% vs Schwab 估算 ~3.8-4.2%。核心利差 +75-140bp，风险调整后 +25-90bp。

**因果链分析**: 利差当前偏宽 → 第一层原因: DeFi 借贷需求因杠杆多头拥挤而推高利率（资金费率正且处于7天高位验证）→ 第二层原因: 市场对 CLARITY Act 利好预期推动了链上杠杆扩张 → 传导后果: 若杠杆清洗发生，DeFi 利率可能急速回落导致利差突然收窄。判定: 周期性波动（杠杆驱动而非基本面结构性变化）。

**交叉验证**: DeFi利率趋势 — 借贷利率↑ + 资金费率↑ + TVL微增 = 3源一致, 置信度: 高。资金迁移方向 — 交易所净流出 + 链上USDC供给减少 + 交易所储备下降 = 3源一致, 置信度: 高。

**失效条件**:
- 利差判断失效如果: 联邦基金利率本月意外降息 ≥25bp（将压缩 CeFi 利率使利差进一步扩大而非收窄）
- 资金迁移判断失效如果: USDC 流出被证实为 OTC 大宗交易而非散户向 CeFi 迁移（检查大额转账地址画像）
- 套利判断失效如果: Pendle USDC池 TVL 低于 $10M（流动性不足以执行有意义的利率锁定）

**建议**: 收益优先用户可继续持有（利差仍正）— 失效如果 Aave 利率跌破 4.5%。风险规避用户考虑部分迁移 — 失效如果 Schwab 产品延迟发布。

## 反模式

1. **不要直接对比 APY 而忽视风险差异**: CeFi 4%（受监管）和 DeFi 5%（智能合约风险）不能简单相减。
2. **不要假设 CeFi 利率是固定的**: CeFi 产品利率尚未公布时必须明确标注为"估算值"。
3. **不要将 stablecoin 流出直接等同于 DeFi→CeFi 迁移**: 资金可能流向 OTC、冷钱包或其他链。
4. **不要忽视执行成本**: DeFi→CeFi 迁移的 Gas 费、桥接费、时间成本会吞掉部分利差。
5. **CeFi 数据限制**: 当前 MCP 不提供 CeFi 产品实时利率，CeFi 利率基于宏观基准估算。
6. **不覆盖**: DeFi 借贷策略构建、杠杆交易建议、具体 CeFi 产品推荐。
7. **不要跳过因果链直接给结论**: 禁止从"利差 = X bp"直接跳到"建议做多/做空"。必须先完成第二层（Why）和第三层（Consequence）分析，否则结论缺乏可验证性。
8. **不要仅凭单一数据源输出"高置信度"结论**: 交叉验证矩阵中至少 2 个独立源必须一致。如果仅有 1 个数据源可用，置信度上限为"低"，且必须在输出中标注数据不足。
9. **不要省略失效条件**: 每个核心结论（利差方向、资金迁移、套利可行性、结构性/周期性判定）必须附带至少 1 条具体的"此判断失效如果..."条件。泛泛的"市场变化可能导致判断错误"不算有效失效条件。
