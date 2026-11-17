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

So for each hour the program looks at the **12 past launches whose hour-1
print is closest in log space** to this case's hour-1 print, and takes the
quantiles of their log-ratios `ln(viewers_h / viewers_1)`, widened 1.3× around
the median:

```
q_tau(h) = p1 * exp(med_h + 1.3 * (quantile_tau(r_h) - med_h))
```

The 1.3× widening is calibrated: with 12 neighbours the raw empirical 5% /
95% quantiles are too narrow (90% coverage ~0.81 in leave-one-out); widened,
coverage is ~0.89 while the pinball loss is unchanged. The GTA VI
pre-release category history in the comparables is skipped — it is not a
launch, so it carries no launch curve. The forecast is deterministic: sorted
inputs, fixed tie-breaks, no randomness, so two runs are byte-identical.

## Validation (leave-one-out on the comparables)

Each past launch predicted from the others, scored on its real hours 2–24
with the contest's log-scale pinball loss:

| subset | this model | house baseline |
|---|---|---|
| all launches (n=34) | **0.439** | 0.761 |
| big launches, hour-1 ≥ 20k (n=27) | **0.263** | 0.321 |
| mega launches, hour-1 ≥ 100k (n=16) | **0.221** | 0.261 |

90% coverage ~0.89 (big launches), typical miss ~48% on big launches
(baseline: ~63%). The house baseline scores ~0.387 on the practice cases;
this entry's practice-case outputs were checked valid (23 rows, monotone
quantiles, in range) but the practice answers are not in the kit, so no
practice loss can be computed locally.

## Run locally

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=<kit>/input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Runs all four practice cases in ~0.4 s total (limit: 8 s per case).
