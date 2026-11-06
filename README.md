# game-signals

Streaming data platform on GCP for the video-game industry: live audience on Twitch and Kick,
concurrent players and reviews on Steam, metadata from IGDB. Built to answer questions a new
game developer actually has (what do launches like mine look like, what early signals matter)
and to publish forecasts that are scored later against reality.

Status: planning. Started 2026-11-03. Open design questions and their proposals: `docs/open-decisions.md`.

## Goals

1. A low-cost streaming pipeline end to end: scheduled Cloud Run jobs -> Pub/Sub -> BigQuery
   (Storage Write API subscription) -> dbt -> Looker Studio. No Dataflow. Budget: 10 USD/month.
2. Analysis notes with a fixed structure (question, data, method, result with uncertainty,
   decision it informs) every two to three weeks.
3. A public forecast ledger: predictions with intervals published before the fact, scored after.

## Sources

| Source | Endpoint | Cadence | Notes |
|---|---|---|---|
| Twitch Helix | `GET /streams` | 15 min | app credentials; uses IGDB ids |
| Kick public API | `GET /livestreams` | 15 min | app token (60 days); no stream id, key is (channel, started_at) |
| Steam Web API | `GetMostPlayedGames`, `GetNumberOfCurrentPlayers`, `/appreviews/{appid}?json=1`, `GetAppList`, appdetails | 30 min / hourly / daily | `GetAppList` needs a key; reviews, players and appdetails do not |
| IGDB | games, external_games | daily | bridge table for categories across platforms |

YouTube Gaming is out of scope for v1 (quota and inconsistent per-game classification).

## Layout

```
ingestion/   Python adapters, one per source, emitting a common "audience sample" schema to Pub/Sub
dbt/         staging -> intermediate -> marts; incremental, partitioned by date, clustered by game
terraform/   project, Pub/Sub topics and BigQuery subscriptions, datasets, Cloud Run jobs, Scheduler, budget alert
analysis/    notebooks and notes; each note follows the five-part structure above
forecasts/   weekly ledger (pre-registered targets, metrics and windows) and scoring code
docs/        decisions (ADRs), data contracts, cost log
```

## Phases

- Phase 1 (weeks 1-2): Twitch feed streaming into BigQuery with one live chart.
- Phase 1b: Kick and Steam adapters, IGDB bridge, dbt marts, Looker Studio dashboard, CI with `dbt build`.
- Phase 2 (from week 6): forecast ledger moves from comparables baseline to models
  (BigQuery ML first, then LightGBM quantile regression in a Cloud Run job).

## Forecast targets (in order of difficulty)

1. 30-day peak players and review count from the first 72 hours.
2. Pre-launch: reaches 10 / 100 / 1,000 reviews within 90 days.
3. Half-life of the launch peak by genre and studio size.
4. Breakouts: games outside the top 200 that enter it within 7 days.
5. Nowcast: Twitch/Kick audience -> next-day Steam players, by genre.

Credibility rules: always against a baseline (median of comparables, persistence); quantile
intervals with a calibration plot; time-based splits only; metric and window pre-registered;
boring models first. "Does not beat the baseline" is a reportable result.

## Variables and known traps

- Pre-launch features: full tag set (not just genre), price, early access, prior demo, languages,
  Steam Deck status, release day/month, releases the same week, studio history (prior games, their
  peaks and reviews), publisher presence.
- Early signals: peak players day 1-3, reviews at 24/48/72 h, review velocity, hours watched and
  distinct channels on Twitch and Kick, viewer concentration, viewers-to-players ratio.
- Targets are modeled in log scale (heavy-tailed outcomes). Intervals come from quantile regression.
- Survivorship: every Steam release of the week is recorded, including the ones that reach zero.
- Leakage: a forecast uses only data that existed at forecast time.
- Not available publicly: wishlists and pre-launch marketing reach. Their absence bounds what any
  pre-launch forecast can do, and quantifying that bound is itself a finding.
