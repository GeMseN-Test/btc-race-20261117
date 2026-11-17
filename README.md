# BTC Race Bot — make.fun bounty `btc-race-24h-20261117-0603`

[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/GeMseN-Test/btc-race-20261117/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/GeMseN-Test/btc-race-20261117/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

Entry for the [make.fun "Race a BTC trading bot live for 24 hours" bounty](https://make.fun/bounty/btc-race-24h-20261117-0603) ($250 in USDC).

A long-only trading bot for 1-minute cbBTC/USDC bars (Orca Whirlpool),
written for the make.fun "Race a BTC trading bot live for 24 hours"
competition. Python 3.13, standard library only, no network at runtime.

## Run

The entrypoint is `main.py` at the repository root. The grader runs it as:

```sh
python3 -P -s /bot/main.py
```

with `PYTHONHASHSEED=0`. It speaks `agent-fun/bars-replay/v1` on
stdin/stdout (newline-delimited JSON) via the vendored shim in
`vendor/makefun_bars.py` (a byte-for-byte copy of the kit's
`sdk/makefun_bars.py`).

## Strategy (see STRATEGY.md for the full write-up)

Two complementary entries, one shared position:

1. **Shallow-dip recovery** — buy when the price sits more than 1%
   below its 6-hour high *and* is recovering now (15-minute return
   above 0.1%, close above the last 15-minute high). Full size: on
   this dev window the day's range is only 2.5% and the real moves
   are V-shaped dips that retrace within hours.
2. **Volatility breakout** — buy when the close breaks above
   SMA(60) + 2.25·sd(60). Small size (25% of cash): breakouts in
   this choppy range are slightly negative on their own, so the leg
   is sized to participate (and to catch a trend day live) without
   dragging the account.

Exits per entry kind: trailing stop (1.2% recovery / 0.4% breakout),
take-profit (2% on the entry close, both legs), close back below the
breakout SMA (breakout leg), or 240 bars max hold.

## Dev-window results (local replay, exact fill model)

| | this bot | v2 bot carried over |
|---|---|---|
| Net PnL | **+20.83 USDC** on $1,000 start | -4.47 USDC (fails dev gate) |
| Trades | 10 (5 round trips) | 18 |
| Fees | 2.84 USDC | 1.80 USDC |
| Max drawdown | 1.16% (guardrail: 25%) | 0.67% |
| Runtime | ~1 s for 1,440 bars | ~0.3 s |

Recovery leg alone: +22.39 USDC, 6 trades. The breakout leg costs
~$1.56 on dev but keeps trend participation for the live race window.

The local grader mirrors the kit's fill model in exact integer
arithmetic (fee 4 bps, slippage 2 bps, constant-product impact
against each bar's open depth, 1-bar fill latency, final liquidation
at the last close). It was cross-checked against the kit's own
`grade-local --driver process` on the previous race: buy-and-hold
reproduces the official score exactly, and EMA-cross matches to
0.2 USDC over 63 trades.

## Robustness

The dev window is one quiet range day (+0.49% total, 2.49% range), so
the parameters were picked for a plateau, not a peak: every one-param
move off the chosen recovery params stays positive except a 0.4%
trailing stop (-11.58, whipsaw) and a 0.3% recovery bar (-0.31).
Pullback and momentum-burst legs were tested and rejected — they
never fire on this window. See STRATEGY.md.

## Determinism

All indicator math is integer fixed-point on the raw scaled prices:
no floats, no wall clock, no environment reads, no randomness. Two
runs produce byte-identical action logs (verified).

## Layout

```
main.py                  entrypoint
vendor/makefun_bars.py   kit SDK shim, byte-for-byte (excluded from authorship)
LICENSE                  MIT
README.md                this file
STRATEGY.md              strategy write-up
```
