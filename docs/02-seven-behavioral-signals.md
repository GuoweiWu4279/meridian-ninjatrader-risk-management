# Seven Behavioral Signal Dimensions (D1 to D7)

Meridian's Psychological Stability Index is a composite of seven behavioral signal dimensions, each measuring a distinct pattern of behavioral deterioration that appears in the order stream during a live trading session. The signals are labeled D1 through D7.

Each signal is computed from order-level events: fills, order modifications, position changes, and session timing. Each is measured relative to the trader's own adaptive baseline, not against a fixed universal threshold. The combined output of all seven dimensions, weighted according to the trader's configuration, produces the composite PSI score.

---

## D1: Revenge Entry

**What it measures:** The tendency to re-enter a position rapidly after a stop-out, typically in the same direction and at elevated size, driven by the impulse to recover the loss rather than by a new technical assessment.

**How it fires:** D1 activates when a new entry occurs within an abnormally short interval after a losing exit, particularly when the new entry size is elevated relative to the trader's baseline. The direction weighting is asymmetric: a reversal entry carries less weight than a same-direction re-entry, because reversals can represent legitimate technical responses to price action while same-direction rapid re-entries are more consistently associated with emotional loss-chasing behavior.

**Behavioral finance context:** The loss aversion asymmetry documented by Kahneman and Tversky (1979) establishes that the psychological pain of a loss is roughly twice the magnitude of equivalent gain pleasure. Under this asymmetry, traders experience significant impulsive pressure to immediately recover losses. Experimental evidence in the trading context (Odean 1998, Barber and Odean 2000) shows that this pressure reliably produces worse execution outcomes when acted on. Revenge Entry is the behavioral signature of loss aversion asymmetry expressing itself in the order stream.

---

## D2: Stop Manipulation

**What it measures:** The modification of a stop-loss order in the adverse direction after the price has moved against the position, increasing the potential loss rather than protecting against it.

**How it fires:** D2 activates when a stop-loss order is moved further from the entry price on a losing position. The signal is sensitive to the pattern of progressive widening: multiple modifications in the same session, each moving the stop further out, produce a stronger signal than a single adjustment. The system also tracks naked position time, where a position exists without a stop-loss, as a continuous state signal feeding into D2.

**Behavioral finance context:** Stop manipulation is the execution-level expression of the disposition effect (Shefrin and Statman 1985): the tendency to hold losing positions too long while cutting winning ones too early. Widening a stop on a losing position extends the hold time on a trade that has already gone against the plan. Research on professional traders (Coval and Shumway 2005) confirms that traders experiencing losses in a session take on significantly increased risk in subsequent trades, consistent with the behavior D2 captures.

---

## D3: Size Spike

**What it measures:** Position sizing that exceeds the trader's declared rules or normal historical range, particularly when the spike occurs during or immediately after a losing sequence.

**How it fires:** D3 has two components. The first is a rule violation component: if the trader has declared a maximum position size and an entry exceeds it, D3 fires with a penalty signal. The second is a behavioral pattern component: D3 also monitors the relationship between recent losses and position size escalation, where sizing that is elevated relative to the trader's historical distribution carries heavier weight when it occurs during a losing sequence. A size increase on a winning streak is different from the same size increase after three consecutive losses.

**Behavioral finance context:** Position size escalation under loss conditions is one of the most consistently documented behavioral patterns in trader research. The impulse to "make it back" with a larger trade is mathematically counterproductive: larger positions during periods of behavioral deterioration compound the risk at precisely the moment when execution quality is most likely to be impaired. D3 is designed to detect this escalation pattern from the order data rather than from self-reports.

---

## D4: Rushed Exit

**What it measures:** Exits from profitable positions that collapse significantly earlier than the trader's own historical exit timing, capturing the "cut winners short" pattern.

**How it fires:** D4 fires when a trade is exited for a profit but at a hold time that is abnormally short relative to the trader's own historical distribution for winning trades. The signal requires a sufficiently mature baseline before it fires reliably: the system needs enough historical trades to establish a meaningful distribution of the trader's normal exit timing. In early sessions, D4 is intentionally suppressed to prevent false positives from incomplete baseline data.

**Behavioral finance context:** The disposition effect's "cut winners short" side is the complement of its "let losers run" side. Traders systematically exit profitable positions too early, preserving the certainty of a gain over the possibility of a larger one. This asymmetry — paired with the stop-manipulation tendency to extend losers — produces a characteristic return distribution that underperforms the trader's stated strategy. D4 measures this behavioral pattern from exit timing rather than from P&L attribution.

---

## D5: Position Overstay

**What it measures:** Holding a losing position significantly beyond the trader's own historical tolerance window for adverse positions, capturing the "let losers run" pattern.

**How it fires:** D5 is a continuous state signal: it does not fire on a discrete event but accumulates as a losing position's hold time extends past the trader's baseline. The signal records deep adverse excursion even when the trade later recovers: if a position reached a significant adverse excursion and was then held until breakeven or a small profit, the behavioral pattern of overstaying the loss is still captured. The recovery outcome does not erase the behavioral evidence.

