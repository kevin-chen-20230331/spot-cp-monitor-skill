# Spot CP Monitor Skill

> Binance Spot Market Share (CP = Competitive Position) 监控知识库 + 自动化日报/周报生成

## 概念

**CP = Competitive Position** = Binance Spot 在全市场的交易量占比（market share）。现货运营最核心指标。

**默认汇报版本：excl. DEX + after discount**（`exchange_name != 'DEX'` + 使用 `vol` 列，即折扣后交易量）

---

## 1. 维度 (Dimensions)

| 维度 | 说明 |
|------|------|
| `exchange_name` | 交易所名称（标准化后） |
| `is_dex` | 是否 DEX。**默认只看 CEX** (`exchange_name != 'DEX'`) |
| `symbol` | 交易对，如 BTCUSDT |
| `base` | 基础币，如 BTC |
| `quote` | 计价币，如 USDT |
| `is_0_fee` | 是否零手续费交易对 |

---

## 2. 核心指标 (Metrics)

### 最关键的 4 个指标

| 指标 | 含义 | 重要度 |
|------|------|--------|
| **`vol`** (= `volume_with_discount`) | 折扣后交易量 | ⭐⭐⭐ |
| **`Spot_cp_offcial`** | 官方 CP（折扣后） | ⭐⭐⭐ |
| `vol` excl. 0 fee | 折扣后，排除零手续费 | ⭐⭐ |
| `Spot_cp_offcial_no_0fee` | CP 排除零手续费 | ⭐⭐ |

### dwa_main_spot_cp 关键列

| 列名 | 含义 |
|------|------|
| `vol` | 折扣后交易量（= volume_with_discount） |
| `vol_original` | 原始交易量（未折扣） |
| `cp` | 折扣后 CP |
| `exchange_name` | 交易所名称（含 'DEX'） |
| `base` | 基础币 |
| `quote` | 计价币 |
| `date_key` | 日期分区，YYYYMMDD 整数 |

---

## 3. 数据链路 (Data Pipeline)

```
Raw Data Source (CMC)
  └─ bnb_dws.fact_cmc_exchange_s              ← CoinMarketCap 原始数据
       │
Layer 1 (加工层)
  └─ bnb_dwa.dwa_main_spot_cp                 ← CP 主表（discount + symbol 剔除）
       │
Layer 2 (监控/分析层)
  ├─ bnb_dwa.spot_token_mkt_share_chg_1d      ← Rolling 7d token contribution（预计算）
  └─ bnb_dwa.spot_cp_hexa_source_1d           ← VIP/Retail/Country/UID 下钻
```

### 表速查

| 需求 | 表 | 分区 |
|------|-----|------|
| 原始 CMC 数据 | `bnb_dws.fact_cmc_exchange_s` | `date_key` |
| CP 主表（exchange + token level） | `bnb_dwa.dwa_main_spot_cp` | `date_key` (YYYYMMDD int) |
| Weekly token contribution（预计算） | `bnb_dwa.spot_token_mkt_share_chg_1d` | `date_key` (YYYYMMDD int) |
| VIP/Retail/Country/UID 下钻 | `bnb_dwa.spot_cp_hexa_source_1d` | `dt` (YYYY-MM-DD string) |

### spot_token_mkt_share_chg_1d 结构

| 列 | 含义 |
|----|------|
| `week_period` | 周期标签，如 "2026-03-17 - 2026-03-23" |
| `exchange_name` | 交易所名称 |
| `vol` | 该所 7d 总交易量 |
| `contri_type` | `top` = gainers, `bottom` = losers |
| `paste_content` | 预格式化文本，如 "BTC +0.43%, XAUt +0.22%, WBTC +0.1%" |
| `date_key` | 分区日期 |

> 每个交易所每天 2 行（top + bottom），rolling 7d 计算

### spot_cp_hexa_source_1d 结构

| 列 | 含义 |
|----|------|
| `dt` | 日期（YYYY-MM-DD 字符串） |
| `exchange_name_1` | 交易所（通常 filter `= 'Binance'`） |
| `dimension_1` | 第一维度：`VIP` / `Retail` / `all` |
| `dimension_2` | 第二维度：`Token` |
| `value_1` | dimension_1 细分值（VIP level 如 "VIP 2-6"、UID 如 "20***133"、region） |
| `value_2` | dimension_2 细分值（token name） |
| `discount_vol` | Binance 折扣后交易量 |
| `all_exchange_discount_vol` | 全市场折扣后交易量（已 excl. DEX） |

---

## 4. Daily 监控

### 报告格式

```
Spot Daily CP ({date}) — excl. DEX

1). Binance CP DoD {chg}% (from {prev}% to {cur}%).

Top 3 CP increasing exchanges:
{Exchange} CP +{x}% (from {a}% to {b}%)
...

Top 3 CP decreasing exchanges:
{Exchange} CP -{x}% (from {a}% to {b}%)
...

2). For Binance, drill down to VIP/Retail:
For VIP, DoD {chg}% (from {a}% to {b}%)
For Retail, DoD {chg}% (from {a}% to {b}%)

3). Then further drill down (VIP sub-level):
{VIP_level_or_UID}, DoD {chg}% (from {a}% to {b}%)
...

4). By Token:
{Token}, DoD {chg}% (from {a}% to {b}%)
...
```

