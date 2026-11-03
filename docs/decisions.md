# Decisions

Format: date, decision, why, what it rules out. One entry per decision; never edit an old entry, add a new one.

## 2026-11-02 Domain: video games, not energy

Chosen because the data is a real event stream (viewers and players change by the minute), the
sources are companies with documented APIs rather than agencies that may stop publishing, and the
topic fits the rest of the portfolio. Plan B kept on record: EIA Open Data v2 and ENTSO-E.

## 2026-11-02 Kick as a second source from day one

Twitch holds roughly half of gaming hours watched in 2026; Kick is in the low teens but growing
fastest. Part of Kick's growth is IRL and gambling, so its per-game weight is below its share.

## 2026-11-02 No Dataflow

Pub/Sub to BigQuery subscription (Storage Write API) plus dbt incremental models cover the need at
cents per month. Dataflow would dominate the budget without changing any result.

## 2026-11-02 Forecast ledger is append-only and pre-registered

Each weekly forecast is a dated row in BigQuery plus a hash committed to this repo before outcomes
exist. Scoring happens only on the pre-registered metric and window.