**Behavioral finance context:** D5 is the direct execution-level measurement of the loss-aversion-driven tendency to hold losers. Unlike D4, which captures premature exits on winners, D5 captures extended holds on losers. Research on retail futures traders (Garvey and Murphy 2004) documents that losing trades are held substantially longer than winning trades, the precise asymmetry D5 measures. Position Overstay is the most structurally significant behavioral signal in terms of its relationship to session-level P&L outcomes.

---

## D6: Rule Violations

**What it measures:** Trading activity that crosses the session parameters the trader themselves declared: trading outside the defined time window, position sizing outside the declared range, entering an instrument they marked as off-limits, taking more trades than the declared maximum.

**How it fires:** D6 fires on discrete rule-crossing events. The trader declares their own session parameters in the Meridian configuration: session time window, maximum position size, allowed instruments, maximum trade count per session, stop-loss range requirements. When any of these is crossed, D6 fires with a penalty signal. The signal uses a cooldown to prevent double-counting: a trade that violates multiple rules at once counts once, not multiple times.

**Behavioral finance context:** The gap between stated rules and actual behavior is one of the most consistent findings in trader self-assessment research. Traders who perform well in simulation or in calm post-session reflection routinely violate their own rules in live conditions. Rule Violations measures this gap from the outside: it does not require the trader to self-report whether they violated a rule; it observes whether the order they placed was consistent with the parameters they declared.

---

## D7: Overtrading Pace

**What it measures:** Entry frequency that has accelerated significantly above the trader's own established session rhythm, independent of whether the individual trades are within size rules.

**How it fires:** D7 monitors the pace at which the trader is entering positions and compares it to the trader's own baseline entry frequency distribution. The signal has a warm-up window at the start of each session: early trades, when the session is establishing its pace, are given wider tolerance to prevent false positives from normal session opening variability. Once the session has established enough data points, D7 becomes sensitive to significant pace acceleration. The signal is primarily relevant for traders who can have overtrading as a distinct behavioral failure mode: it is less applicable to low-frequency traders who may enter only a few times per session.

**Behavioral finance context:** Overtrading is among the most replicated findings in the retail trading literature. Barber and Odean (2000) document that high-turnover retail traders underperform low-turnover traders consistently, controlling for other factors. The mechanism is not simply transaction costs: overtrading during behavioral deterioration represents compulsive entry-seeking behavior, where the trader is entering because the urge to be in a trade is elevated, not because a new signal with positive expected value has appeared. D7 measures the pace signature of this behavioral state from the execution record.

---

## Signal composition and weighting

The seven signals feed into the PSI score through a weighted composition. Each signal has an individual weight that the trader can tune from 0 to a maximum value. Setting a signal's weight to 0 disables it entirely, which is appropriate for trading styles where that signal is not meaningful (for example, a long-only swing trader may reasonably disable D7 if session trade count is always low by design).

The default weights reflect the relative frequency and impact of each behavioral pattern across discretionary futures traders, based on observed session data. They are a starting point, not a prescription.

The composition uses an exponentially weighted moving average framework for each signal dimension, which produces continuous PSI updates rather than step-function jumps. Signals decay over time when no further evidence of the pattern arrives, which allows PSI to recover within a session if behavior improves. The decay rates are calibrated so that a single isolated signal event does not produce a severe PSI drop, while sustained multi-dimensional pressure produces a sustained low reading.

---

## Signal interaction effects

The seven signals are designed to be independent in their measurement, but their simultaneous activation is not independent in its meaning. When D1 (Revenge Entry), D2 (Stop Manipulation), and D3 (Size Spike) are all active simultaneously, the compound reading is more significant than any individual signal would be alone. The PSI scoring framework captures this through the additive composition: when multiple signals are active at the same time, the composite score reflects the aggregate pressure.

This interaction effect is one of the reasons PSI is more informative than any individual signal tracked in isolation. A single high D1 reading may reflect a legitimate aggressive entry. A simultaneous high reading on D1, D2, D3, and D7 is a different situation entirely, and the composite PSI score reflects that difference.

---

## Further reading

- [The Psychological Stability Index (PSI)](01-psychological-stability-index.md)
- [NinjaTrader 8 Integration Architecture](03-ninjatrader-integration-architecture.md)
- [Meridian product documentation](https://www.meridianpsi.com/psi-monitor)
- [PSI Monitor feature page](https://www.meridianpsi.com/psi-monitor)

**Research references:**

Barber, B. and Odean, T. (2000). "Trading Is Hazardous to Your Wealth." *Journal of Finance* 55(2).

Coval, J. and Shumway, T. (2005). "Do Behavioral Biases Affect Prices?" *Journal of Finance* 60(1).

Garvey, R. and Murphy, A. (2004). "Are Professional Traders Too Slow to Realize Their Losses?" *Financial Analysts Journal* 60(4).

Kahneman, D. and Tversky, A. (1979). "Prospect Theory: An Analysis of Decision under Risk." *Econometrica* 47(2).

Odean, T. (1998). "Are Investors Reluctant to Realize Their Losses?" *Journal of Finance* 53(5).

Shefrin, H. and Statman, M. (1985). "The Disposition to Sell Winners Too Early and Ride Losers Too Long." *Journal of Finance* 40(3).

---

*This document describes the behavioral signal framework used by Meridian, a real-time psychological stability monitor for NinjaTrader 8. Trading involves substantial risk of loss. Meridian does not provide trading signals or investment advice.*
