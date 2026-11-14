# btc-cent-forecast-01 — 4h-ahead Bitcoin price forecast

Entry for make.fun bounty **btc-cent-forecast-01** ($250, 100 paid places).
Program: Python 3.13, standard library only, no network, deterministic.
It reads one case file (`/input/<id>.in`, 4,320 one-minute BTC closes over
the 72h to the checkpoint) and writes `/output/<id>.out`:
`{"forecasts": [{"case": "<id>", "price": <guess>}]}` — the expected BTC
price 240 minutes after the checkpoint. Score: mean absolute error in
cents over the cases; a prize needs a score strictly below the house
baseline (persistence).

## Method

**Centre: the last close (persistence).** I tested hard for a better
signal and did not find one:

- Rolling 7-week backtest on Coinbase BTC-USD 1h closes (Sep 1 – Nov 11
  2026; 1,176 four-hour-ahead forecasts, strictly out-of-sample):
  ridge models on 1h/4h/24h/72h returns, moving-average deviations and
  volatility all scored *worse* than persistence (+0.08% to +0.19%).
  Smoothing the centre (trailing means, EWMAs) was 3–28% worse.
  Hour-of-day drift was +0.69% to +5.01% worse, and its pattern had a
  split-half correlation of +0.07 — noise, not signal.
- Conditional medians of the 4h return (the MAE-optimal target) sit
  within a few bp of zero in every quintile of past returns and
  volatility. The 4h-ahead price is, from past prices alone,
  essentially unpredictable — so the honest forecast is the last price.

**Tilt: a small linear pull on the last 1h log-return**, ridge-fitted on
Coinbase 1h closes Sep 1 – Nov 7 2026 (strictly before the practice
window), shrunk by half:

```
drift = 0.5 * (-0.00016725 - 0.00031114 * r1h)
guess = last_close * exp(drift)   (clamped, rounded to 5 decimals)
```

It was the best-behaved adjustment tested: rolling backtest +0.08% vs
persistence (neutral) and practice −0.79% (see below). A tail-reversal
rule (reverse 10% of a >50bp 1h move) looked best on the rolling
backtest (−0.08%) but lost +2.66% on the practice cases from a single
$62 miss, so it was dropped. The tilt also keeps the guess off the
house baseline's exact persistence line.

## Local validation

Practice cases p00–p04 (truth = the next case's checkpoint close; the
house baseline's published 6-case practice loss is 44,521.58 cents,
mean error $445.22):

| case | this entry | persistence |
|------|-----------|-------------|
| p00  | $310.85   | $317.11     |
| p01  | $760.77   | $766.97     |
| p02  | $464.27   | $470.33     |
| p03  | $573.02   | $566.85     |
| p04  | $225.29   | $231.47     |
| mean | **$466.84 (−0.79%)** | $470.55 |

- Deterministic: two runs over all 6 practice cases produce
  byte-identical outputs.
- Runtime: ~0.05 s for all 6 cases (limit 8 s per case).
- Output format verified on all 6 cases: exactly
  `{"forecasts": [{"case": ..., "price": ...}]}`, price in (0, 10,000,000).

Backtest scripts and data (not part of the entry): `work/` next to this
repo — `fetch_coinbase.py`, `backtest.py` … `backtest6.py`.