### SQL Queries

#### Q1: Exchange CP DoD (excl. DEX)
```sql
WITH yesterday AS (
  SELECT exchange_name,
    SUM(vol) AS vol,
    SUM(vol) / NULLIF(SUM(SUM(vol)) OVER (), 0) AS cp
  FROM bnb_dwa.dwa_main_spot_cp
  WHERE date_key = {date_key} AND exchange_name != 'DEX'
  GROUP BY exchange_name
),
day_before AS (
  SELECT exchange_name,
    SUM(vol) AS vol,
    SUM(vol) / NULLIF(SUM(SUM(vol)) OVER (), 0) AS cp
  FROM bnb_dwa.dwa_main_spot_cp
  WHERE date_key = {prev_date_key} AND exchange_name != 'DEX'
  GROUP BY exchange_name
)
SELECT y.exchange_name,
  ROUND(y.vol/1e9, 2) AS vol_bn,
  ROUND(y.cp*100, 2) AS cp_pct,
  ROUND(d.cp*100, 2) AS prev_cp_pct,
  ROUND((y.cp - d.cp)*100, 2) AS cp_chg_pp
FROM yesterday y
LEFT JOIN day_before d ON y.exchange_name = d.exchange_name
ORDER BY y.cp DESC
```

#### Q2: VIP/Retail Breakdown
```sql
WITH agg AS (
  SELECT dt, dimension_1,
    SUM(discount_vol) AS bnb_vol,
    MAX(all_exchange_discount_vol) AS mkt_vol
  FROM bnb_dwa.spot_cp_hexa_source_1d
  WHERE dt IN ('{date}', '{prev_date}')
    AND exchange_name_1 = 'Binance'
    AND dimension_1 IN ('VIP', 'Retail')
    AND dimension_2 = 'Token'
  GROUP BY dt, dimension_1
)
SELECT dt, dimension_1,
  ROUND(bnb_vol/1e9, 2) AS bnb_vol_bn,
  ROUND(bnb_vol/NULLIF(mkt_vol,0)*100, 2) AS cp_pct
FROM agg ORDER BY dt, dimension_1
```

#### Q3: VIP Sub-Level + Top UIDs
```sql
WITH vip_detail AS (
  SELECT dt, value_1,
    SUM(discount_vol) AS bnb_vol,
    MAX(all_exchange_discount_vol) AS mkt_vol
  FROM bnb_dwa.spot_cp_hexa_source_1d
  WHERE dt IN ('{date}', '{prev_date}')
    AND exchange_name_1 = 'Binance'
    AND dimension_1 = 'VIP'
    AND dimension_2 = 'Token'
  GROUP BY dt, value_1
),
ranked AS (
  SELECT *, ROUND(bnb_vol/NULLIF(mkt_vol,0)*100, 2) AS cp_pct,
    ROW_NUMBER() OVER (PARTITION BY dt ORDER BY bnb_vol DESC) AS rn
  FROM vip_detail
)
SELECT dt, value_1, ROUND(bnb_vol/1e9, 2) AS vol_bn, cp_pct
FROM ranked WHERE rn <= 10 ORDER BY dt, rn
```

#### Q4: Token CP Contribution
```sql
WITH token_cp AS (
  SELECT dt, value_2 AS token,
    SUM(discount_vol) AS bnb_vol,
    MAX(all_exchange_discount_vol) AS mkt_vol
  FROM bnb_dwa.spot_cp_hexa_source_1d
  WHERE dt IN ('{date}', '{prev_date}')
    AND exchange_name_1 = 'Binance'
    AND dimension_2 = 'Token'
  GROUP BY dt, value_2
),
pivot AS (
  SELECT token,
    MAX(CASE WHEN dt='{date}' THEN ROUND(bnb_vol/NULLIF(mkt_vol,0)*100,2) END) AS cp_today,
    MAX(CASE WHEN dt='{prev_date}' THEN ROUND(bnb_vol/NULLIF(mkt_vol,0)*100,2) END) AS cp_prev,
    MAX(CASE WHEN dt='{date}' THEN ROUND(bnb_vol/NULLIF(mkt_vol,0)*100,2) END)
    - MAX(CASE WHEN dt='{prev_date}' THEN ROUND(bnb_vol/NULLIF(mkt_vol,0)*100,2) END) AS chg
  FROM token_cp GROUP BY token
  HAVING MAX(CASE WHEN dt='{date}' THEN bnb_vol END) > 0
)
SELECT * FROM pivot ORDER BY ABS(chg) DESC LIMIT 10
```

---

## 5. Weekly 监控

### 报告格式

