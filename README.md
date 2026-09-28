# Causal discovery in markets with hidden drivers

Most of what moves a market is never observed directly: order flow,
positioning, funding stress, news. Prices are also sampled at intervals that
can be too coarse to see which asset moves first. This project is an
end-to-end pipeline that takes observational time series and reports which
causal statements the data support, which ones an unmeasured driver could
explain, and which ones the sampling interval cannot decide. It reports an
undecided link as undecided instead of forcing a direction, and it says what
extra information (a finer clock, a scheduled event, a measured driver) would
decide it.

The pipeline is a Python package of about 11,000 lines with 70 automated
tests against closed-form answers and exact oracles. It runs on one desktop
for everything reported here except the heaviest nonparametric checks, which
run on a SLURM cluster. The implementation is private (see
[Code availability](#code-availability)).

## Highlights

- **Bar data cannot say which liquid asset moves first.** Even at one minute,
  18 of 21 pairs of seven liquid funds adjust to each other within the bar,
  and every pair inside the equity, credit and volatility complex does. [More](results/cross-asset.md#bar-data-rarely-reveal-who-drives-whom)
- **Most co-movement comes from a few hidden drivers.** Measuring stand-ins
  for market-wide drivers from 22 other funds removes 10 of the 13 robust
  daily links between twelve funds.
  [More](results/cross-asset.md#the-daily-network-and-what-hidden-drivers-explain)
- **Relations change at breaks the data date on their own.** Since February
  2021 long Treasuries no longer hedge equities.
  [More](results/cross-asset.md#three-hidden-drivers-and-the-2021-break)
- **A company's implied volatility follows its stock and never leads its
  return.** Fifteen years of daily data, eleven underlyings.
  [More](results/single-companies.md#implied-volatility-follows-the-stock-and-leads-no-return)
- **Scheduled events settle directions that timing cannot.** On earnings
  days, in all five companies studied, neither the company's return nor its
  implied volatility moves the market or the VIX on the same day.
  [More](results/single-companies.md#scheduled-events-settle-same-day-directions)
- **A forecasting "signal" that was the earnings calendar.** Heavy volume
  followed by falling implied volatility disappears once earnings sessions
  are removed.
  [More](results/single-companies.md#the-volume-lead-on-implied-volatility-is-the-earnings-calendar)
- **Machine learning adds magnitude, not direction.** A gradient-boosted
  re-check confirms the company graphs and finds 160 links of size (volume
  and range responding to large moves), and nothing of substance about the
  sign of the next day's return (two flags, both with correlations below
  0.1).
  [More](results/single-companies.md#what-a-machine-learning-re-check-adds)
- **Validated before use.** Every stage is scored on simulated systems with
  known answers; the checks found and fixed two failure modes of the search
  and one of the re-check. [More](results/validation.md)

## What the pipeline answers

For any multivariate or gridded time series, in order:

1. Is the sampling interval fine enough to tell direction, pair by pair?
2. Where are the regime changes, and which variables change their mechanism
   across them?
3. Which links exist, which have a definite direction, and which could come
   from a hidden common driver?
4. How many hidden drivers are there, can they be recovered, and can
   stand-ins for them be measured from series outside the graph?
5. Can scheduled events (earnings releases, central-bank meetings) settle
   directions that timing leaves open?
6. Which links are nonlinear, and in which regime?
7. How strong is each directed coupling, in information units per unit time?
8. Which links forecast out of sample, and does the forecast pay for trading
   costs?

Every answer carries its own reliability check: false-discovery control on
links, stability under subsampling and aggregation, bootstrap intervals, a
re-check of every pair with a second, nonparametric test, and replication in
both halves of the record.

## Results

| Page | What it covers |
|---|---|
| [Across asset classes](results/cross-asset.md) | ETFs for equities, rates, credit, commodities, currencies, volatility and crypto; 1-minute, 5-minute and daily bars, 2007 to 2026 |
| [Single companies and options](results/single-companies.md) | Apple, Amazon, Alphabet, Goldman Sachs, IBM and six funds with their implied volatility, 2011 to 2026; option contracts for two years; scheduled events; the machine-learning re-check |
| [Validation](results/validation.md) | Every stage against known answers, including the corrections found at scale |

![daily with proxies](figures/12_daily_graph_with_driver_proxies.png)

*Daily network of twelve funds (a) on its own and (b) with five measured
stand-ins for market-wide drivers. Circles are ends the data leave
undetermined. Most links in (a) disappear once the drivers are measured.*

## What holds up, in trading terms

- Direction between liquid ETFs cannot be read from daily, 5-minute or even
  1-minute bars; the sampling diagnostic says so before any search is run.
  A claimed lead of one liquid fund over another should be treated as an
  artefact until it is shown on quote data and after costs.
- The common structure is a few hidden drivers, and the regime breaks can be
  dated from the data alone. Since the 2021 break, long Treasuries no longer
  hedge equities, so hedge ratios should be estimated within the current
  regime.
- Across single stocks, abnormal volume is a stable forecaster of next-day
  volatility and carries no information about direction.
- A company's implied volatility responds to its stock and to the VIX on the
  same day and leads nothing about the next day's return. What forecasts is
  activity: more trading after a rise in implied volatility, a wider range
  after a fall in price. These are inputs for sizing and volatility
  positions, not for direction.
- Before trading a pattern around implied volatility, check it against the
  earnings calendar.
- No cross-asset lead at 5 minutes or longer survives trading costs.

## Engineering

- Package: estimators, statistical tests, structure search, diagnostics,
  event analysis, machine-learning tests, plotting, a command-line tool and a
  configuration system. About 6,500 lines in the library and 4,800 in
  applications and scripts.
- Testing: 70 tests pinning each estimator and diagnostic to a closed form or
  an exact oracle, including soundness checks on 40 random graphs with hidden
  nodes.
- Speed: a machine-learning re-check at about a second per test, against
  minutes for the nonparametric test it complements; the twelve
  single-company records re-check in nine minutes on one desktop.
- High-performance computing: SLURM array jobs on 96-core AMD nodes (Stony
  Brook SeaWulf), with pinned environments and measured run-time and memory
  budgets for every job. Search costs are measured before a job is sized.
- Profiling found and fixed problems before they cost cluster time: a numpy
  2.5 incompatibility in a dependency that crashed every nonparametric
  search, a quadrature that needed more than 18 GB of memory (now 0.6 GB,
  with identical results), and a search setting whose cost grew roughly with
  the square of the sample size.
- Output: journal-style figures, checked automatically for font size, tick
  direction and legend overlap.

Using it on any dataset is one command:

```bash
ecausal-run --data prices.csv --time-col date --dt 1 --time-unit day --out results/prices
```

## Data sources

Polygon.io (daily, 1-minute and option-contract bars), Yahoo Finance (daily
history from 2007 and earnings dates), the Cboe Options Exchange (daily
implied-volatility indices), the Federal Reserve (FOMC meeting calendars),
FRED (rates) and a local store of US stock prices. None of the data are
redistributed here; the figures and tables show derived results only.

## Code availability

The implementation, validation suite and cluster scripts are in a private
repository. Hiring teams can request read access through my GitHub profile.

## Copyright

Copyright (c) 2026 Yaohui Shu. All rights reserved. See [LICENSE](LICENSE).
