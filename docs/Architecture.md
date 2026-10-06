# Architecture

This project is an educational stock-screening and paper-trading prototype built
with Microsoft Fabric.

## Data flow

1. Universe, price, and fundamentals are ingested into Bronze tables.
2. Silver notebooks clean data and calculate price and fundamental features.
3. The scoring notebook creates a ranked candidate list in Gold.
4. Data-quality checks audit the outputs.
5. The paper-trading notebook updates simulated positions, cash, and closed trades.
6. Power BI reads the Lakehouse tables through a Direct Lake semantic model.

## Paper-trading tables

- `gold_paper_portfolio`: simulated open positions
- `gold_paper_cash`: simulated cash balance
- `gold_paper_ledger`: simulated closed trades

The paper-account initialization notebook is a one-time setup step and should not
be scheduled if it resets these tables.
