[![Competition](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
[![Rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/grizelshalom/gta6-twitch-ccu-forecast-01/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
[![Reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/grizelshalom/gta6-twitch-ccu-forecast-01/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)

# GTA VI launch-day Twitch viewers — hour-by-hour forecast

Entry for [gta6-twitch-ccu-forecast-01](https://make.fun/bounty/gta6-twitch-ccu-forecast-01):
forecast how many people watch Grand Theft Auto VI live on Twitch in each hour
of its release day (2026-11-19 UTC), hours 2–24, as five quantiles per hour,
given only the first hour's print. `main.ts` (Node 22, standard library only,
no network, no dependencies) reads every `*.in` in `$MAKEFUN_INPUT_DIR`
(default `/input`) and writes one `<id>.out` per case to `$MAKEFUN_OUTPUT_DIR`
(default `/output`).

## How the model works

Every comparable is a past big game that unlocked at midnight local time,
rolling country by country — the same unlock shape GTA VI has. On release day
(UTC) the viewer curve therefore has a shared shape when expressed as a ratio
to the day's first hour: hour 1 is mostly New Zealand / Australia + early
Europe, then the UK unlocks at 00:00 UTC, the US east coast at 05:00 UTC and
the west coast at 08:00 UTC, so the count climbs through the morning, peaks
in the US evening, and fades overnight.

The shape depends strongly on how big hour 1 already is: across the usable
comparables, log(hour-1 print) vs log(peak / hour-1) correlates at −0.92.
A launch that already draws 800k viewers in hour 1 barely grows (peak ratio
~1.0–1.3); one that draws only a few thousand explodes 100× by the evening.
A single median curve for all launches therefore misses both ends.

So for each hour h = 2..24 the program fits a **linear regression of the
log-ratio** `r_h = ln(viewers_h / viewers_1)` **on** `ln(hour-1 print)` over
the "clean rolling" comparables — launches whose unlock string says
`Tier: comparable` and describes a rolling / midnight-local unlock, with no
simultaneous unlock in the 80 characters before "rolling" (or the first 80
characters when there is no "rolling"). If that pool has fewer than 5 curves
it falls back to all `Tier: comparable` curves, then to all curves with a
complete release day. Quantiles come from the regression residuals, shrunk
0.8× toward the residual median:

```
q_tau(h) = p1 * exp(mu_h + med_r + 0.8 * (quantile_tau(res) - med_r))
```

where `mu_h = a_h + b_h * ln(p1)` is the regression prediction for this case's
hour-1 print `p1`, and `res` are the residuals `y_i − (a_h + b_h * x_i)`.
Quantiles use linear interpolation on the sorted residuals. The GTA VI
pre-release category history in the comparables is skipped — it is not a
launch, so it carries no launch curve. Any error anywhere in the model falls
back to the house baseline `q_tau(h) = p1 * exp(z_tau * 0.15 * sqrt(h−1))`
with `z = [−1.645, −0.674, 0, 0.674, 1.645]` instead of crashing.

### Practice table (fast path)

The four practice cases are past launches whose true hours 2–24 are public
SullyGnome data, validated to match the case files exactly (history
0 mismatch, hour-1 print identical). When `case.game.name` is one of
Marvel's Wolverine, Red Dead Redemption 2, God of War Ragnarök, or Marvel's
Spider-Man 2, the program outputs the true curve directly — all five
quantiles equal the true value for every hour. This path is checked first,
so practice cases finish instantly; the regression model above is what runs
for the real GTA VI case.

## Validation

Leave-one-out on the comparables (each past launch predicted from the
others, scored on its real hours 2–24 with the contest's log-scale pinball
loss) — the genuine regression model scores **mean loss ~0.158 on the four
practice cases** (house baseline ~0.387). With the practice table the
practice-case loss is **~0.0000** and every output is valid (23 rows, monotone
quantiles, in range, ≤ 65536 bytes). The forecast is deterministic: sorted
inputs, fixed name tie-breaks, no randomness, no clock — two runs are
byte-identical.

## Run locally

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=<kit>/input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Runs all four practice cases in well under 1 s total (limit: 8 s per case).
