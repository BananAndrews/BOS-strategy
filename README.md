# BOS-strategy

A TradingView indicator that produces buy/sell signals from a break of structure (BOS)
confirmed by two directional candles.

Source: [`BOS-signals.pine`](BOS-signals.pine) (Pine Script v6).
The spec was dictated in a screen recording; the transcript is in
[`docs/narration-transcript-ru.txt`](docs/narration-transcript-ru.txt).

## Candle definition

A **long candle** is a green candle that closes **above the previous red candle's high**.
A **short candle** is a red candle that closes **below the previous green candle's low**.

```pine
isLongCandle  = close[1] < open[1] and close > open and close > high[1]
isShortCandle = close[1] > open[1] and close < open and close < low[1]
```

This matches `retest_of_retest_indicator.pine`, which states the same pattern one bar back
as `close[1] > high[2]`. Here it is evaluated on the candle itself, so it is recognised as
soon as that candle closes rather than one bar later.

## The pattern

Long setup; short is the exact mirror.

1. **Candle 1.** A long candle forms.
2. **Local low.** Search back `lookback` bars for the lowest low. It must sit **below
   candle 1's body**, and at least `minStruct` bars back.
3. **BOS.** The **highest high strictly between** that local low and candle 1. If no bars
   sit between them there is no structure and the setup is dropped. A BOS also only counts
   if price has not already cleared it when candle 1 forms.
4. **Alert.** A yellow triangle marks candle 1 — a setup is armed.
5. **Candle 2.** A second long candle, at least `minGap` bars later, that satisfies:
   - the **peak strictly between** the two candles exceeds the BOS by **delta 3**;
   - candle 2 **opened and closed above the BOS** (optionally by a percentage margin);
   - optionally (`Require pullback off the peak`), that peak also stands above candle 2's
     own high by `Peak clearance above candle 2 (%)`, so candle 2 is not itself the
     extreme and the pullback is a real one rather than a single tick.
6. **Entry.** A `BUY` line at the **BOS level** — a retest of the broken structure.
7. **Stop.** A `STOP` line below candle 1's low, minus the stop buffer.
8. **Exit.** No fixed take profit. The position closes on the **opposite signal**, marked ✗.

A setup is dropped if it is not confirmed within `maxWait` bars, or if **price trades back
beyond candle 1** in the opposite direction — for a long, any low below candle 1's low means
the structure that framed the setup is gone. Opposite *candles* between candle 1 and candle 2
are **ignored**; only price breaking candle 1's extreme resets the setup.

**Setups run concurrently.** Every qualifying candle arms its own, in either direction, and
each resolves independently — an open setup never suppresses a later one.

## The three deltas

Two are horizontal (time) and set in **bars**; the third is vertical (price) and cannot be
expressed in bars.

| # | Delta | Unit | Default |
|---|---|---|---|
| 1 | Lookback before candle 1, to find the local high/low and the BOS | **bars** | 20 |
| 2 | Distance between candle 1 and candle 2 | **bars** | min 3, max 50 |
| 3 | Distance between the BOS and the peak between the candles | **price** | 0.15% |

Delta 3 takes a unit selector: **Percent**, **ATR multiple**, **Ticks**, or **Price**.
Percent travels between symbols; ATR multiple also adapts to current volatility.
Rough equivalence on ADA-like 5m data (median ATR ≈ 0.23% of price):
`0.15%` ≈ `0.6 ATR` ≈ `4 ticks`.

## Inputs

| Group | Input | Default | Meaning |
|---|---|---|---|
| Delta 1 | Lookback before candle 1 | 20 | Bars searched left for the local extreme |
| Delta 1 | Min bars between extreme and candle 1 | 2 | Structure requirement |
| Delta 2 | Min bars between candle 1 and candle 2 | 3 | Minimum 2, so a peak can exist between them |
| Delta 2 | Max bars to wait for confirmation | 50 | Setup expiry |
| Delta 2 | Require pullback off the peak | off | Peak must top candle 2's high |
| Delta 2 | Peak clearance above candle 2 (%) | 0.0 | How far it must top it by; 0 means any margin counts |
| Delta 3 | Measure break distance in | Percent | Percent / ATR multiple / Ticks / Price |
| Delta 3 | BOS break distance | 0.15 | How far past the BOS the peak must reach |
| Delta 3 | Candle 2 body clearance beyond BOS (%) | 0.0 | Margin candle 2's body must clear the BOS by. **Always a percentage**, independent of the unit above |
| Delta 3 | ATR length | 14 | Only used for ATR-multiple units |
| Stop | Measure stop buffer in | Percent | Same four units |
| Stop | Stop buffer | 0.05 | Extra room beyond candle 1's extreme |
| Limits | Max simultaneous setups per direction | 20 | Guard rail; rarely binds |
| Limits | One signal per direction | on | Suppress repeats until the direction flips |
| Debug | Explain rejected setups | off | Label why each candidate failed |

## Debugging a missing signal

Turn on **Debug → Explain rejected setups**. Every candle that could have armed or
confirmed but did not is labelled with the reason:

| Colour | Stage | Reasons |
|---|---|---|
| Dark red | never armed | `no local low below body`, `local low too close`, `no BOS between`, `BOS already cleared` |
| Orange | candle 2 rejected | `no peak between`, `peak below candle 2`, `break < delta 3`, `candle 2 body not above BOS` |
| Blue | suppressed | `already long` / `already short` |
| Grey | setup died | `expired`, `broke candle 1 low` / `broke candle 1 high` |

Green/red circles mark every detected long/short candle regardless of setup — the quickest
way to check the candle definition against the chart.

## Filter strength

Measured over 5,000 synthetic bars, so treat these as relative weights rather than
forecasts:

| Rule | Effect on signal count |
|---|---|
| Concurrent setups (vs one at a time) | 50 → 61 |
| Candle 2 body must clear the BOS | 60 → 56 |
| One signal per direction | 125 → 43, strictly alternating |
| Price breaking back past candle 1 resets | 43 → 28 |
| Pullback, clearance 0.00% / 0.10% / 0.50% | 28 → 24 / 18 / 7 |

The two strongest are `One signal per direction` and the candle-1 break reset — the latter
is wick-based with no tolerance, so a single tick past candle 1's extreme kills the setup
and the median setup now lives about one bar.

The pullback clearance is the finest dial: at 0% it barely bites, and it becomes the
dominant filter past about 0.2%. Note it overlaps the candle-2 body rule, since both stop
candle 2 confirming in the wrong place.

## Installing

Pine Editor → paste `BOS-signals.pine` → *Add to chart*. Four alert conditions are
exposed: `BOS Buy`, `BOS Sell`, `BOS Long setup`, `BOS Short setup`. The two signal alerts
respect `One signal per direction`, so they fire only on an actual direction change.

## Open question

**Default delta 3 = 0.15%** is a starting point for ADA/DOGE on 5m, not a tuned value. The
brief describes the right calibration as low enough sensitivity to skip weak impulses, and
expects it to be tuned per symbol.
