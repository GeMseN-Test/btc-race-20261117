# STRATEGY.md — BTC race bot for btc-race-24h-20261117-0603

## The dev window

The frozen 24h dev window (dataset `fe7f72eb…`, 1,440 one-minute bars)
is a quiet range day: BTC opens $64,461, closes $64,776 (+0.49%),
low $63,874, high $65,467 (2.49% range). Hourly returns alternate
+0.4 / -0.6 / +1.2 / -0.5 — chop, no trend. Buy-and-hold makes
+1.07 USDC; almost every active baseline loses (EMA-cross -106,
RSI-Bollinger -53, Donchian -8.3, momentum burst -4.5). The only
real moves are two V-shaped dips: a -2.06% 60-minute slide around
bar 514 that retraces within ~2 hours, and a smaller dip near
bar 1126.

## What did not work

The previous race's v2 bot (volatility breakout SMA(90)+2.00σ at
25% size + deep capitulation recovery at 4% drop, 0.3% recovery bar)
scores **-4.47 USDC, 18 trades** on this window — it fails the dev
gate (needs > $0). Staged sweeps (324 vb combos, 216 rec combos,
124 pullback/burst combos) could not lift it above -1.44: the
breakout leg churns on false breakouts, and the deep-recovery leg
never fires once — there is no 4% washout on this window at all.

## The winning shape

A shallow-dip recovery leg, tuned to this window's actual moves:

- Buy when close is > **1%** below the 6-hour high **and** the
  15-minute return is above **0.1%** **and** the close is above the
  last 15-minute high. Full size.
- Exits: **2%** take-profit, **1.2%** trailing stop, 240 bars max hold.

Recovery leg alone: **+22.39 USDC, 6 trades (3 round trips)** —
+17.27 on the bar-393 dip (bought the slide, sold the retrace),
+2.48 and +2.64 on the two smaller dips. Fees 2.44, max DD 1.18%.

The breakout leg is kept, strict, at 25% size (SMA(60) + 2.25σ,
2% TP, 0.4% trail) because the race window is a different day: if
it trends, breakouts pay. On dev it costs ~$1.56 (combined
**+20.83 USDC, 10 trades, fees 2.84, max DD 1.16%**), a fair price
for trend participation.

## Robustness (one-param moves off the chosen recovery params)

| param | values tried | PnL range |
|---|---|---|
| drop | 75 / 100 / 125 / 150 / 200 bps | +18.98 / +22.39 / +22.39 / +2.48 / no trades |
| recovery bar (15m) | 5 / 10 / 15 / 20 / 30 bps | +22.39 / +22.39 / +22.31 / +0.32 / -0.31 |
| take-profit | 1 / 1.5 / 2 / 3 / 4% | +18.86 / +18.20 / +22.39 / +10.58 / +10.58 |
| trailing stop | 0.4 / 0.8 / 1.2 / 2 / 3% | -11.58 / +22.39 / +22.39 / +22.39 / +22.39 |
| lookback | 60 / 120 / 180 / 360 / 720 bars | +2.48 / +22.39 / +22.39 / +22.39 / +2.63 |
| max hold | 60 / 120 / 240 / 480 / none | -0.98 / +16.30 / +22.39 / +20.21 / +17.05 |

Plateau, not a peak: drop 75–125, recovery bar 5–15, TP 1–4%,
trail 0.8–3%, lookback 120–360 all stay positive. The two sharp
edges are a 0.4% trail (whipsaw: 10 trades, -11.58) and a 0.3%
recovery bar (the dip confirmation arrives too late, -0.31) —
both avoided by a wide margin.

## Rejected legs

- **Uptrend pullback** (SMA(30) > SMA(120), dip below SMA(30),
  RSI > 45): never fires on this window.
- **Momentum burst** (15m return > 60 bps, 5m return > 15 bps):
  never fires on this window.
- **Deep recovery** (drop 3–4%): never fires on this window.

## Costs and guardrails

A full-size round trip pays ~26–40 bps (4 bps fee each way,
2 bps modelled slippage each way, plus constant-product impact
against ~$0.4–0.9M of pool depth). Entries only fire on moves
worth several times that. Max drawdown 1.16% (guardrail 25%),
at least 1 trade (10 trades), deterministic integer math.
