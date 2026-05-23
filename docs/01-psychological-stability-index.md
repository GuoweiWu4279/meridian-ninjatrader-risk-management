# The Psychological Stability Index (PSI)

## What PSI measures

The Psychological Stability Index is a composite real-time score, ranging from 0 to 100, that reflects whether a trader is currently operating within their own established behavioral norms. A score of 100 means all measured behavioral dimensions are within the trader's personal baseline. A score approaching 0 means sustained, multi-dimensional behavioral distress signals are present simultaneously.

The critical design constraint is that PSI measures behavior, not outcomes. This distinction is not semantic — it determines everything about when the score can act as a useful signal.

A trader who has lost $500 has produced a financial outcome. That outcome is the result of decisions and behaviors that occurred over the preceding period. The behavioral events that produced it — the shortened re-entry intervals, the widened stops, the increased sizing under loss conditions — occurred before the $500 loss was finalized. A score built on financial outcomes can only confirm what already happened. A score built on behavioral events can signal that a deterioration is in progress while it is still in progress.

This is the difference between a lagging indicator and a leading indicator. PSI is designed to be a leading indicator of behavioral state, not a lagging report of financial damage.

---

## The four stability zones

PSI values are grouped into four named zones. The zone names are deliberate: they are calibrated to produce appropriate cognitive salience without triggering defensive dismissal.

**Stable (88 to 100)**

All tracked behavioral dimensions are within normal parameters for this trader. Session execution is consistent with the trader's own established patterns. No intervention is indicated.

**Caution (72 to 87)**

One or more behavioral dimensions are showing readings above baseline. This is an early-warning state. The session has not deteriorated, but a direction has been established that warrants attention. Meridian surfaces this visually; no automated Guard response fires at this level unless the trader has specifically configured one.

**Warning (55 to 71)**

Multiple behavioral dimensions are simultaneously active. The session is under measurable pressure across more than one axis. This zone was added in Meridian v1.2.0 to provide a clearer intermediate signal between Caution and Critical. Most Guard configurations will begin firing at or before the bottom of this zone.

**Critical (0 to 54)**

Sustained behavioral distress across multiple dimensions simultaneously. The probability of a significant behavioral failure — revenge trading, oversizing, stop manipulation — is substantially elevated. The majority of Guard enforcement responses are calibrated to fire at or near this threshold.

---

## PSI vs Composure: two different measurements

PSI and Composure are both outputs of the Meridian system, but they measure different things and should not be confused.

**PSI** is a real-time measurement of the trader's current behavioral state during an active session. It updates continuously, under 100 milliseconds after every order event. It can move up and down within a session as behavioral signals activate and decay. PSI is the live diagnostic.

**Composure** is a session-quality score computed at the end of the session. It is a weighted function of the time the trader spent in each PSI zone throughout the session: a session spent primarily in the Stable zone receives a high Composure score; a session that spent significant time in Critical receives a low one. Composure is the session-level grade. It is tracked over a rolling six-month period to surface trends in behavioral quality over time.

The distinction matters because a trader can recover within a session. A session that starts in Critical but ends in Stable will show a PSI of 85 at close. That PSI reading is accurate but misleading as a summary of the session. Composure captures the full session arc.

---

## Adaptive baseline calibration

One of the most consequential design decisions in the PSI framework is that thresholds are personal, not universal.

A fixed threshold system would define "overtrading" as, for example, more than 15 trades in a session. That threshold is meaningless without context. A trader who normally places 30 trades in a session is not overtrading at 15. A trader who normally places 6 is potentially under significant behavioral pressure at 10. The same absolute number has different meaning depending on who is being measured.

Meridian's adaptive baseline learns each trader's own patterns across sessions and uses those patterns as the reference point for each behavioral signal. The baseline is built using an online adaptive statistics model: it updates continuously, weighted toward recent sessions, without requiring manual calibration or a defined "training period."

In practice, the system reaches a meaningfully calibrated state after approximately 20 to 30 live sessions. The first 5 to 10 sessions operate with wider confidence intervals and more conservative signal sensitivity. There is no defined end state; the baseline continues adapting as the trader's style, instruments, or session patterns evolve.

This design choice means PSI is a personal measurement. A score of 60 means something different for each trader, because the signals that produced it are each measured against that trader's own baseline, not a generic industry threshold.

---

## Measurement accuracy as the primary design constraint

The PSI framework enforces a strict hierarchy of correctness criteria for any change to the scoring system:

1. Measurement accuracy: Does PSI reflect what the trader is actually doing right now?
2. Signal classification compliance: Does each signal fire on the correct type of event?
3. Platform correctness: Does the implementation handle NinjaTrader 8's execution semantics accurately?
4. Technical correctness: Is the code logically sound?

A technically correct implementation that fails the first two criteria is wrong by the framework's definition, regardless of how cleanly it is written. Measurement accuracy is not a property of the output; it is the purpose of the output. The PSI score has no value if it does not accurately reflect behavioral state.

This constraint has direct implications for what PSI must never respond to: platform event ordering quirks, partial-fill artifacts, clock-domain switches between live and replay trading, or position-tracking corrections that do not represent actual trader decisions. Each of these can produce spurious movements in a naive implementation. The framework explicitly forbids them.

---

## The design principle: measure the trader, not the market

PSI does not incorporate market data. It does not adjust scores based on volatility, news events, or price action. This is intentional.

The behavioral signals Meridian measures are meaningful precisely because they are deviations from the trader's own established patterns, independent of what the market is doing. A revenge-trading entry is a revenge-trading entry whether the market is trending or ranging. An unusually shortened re-entry interval is a behavioral signal whether the preceding stop-out was caused by a spike or a slow grind. The market context may explain why the trader is under stress; it does not change whether the behavior is consistent with the trader's own norms.

Excluding market data also keeps the system unambiguously in the behavioral monitoring category. PSI is not a market signal, a volatility indicator, or a regime-classification model. It is a measurement of trader behavior against the trader's own baseline.

---

## Further reading

- [Seven Behavioral Signal Dimensions (D1 to D7)](02-seven-behavioral-signals.md)
- [NinjaTrader 8 Integration Architecture](03-ninjatrader-integration-architecture.md)
- [Meridian product documentation](https://www.meridianpsi.com/what-is-meridian-psi)
- [PSI glossary entry](https://www.meridianpsi.com/glossary)

---

*This document describes the conceptual framework behind the Meridian PSI scoring system. Meridian is a real-time psychological stability monitor for NinjaTrader 8 traders, developed and maintained by an Official NinjaTrader Ecosystem Vendor. Trading involves substantial risk of loss. Meridian does not provide trading signals or investment advice.*
