[README.md](https://github.com/user-attachments/files/26233840/README.md)
# Spot CP Monitor Skill

Binance Spot Market Share (CP = Competitive Position) 监控知识库 + 日报/周报自动化生成。

## 安装

```bash
openclaw skill install ./spot-cp-monitor
```

或从 git 安装：
```bash
openclaw skill install https://github.com/<org>/spot-cp-monitor-skill
```

## 依赖

- **bnquery-byc**: BQuery API 查询工具（用于执行 SQL）
- **weasso-skill**: SSO 认证（BQuery 登录需要）
- **byc-skill**: BYC 网络通道

## 功能

### Daily CP Report
- 4 层下钻：Exchange CP → VIP/Retail → VIP Sub-Level/UID → Token
- 自动拉取数据并格式化输出

### Weekly CP Report
- Exchange-level WoW 对比
- Token contribution（使用预计算表 `spot_token_mkt_share_chg_1d`）
- Top Gainers / Losers 结构化分析
- 宏观 context 栏位（手动补充）

### 数据表

| 表 | 用途 |
|----|------|
| `bnb_dwa.dwa_main_spot_cp` | CP 主表（exchange + token level） |
| `bnb_dwa.spot_token_mkt_share_chg_1d` | Rolling 7d token contribution（预计算） |
| `bnb_dwa.spot_cp_hexa_source_1d` | VIP/Retail/Country/UID 下钻 |

### 默认口径
- **excl. DEX** (`exchange_name != 'DEX'`)
- **After discount** (使用 `vol` 列 = 折扣后交易量)

## 文件结构

```
spot-cp-monitor/
  SKILL.md          ← 完整知识库 + SQL 模板 + 报告格式
```

## Confluence 文档

Discount Logic + Change History 详见 Confluence Space CDA:
- Page: "Spot CP Volume Logic & Change History" (ID: 540389663)
- Parent: "2. Spot CP Calculation" (ID: 211571873)
