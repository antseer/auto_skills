# CeFi-DeFi 收益率利差实时监控器 — 需求文档

## 需求概述

CLARITY Act 等监管立法正在模糊 CeFi 和 DeFi 的边界。当 Schwab、Fidelity 等传统机构开始提供受监管的稳定币收益产品时，DeFi 协议面临资金虹吸风险——风险厌恶型资金可能从 Aave/Compound 回流 CeFi。同时，利差变化创造了新的套利机会。用户目前每周手动对比利率，但这可能变成每天甚至更高频的工作。

需要一个实时对比 DeFi 借贷协议利率与传统/CeFi 同类产品收益率，追踪 stablecoin 链上流向作为资金迁移领先指标的利差分析工具。

## 用户画像

### Persona 1: Carol — DeFi 收益策略研究员
- **角色**: 全职 DeFi 分析师，研究收益策略
- **日常工作流**: 对比多个 DeFi 协议利率 → 评估风险调整收益 → 撰写策略报告
- **工具栈**: Python, DefiLlama API, Dune Analytics, Claude Code
- **痛点**: 目前每周手动对比 Aave USDC 借贷利率和 Money Market 基金收益率。CLARITY Act 后这可能变成每天的工作，手动对比不可持续。

### Persona 2: Dave — 加密期权/衍生品交易员
- **角色**: 衍生品交易者，关注利率套利机会
- **日常工作流**: 监控 IV/Greeks → 寻找跨市场套利 → 执行交易
- **工具栈**: Python, Deribit API, TradingView, Claude Code
- **痛点**: 想在 CeFi-DeFi 利差扩大时做收敛套利，但缺乏实时利差监控和利率衍生品流动性信息。

## 场景表

| 场景 | 触发条件 | 期望产出 |
|------|----------|----------|
| 监管法案影响评估 | CLARITY Act 等监管法案通过后 | DeFi 收益率竞争力评估 + 资金迁移信号 |
| 机构入场冲击 | 传统机构宣布推出加密/稳定币收益产品 | CeFi vs DeFi 利率对比 + 资金虹吸风险评估 |
| 资金部署决策 | 需要决定在 CeFi 还是 DeFi 部署稳定币资金 | 风险调整后利差分析 + 建议 |
| 利差套利机会 | 寻找 CeFi-DeFi 利差套利机会 | 利差大小 + 执行成本 + 净套利空间评估 |

## 用户故事

1. **As Carol**, I want 自动获取 DeFi 顶级协议（Aave/Compound/Lido）的稳定币借贷利率与 CeFi/TradFi 同类产品的收益率对比, so that 我能在 CLARITY Act 后每天快速评估 DeFi 收益策略是否仍有竞争力。
2. **As Dave**, I want 实时监控 CeFi-DeFi 利差变化和 stablecoin 链上资金流向, so that 我能在利差异常时发现套利机会并评估可用的利率衍生品工具。

## 讨论引言

该需求源自讨论中对 CLARITY Act 监管变化的集中关注。Carol 指出监管法案可能将其手动利率对比工作从每周变成每天，急需自动化；Dave 则看到 CeFi-DeFi 利差收敛/扩大中的套利机会但缺乏监控工具。两者共同指向"CeFi-DeFi 边界模糊化"这一结构性趋势下的工具空白。

## 需求边界

### 包含
- DeFi 协议利率获取（通过市场指标和话题数据）
- CeFi/TradFi 收益率基准估算（基于联邦基金利率、国债收益率）
- Stablecoin 链上资金流向追踪（USDT/USDC）
- 宏观利率基准获取
- 监管动态上下文
- 利差计算与套利可行性评估

### 不包含
- DeFi 借贷策略构建
- 杠杆交易建议
- 具体 CeFi 产品推荐
- CeFi 产品实时利率数据（当前 MCP 不提供）
