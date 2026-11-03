# Forecast ledger

Published every Monday for the games releasing that week. Each row: forecast date, game, target,
horizon, baseline value, model value (if any), lower and upper quantiles, scoring date, outcome,
error. Rows are never edited; corrections are new rows that reference the original.

Scoring starts when the horizon closes. The dashboard reports hit rate, error in log scale and a
calibration plot (share of outcomes inside the 80 % interval, per target).
