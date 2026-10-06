# Power BI Semantic Model — DAX Measures & Configuration

This document contains all DAX measures, table relationships, and dashboard specifications for the Quantitative Stock Selection System.

---

## 1. Semantic Model Configuration

### Storage Mode
**Direct Lake on OneLake** — reads Delta Parquet files natively with zero refresh latency.

### Tables Included

| Table | Purpose |
|---|---|
| `gold_stock_candidates` | Top 15 ranked stock picks |
| `gold_screen_current` | All screened stocks with scores and current prices |
| `gold_paper_portfolio` | Active simulated holdings |
| `gold_paper_cash` | Simulated cash balance |
| `gold_paper_ledger` | Closed trades with realized P&L |
| `silver_market_regime` | Nifty 50 vs 200-SMA regime status |
| `gold_run_audit` | Pipeline health and data quality summary |
| `backtest_qtr_summary` | Historical strategy performance by style |
| `backtest_qtr_screen` | Per-screen backtest returns |

---

## 2. Table Relationships

| From Table | From Column | To Table | To Column | Cardinality | Cross Filter | Active |
|---|---|---|---|---|---|---|
| `gold_paper_portfolio` | `symbol` | `gold_screen_current` | `symbol` | Many to One (*:1) | Single | ✅ |
| `gold_paper_ledger` | `symbol` | `gold_screen_current` | `symbol` | Many to One (*:1) | Single | ✅ |

### Why This Direction

The `RELATED()` function only resolves from the **Many** side to the **One** side. Your portfolio table (many rows per symbol over time) looks up the current price from the screening table (one row per symbol).

### Setup Steps

1. Open **Model view** in Power BI
2. Drag `symbol` from `gold_paper_portfolio` onto `symbol` in `gold_screen_current`
3. Double-click the relationship line
4. Set Cardinality to **Many to one (*:1)**
5. Set Cross filter direction to **Single**
6. Click **OK**

---

## 3. Paper Trading Account Measures

### Current Cash Balance
```dax
Current Cash = MAX('gold_paper_cash'[cash_balance])
```
Returns the robot's available cash for new purchases.

---

### Total Amount Invested
```dax
Total Invested = SUM('gold_paper_portfolio'[invested_amount])
```
Sum of the cost basis of all active holdings.

---

### Current Portfolio Market Value
```dax
Current Portfolio Value = 
SUMX(
    'gold_paper_portfolio', 
    'gold_paper_portfolio'[quantity] * RELATED('gold_screen_current'[close])
)
```
Multiplies each holding's quantity by today's closing price and sums the result.

**Fallback version** (use if relationship cannot be created):
```dax
Current Portfolio Value = 
SUMX(
    'gold_paper_portfolio',
    'gold_paper_portfolio'[quantity] *
    CALCULATE(
        MAX('gold_screen_current'[close]),
        FILTER(
            ALL('gold_screen_current'),
            'gold_screen_current'[symbol] = EARLIER('gold_paper_portfolio'[symbol])
        )
    )
)
```

---

### Unrealized Profit/Loss
```dax
Unrealized PnL = [Current Portfolio Value] - [Total Invested]
```
Paper gains or losses on positions still held.

---

### Unrealized Return Percentage
```dax
Unrealized PnL Pct = 
DIVIDE([Unrealized PnL], [Total Invested], 0) * 100
```

---

### Total Realized Profit/Loss
```dax
Total Realized PnL = SUM('gold_paper_ledger'[realized_pnl])
```
Cumulative profit or loss from all closed trades.

---

### Total Account Value
```dax
Total Account Value = [Current Cash] + [Current Portfolio Value]
```
Cash plus the market value of holdings. This is your headline number.

---

### Total Return Percentage
```dax
Total Return Pct = 
VAR StartingCapital = 1000000
RETURN
    DIVIDE([Total Account Value] - StartingCapital, StartingCapital, 0) * 100
```
Overall performance since inception, assuming ₹10,00,000 starting capital.

---

### Number of Active Positions
```dax
Active Positions = COUNTROWS('gold_paper_portfolio') + 0
```

---

### Number of Closed Trades
```dax
Closed Trades = COUNTROWS('gold_paper_ledger') + 0
```