```
Spot CP Commentary (Week of {week_label}) — excl. DEX

**1. Binance Overview**
> 1.1 Binance CP {cp}% ({chg}%)
> 1.2 Gain on: {Token1} +{x}%, {Token2} +{x}%, {Token3} +{x}%
> 1.3 Lose on: {Token1} -{x}%, {Token2} -{x}%, {Token3} -{x}%
> 1.4 Binance (7d): {本周 Binance 新闻/公告}

**2. Market Context**
Global Market Volume (CEX) {chg}% to ${total}B, Binance Volume {chg}% to ${bnb}B
> 2.1 {宏观: Fed/rates/geopolitical}
> 2.2 {BTC ETF flows}
> 2.3 {BTC technicals: price range, CVD, premium}
> 2.4 {ETH/altcoin sentiment}

**3. Top Gainers:** {Ex1} (+{x}%), {Ex2} (+{x}%), {Ex3} (+{x}%)
> 3.1 {Exchange}: {paste_content from top}
>     {Exchange}: {新闻/事件}
> 3.2 ...

**4. Top Losers:** {Ex1} (-{x}%), {Ex2} (-{x}%), {Ex3} (-{x}%)
> 4.1 {Exchange}: {paste_content from bottom}
>     {Exchange}: {新闻/事件}
> 4.2 ...
```

### SQL Queries

#### Q1: Exchange CP WoW (excl. DEX)
```sql
WITH this_week AS (
  SELECT exchange_name,
    SUM(vol) AS vol,
    SUM(vol) / NULLIF(SUM(SUM(vol)) OVER (), 0) AS cp
  FROM bnb_dwa.dwa_main_spot_cp
  WHERE date_key BETWEEN {wk_start} AND {wk_end} AND exchange_name != 'DEX'
  GROUP BY exchange_name
),
last_week AS (
  SELECT exchange_name,
    SUM(vol) AS vol,
    SUM(vol) / NULLIF(SUM(SUM(vol)) OVER (), 0) AS cp
  FROM bnb_dwa.dwa_main_spot_cp
  WHERE date_key BETWEEN {prev_start} AND {prev_end} AND exchange_name != 'DEX'
  GROUP BY exchange_name
)
SELECT t.exchange_name,
  ROUND(t.vol/1e9, 2) AS vol_bn,
  ROUND(l.vol/1e9, 2) AS prev_vol_bn,
  ROUND((t.vol-l.vol)/NULLIF(l.vol,0)*100, 1) AS vol_chg_pct,
  ROUND(t.cp*100, 2) AS cp_pct,
  ROUND(l.cp*100, 2) AS prev_cp_pct,
  ROUND((t.cp-l.cp)*100, 2) AS cp_chg_pp
FROM this_week t
LEFT JOIN last_week l ON t.exchange_name = l.exchange_name
ORDER BY t.cp DESC
```

#### Q2: Market Total Volume WoW (excl. DEX)
```sql
SELECT
  CASE WHEN date_key BETWEEN {wk_start} AND {wk_end} THEN 'this_wk' ELSE 'last_wk' END AS wk,
  ROUND(SUM(vol)/1e9, 2) AS total_vol_bn,
  ROUND(SUM(CASE WHEN exchange_name='Binance' THEN vol END)/1e9, 2) AS bnb_vol_bn
FROM bnb_dwa.dwa_main_spot_cp
WHERE date_key BETWEEN {prev_start} AND {wk_end} AND exchange_name != 'DEX'
GROUP BY 1 ORDER BY 1
```

#### Q3: Token Contribution (pre-computed, rolling 7d)
```sql
SELECT week_period, exchange_name, vol, contri_type, paste_content, date_key
FROM bnb_dwa.spot_token_mkt_share_chg_1d
WHERE date_key = {wk_end}
```
> `contri_type = 'top'` → gainers, `'bottom'` → losers
> `paste_content` = 预格式化 "TOKEN +x%, TOKEN -y%"
> 每个交易所 2 行（top + bottom）

---

## 6. 注意事项

- **默认 excl. DEX**: 所有 SQL 加 `AND exchange_name != 'DEX'`
- **hexa_source 表分母**: `all_exchange_discount_vol` 已经是 excl. DEX 的全市场总量
- **UID 脱敏**: VIP drill-down 中 UID 显示为 `20***133` 格式
- **日期格式差异**: `dwa_main_spot_cp` 用 `date_key` (YYYYMMDD int), `hexa_source` 用 `dt` (YYYY-MM-DD string)
- **Trino 5 分钟限制**: 复杂查询可能超时，ETL 建表用 Spark
- **Confluence 文档**: 详细的 Discount Logic + Change Log 在 Confluence Space CDA, pageId `540389663`

---

## 7. Discount Logic 概要

折扣用于调整竞对的虚假交易量。核心逻辑：对特定交易所的特定交易对/全局应用折扣系数。

详见 Confluence 页面 "Spot CP Volume Logic & Change History" (pageId: `540389663`)

**当前活跃折扣 (截至 2025-10-07):**
- MEXC: 70% (全局)
- KuCoin: 70% (全局)
- Gate.io: 63% (全局)
- Crypto.com: 75.5% (全局)
- Huobi: 58% (全局)
- Bitget: 50.1% (全局)
- Binance/Bybit/OKX/Coinbase: 无折扣（CLEAN）

---

*Last updated: 2026-03-24 by 小K 🦊*
