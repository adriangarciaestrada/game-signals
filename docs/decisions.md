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

## 2026-11-05 Secrets: Secret Manager only, identity federation everywhere

Google Secret Manager is the single store for application secrets (Twitch, Kick and Steam API
credentials). Cloud Run jobs read them through their attached service account; local development
reads the same versions with `gcloud secrets versions access` under Application Default
Credentials. GCP access never uses files: ADC locally, the job's service account on Cloud Run,
Workload Identity Federation on GitHub Actions. Terraform creates the secret containers and IAM
bindings but never the secret values, because anything passed through a Terraform variable is
stored in plain text in the state file.

Why: the project needs two or three long-lived client secrets and nothing else; Secret Manager is
already part of the project, has a permanent free tier that covers this usage, logs every access,
and keeps one source of truth for local and production. A separate password manager would still
need a bootstrap token inside GCP, so it adds a platform without adding security.

Rules out: service-account JSON keys anywhere; `.env` files with real values; a second secret
platform (Bitwarden, Doppler, Infisical, sops); secret values in Terraform variables or state.
Console logins for the third-party developer portals are personal credentials and stay in a
personal password manager outside the project.

## 2026-11-05 Cadence: 15 minutes for Twitch and Kick, 30 minutes for Steam players

Why: 30 minutes undercounts short streams, which is most of the signal for small games, the
project's core question. 5 minutes would put two jobs past the Cloud Run free tier (about 2 USD
per job per month) and Secret Manager past its free reads. 15 minutes keeps both at zero and
gives 288 samples in the first 72 hours of a launch. Steam concurrent players move slowly and are
one call per game, so 30 minutes is enough there. Reviews for games in their first 72 hours are
polled hourly because targets reference review counts at 24, 48 and 72 hours.

Rules out: a uniform cadence across sources; 5-minute polling as the default. An adaptive 5-minute
window for tracked launches remains open for phase 1b, to be decided with data from a short
5-minute experiment subsampled to 15 and 30 minutes.