---

### Win Rate on Closed Trades
```dax
Closed Trade Win Rate = 
VAR Winners = 
    CALCULATE(
        COUNTROWS('gold_paper_ledger'),
        'gold_paper_ledger'[realized_pnl] > 0
    )
VAR Total = COUNTROWS('gold_paper_ledger')
RETURN
    DIVIDE(Winners, Total, 0) * 100
```

---

### Per-Stock Unrealized P&L (for table visuals)
```dax
Stock PnL = 
SUMX(
    'gold_paper_portfolio', 
    (RELATED('gold_screen_current'[close]) - 'gold_paper_portfolio'[buy_price]) 
    * 'gold_paper_portfolio'[quantity]
)
```
Use this inside a table visual to show profit per holding.

---

### Per-Stock Return Percentage
```dax
Stock Return Pct = 
AVERAGEX(
    'gold_paper_portfolio',
    DIVIDE(
        RELATED('gold_screen_current'[close]) - 'gold_paper_portfolio'[buy_price],
        'gold_paper_portfolio'[buy_price],
        0
    ) * 100
)
```

---

### Days Held
```dax
Days Held = 
AVERAGEX(
    'gold_paper_portfolio',
    DATEDIFF('gold_paper_portfolio'[buy_date], TODAY(), DAY)
)
```

---

## 4. Market Status Measures

### Market Regime Banner
```dax
Market Regime Banner = 
VAR LatestDate = MAX('silver_market_regime'[trade_date])
VAR CurrentRegime = 
    CALCULATE(
        MAX('silver_market_regime'[market_regime]),
        'silver_market_regime'[trade_date] = LatestDate
    )
RETURN
    SWITCH(
        CurrentRegime,
        "RISK_ON", "🟢 RISK-ON (Nifty > 200 SMA) — Bullish",
        "RISK_OFF", "🔴 RISK-OFF (Nifty < 200 SMA) — Defensive",
        "⚪ UNKNOWN"
    )
```

---

### Nifty Distance from 200-SMA
```dax
Nifty vs SMA200 Pct = 
VAR LatestDate = MAX('silver_market_regime'[trade_date])
VAR NiftyClose = 
    CALCULATE(MAX('silver_market_regime'[close]), 'silver_market_regime'[trade_date] = LatestDate)
VAR NiftySMA = 
    CALCULATE(MAX('silver_market_regime'[sma_200]), 'silver_market_regime'[trade_date] = LatestDate)
RETURN
    DIVIDE(NiftyClose - NiftySMA, NiftySMA, 0) * 100
```

---

### Data Pipeline Health
```dax
Engine Health = 
VAR LatestRun = MAX('gold_run_audit'[run_timestamp])
VAR AuditStatus = 
    CALCULATE(
        MAX('gold_run_audit'[overall_status]),
        'gold_run_audit'[run_timestamp] = LatestRun
    )
RETURN
    SWITCH(
        AuditStatus,
        "HEALTHY", "🟢 SYSTEM HEALTHY",
        "DEGRADED", "🟡 WARNINGS PRESENT",
        "BLOCKED", "🔴 DATA ISSUES — REVIEW REQUIRED",
        "⚪ UNKNOWN"
    )
```

---

### Last Pipeline Run
```dax
Last Updated = 
"Last refreshed: " & 
FORMAT(MAX('gold_run_audit'[run_timestamp]), "DD MMM YYYY, HH:MM")
```

---

## 5. Screening Measures

### Total Candidates
```dax
Total Candidates = COUNTROWS('gold_stock_candidates') + 0
```

---

### Total Universe Screened
```dax
Total Screened = COUNTROWS('gold_screen_current')
```

---

### Eligible Stock Count
```dax
Eligible Stocks = 
CALCULATE(
    COUNTROWS('gold_screen_current'),
    'gold_screen_current'[eligibility_status] = "ELIGIBLE"
)
```

---

### Eligibility Rate
```dax
Eligibility Rate Pct = 
DIVIDE([Eligible Stocks], [Total Screened], 0) * 100
```

---

### Average Candidate Score
```dax
Avg Candidate Score = AVERAGE('gold_stock_candidates'[composite_score])
```

---

