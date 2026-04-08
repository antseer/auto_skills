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

### Step 6: 构建利差分析报告
计算关键利差:
- **核心利差** = DeFi 借贷利率 - CeFi 估算收益率
- **风险调整利差** = 核心利差 - DeFi 智能合约风险溢价
- **资金迁移信号** = stablecoin 净流向方向和量级

调查可用的 DeFi 利率衍生品工具（仅当核心利差 > 50bp 时执行）:
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
| 对比 | 利差 | 风险调整后利差 | 信号 |
|------|------|-------------|------|

---

## Stablecoin 资金流向（资金迁移信号）
[资金流向表格 + 综合判断]

## 监管动态
[最新进展 + 对利差的潜在影响]

## 套利机会评估
[可行性 + 利差大小 + 执行成本 + 净套利空间]

## 利率衍生品工具（仅当核心利差 > 50bp 时展示）
| 工具 | 类型 | 当前利率/赔率 | 流动性深度 | 适用策略 |
|------|------|------------|----------|---------|
| Pendle PT-USDC | 固定收益代币 | [当前隐含固定利率] | [TVL/深度] | 锁定 DeFi 固定收益，规避利率下行风险 |
| Pendle YT-USDC | 浮动收益代币 | [杠杆收益敞口] | [TVL/深度] | 做多 DeFi 利率，押注利差扩大 |
| 利率互换 | 固定-浮动互换 | [互换价格] | [可用平台] | CeFi-DeFi 利差收敛套利 |

## 建议
- 收益优先用户: [建议]
- 风险规避用户: [建议]
- 套利交易者: [建议（含利率衍生品工具建议）]

## 局限性声明
CeFi 收益率为基于宏观基准的估算值。DeFi 利率波动频繁，展示的是快照值。Stablecoin 资金流向是代理指标而非直接证据。本报告不构成投资建议。
```

## 使用示例

**输入:**
```
CLARITY Act 通过了，Schwab 可能要推稳定币理财。我在 Aave 存的 USDC 要不要搬出来？
```

**输出:** 生成利差分析报告，Aave USDC 5.2% vs Money Market ~4.45% vs Schwab 估算 ~3.8-4.2%。核心利差 +75-140bp，风险调整后 +25-90bp。USDC 7天流出 -$480M 显示初步 DeFi→CeFi 迁移信号。建议: 收益优先用户可继续持有（利差仍正），风险规避用户考虑部分迁移。

## 反模式

1. **不要直接对比 APY 而忽视风险差异**: CeFi 4%（受监管）和 DeFi 5%（智能合约风险）不能简单相减。
2. **不要假设 CeFi 利率是固定的**: CeFi 产品利率尚未公布时必须明确标注为"估算值"。
3. **不要将 stablecoin 流出直接等同于 DeFi→CeFi 迁移**: 资金可能流向 OTC、冷钱包或其他链。
4. **不要忽视执行成本**: DeFi→CeFi 迁移的 Gas 费、桥接费、时间成本会吞掉部分利差。
5. **CeFi 数据限制**: 当前 MCP 不提供 CeFi 产品实时利率，CeFi 利率基于宏观基准估算。
6. **不覆盖**: DeFi 借贷策略构建、杠杆交易建议、具体 CeFi 产品推荐。
