# Fabric Pipeline

## Pipeline
`PL_Stock_Selection_Weekly`

## Purpose
Refresh the stock-selection data and update the Fabric paper-trading simulation.
The pipeline does not connect to a broker or place real orders.

## Schedule
- Frequency: Weekly
- Day: Tuesday
- Time: 06:00
- Time zone: Asia/Kolkata (UTC+05:30)

Recheck or recreate the schedule in Fabric after deploying the pipeline. The
schedule may be workspace/service configuration rather than part of the
source-controlled pipeline definition.

## Activities

Run each activity only after the preceding activity succeeds:

1. `NB01_Universe_Ingestion`
2. `NB02_Price_Bronze_Ingestion`
3. `NB03_Fundamental_Bronze_Ingestion`
4. `NB04_Silver_Price_Features`
5. `NB05_Silver_Fundamental_Features`
6. `NB06_Scoring_and_Gold_Output`
7. `NB07_Data_Quality_Checks`
8. `NB08_Paper_Trader`

The paper trader must not run if the data-quality check is blocked.

## One-time setup

The paper-account initialization notebook is run manually once. It is not part
of the recurring schedule because it may reset simulated cash, positions, or
the trade ledger.

## Outputs

- `gold_stock_candidates`
- `gold_paper_portfolio`
- `gold_paper_cash`
- `gold_paper_ledger`
- `gold_data_quality_results`
- `gold_run_audit`

## Deployment notes

After importing or connecting the project in another workspace, verify the
workspace, Lakehouse, notebook references, and schedule in Fabric. Do not commit
credentials, personal holdings, access tokens, run logs, or generated Delta
data.
