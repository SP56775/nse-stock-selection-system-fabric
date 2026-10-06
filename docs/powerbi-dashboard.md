# Power BI Dashboard

The report uses a Direct Lake semantic model connected to the Fabric Lakehouse.

## Pages

1. **Candidate screen:** ranked candidates, factor scores, and market regime.
2. **Paper portfolio:** simulated cash, open positions, and unrealized P&L.
3. **Trade ledger and validation:** closed simulated trades and backtest summaries.
4. **Data quality:** pipeline audit and data-quality checks.

The portfolio page should use the `gold_paper_*` tables, not the legacy
`gold_portfolio_actions` table, when displaying the paper-trading simulation.