### Growth Confirmed Count
```dax
Growth Confirmed = 
CALCULATE(
    COUNTROWS('gold_stock_candidates'),
    'gold_stock_candidates'[growth_confirmation_flag] = TRUE()
) + 0
```

---

### Graham Value Count
```dax
Graham Value Stocks = 
CALCULATE(
    COUNTROWS('gold_stock_candidates'),
    'gold_stock_candidates'[graham_value_flag] = TRUE()
) + 0
```

---

## 6. Dashboard Page Specifications

### Page 1: Command Center (Buy List)

| Position | Visual Type | Field / Measure |
|---|---|---|
| Top-left | Card | `[Market Regime Banner]` |
| Top-center | Card | `[Engine Health]` |
| Top-right | Card | `[Total Candidates]` |
| Top-far-right | Card | `[Avg Candidate Score]` |
| Center | Table | `gold_stock_candidates`: candidate_rank, symbol, company_name, sector, composite_score, research_status, roic, earnings_yield |
| Bottom-left | Donut Chart | Legend: `sector`, Values: Count of `symbol` |
| Bottom-right | Card | `[Last Updated]` |

**Conditional Formatting:**
- `composite_score` → Data bars (green gradient)
- `roic` and `earnings_yield` → Format as percentage, 1 decimal

---

### Page 2: Paper Trading Portfolio

| Position | Visual Type | Field / Measure |
|---|---|---|
| Top row (4 cards) | Card | `[Total Account Value]`, `[Current Cash]`, `[Unrealized PnL]`, `[Total Realized PnL]` |
| Second row (3 cards) | Card | `[Total Return Pct]`, `[Active Positions]`, `[Closed Trade Win Rate]` |
| Center-top | Table | `gold_paper_portfolio`: symbol, buy_date, quantity, buy_price + `gold_screen_current[close]` + `[Stock PnL]` |
| Center-bottom | Table | `gold_paper_ledger`: symbol, buy_date, sell_date, buy_price, sell_price, pnl_pct, realized_pnl, reason |

**Conditional Formatting:**
- `[Stock PnL]` → Font color: Green if > 0, Red if < 0
- `realized_pnl` → Background: Light green if > 0, Light red if < 0
- `reason` → Background: Red for `STOP_LOSS_TRIGGERED`, Yellow for `DROPPED_FROM_CANDIDATES`

---

### Page 3: Backtest Evidence

| Position | Visual Type | Field / Measure |
|---|---|---|
| Top | Table | `backtest_qtr_summary`: style, avg_screen_return_pct, stock_win_rate_vs_nifty_pct, sharpe_ratio, max_drawdown_pct |
| Center-left | Bar Chart | X: `style`, Y: `avg_screen_return_pct` |
| Center-right | Bar Chart | X: `style`, Y: `stock_win_rate_vs_nifty_pct` |
| Bottom | Line Chart | X: `screen_date`, Y: `avg_return`, Legend: `style` (from `backtest_qtr_screen`) |

---

## 7. Formatting Guidelines

### Number Formats

| Measure Type | Format | Example |
|---|---|---|
| Currency (₹) | `#,##0` with ₹ prefix | ₹10,45,230 |
| Percentage | `0.00%` | 7.65% |
| Score (0-100) | `0.0` | 74.2 |
| Ratio | `0.00` | 1.25 |

### Color Scheme

| Element | Color | Hex |
|---|---|---|
| Positive / Healthy | Green | `#107C10` |
| Negative / Alert | Red | `#D13438` |
| Warning / Review | Amber | `#F7A600` |
| Neutral / Info | Blue | `#0078D4` |
| Background | Light Gray | `#F3F2F1` |

---

## 8. Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `RELATED()` returns blank | No relationship between tables | Create Many-to-One relationship from portfolio to screen |
| Measures show blank | Table is empty (0 rows) | Normal when paper portfolio has no holdings yet |
| `Current Portfolio Value` is wrong | Duplicate symbols in `gold_screen_current` | Add `.dropDuplicates(["symbol"])` in NB06 |
| Cards show "(Blank)" | Measure returns null | Add `+ 0` to count measures |
| Dashboard not refreshing | Semantic model cached | Direct Lake should be instant; verify storage mode |
