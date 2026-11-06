# Data contract

Three event types flow from the ingestion adapters through Pub/Sub into BigQuery raw tables. The
machine-readable definitions live in `contracts/`:

| Event | Avro schema | BigQuery table schema | Producers | Cadence |
|---|---|---|---|---|
| `audience_sample` | `contracts/audience_sample.avsc` | `contracts/bigquery/audience_sample.json` | Twitch, Kick adapters | 15 min |
| `player_sample` | `contracts/player_sample.avsc` | `contracts/bigquery/player_sample.json` | Steam players adapter | 30 min |
| `review_snapshot` | `contracts/review_snapshot.avsc` | `contracts/bigquery/review_snapshot.json` | Steam reviews adapter, one-off backfill | hourly / daily |

The Avro file is the source of truth. The BigQuery JSON is derived from it (same fields plus the
four Pub/Sub metadata columns the subscription writes) and a test keeps them in sync. Terraform
reads both: the Avro file becomes the topic schema, the JSON becomes the table definition.

## Conventions shared by all events

- Every timestamp is UTC, Avro `timestamp-micros`, BigQuery `TIMESTAMP`.
- `sampled_at` (or `snapshot_at`) is the start of the sampling bucket, computed by flooring the
  current time to the cadence. `observed_at` is when the API answered. Analyses use the bucket;
  `observed_at` exists to measure latency and to debug.
- `run_id` identifies the job execution. A retried execution inside the same bucket may publish
  the same rows twice; raw tables keep both and dbt staging deduplicates on the event's key.
- `sample_tier` records how a row entered the sample. Rows with tier `top` are subject to a
  viewer floor (Twitch: 10 viewers) or a rank cutoff (Steam: top 100); rows with tier
  `watchlist` have no floor. Any metric over small games must filter or weight by tier.
- Enumerations are plain strings, not Avro enums, because Pub/Sub maps enums to integers in
  BigQuery.
- Adapters emit platform-native ids only. The mapping to the canonical IGDB id is a dbt concern
  (`dim_game`).
- Breaking changes bump `schema_version` and create a new topic, table and schema file with a
  `_v2` suffix. Additive nullable fields do not bump the version.

## Raw table layout

Each event type has one topic, one BigQuery subscription with `use_topic_schema` and
`write_metadata`, one raw table, and one dead-letter topic with its own BigQuery subscription.
Raw tables are partitioned by day on the bucket timestamp, clustered on (`source`,
`platform_game_id`) or (`source`, `appid`), and require a partition filter.

## Deduplication keys

| Event | Key |
|---|---|
| `audience_sample` | `source`, `channel_id`, `sampled_at` |
| `player_sample` | `appid`, `sampled_at` |
| `review_snapshot` | `appid`, `snapshot_at`, `source` |

## Known gaps, by design

- Stream titles and tags are not collected in v1.
- Twitch streams under 10 viewers are observed only for watchlist games.
- Kick exposes no stream id; a stream is identified by (`channel_id`, `stream_started_at`) and
  its end is inferred in dbt when it is absent from two consecutive samples.
- Concurrent-player history before the project started cannot be reconstructed; review history
  can (`source = steam_backfill`).
