# Paper-Trading Simulation

This project simulates trades within Fabric. It does not connect to a broker,
place real orders, or manage a real investment account.

## State tables

- `gold_paper_portfolio` stores open simulated positions.
- `gold_paper_cash` stores the simulated cash balance.
- `gold_paper_ledger` records simulated closed trades.

## Assumptions to verify in the notebook

Document the actual implementation for:

- Starting simulated cash: [enter value]
- Entry timing and simulated fill price: [enter rule]
- Exit rules: [enter rules]
- Transaction costs and slippage: [included/excluded]
- Dividends and corporate actions: [included/excluded]
- Rebalancing frequency: [enter frequency]

These results are simulations, not actual execution results or evidence of future
performance.
