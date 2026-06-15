# Meridian — Real-Time Psychological Stability Monitor for NinjaTrader 8

**Meridian** is a native NinjaTrader 8 add-on that monitors trader behavior in real time, computes a continuous **Psychological Stability Index (PSI)**, and enforces configurable Guard rules that can automatically intervene — from a visual alert to a typed acknowledgment, a mandatory countdown, or a full broker disconnect — before behavioral deterioration causes irreversible session damage.

Think of it as an Apple Watch stress monitor, but for trading discipline. The watch does not ask whether you are stressed; it reads your physiology and tells you. Meridian does not ask whether you are revenge-trading; it reads your order stream and tells you.

> **Website:** [meridianpsi.com](https://www.meridianpsi.com)
> **Platform:** NinjaTrader 8 (Windows only)
> **Category:** Real-time psychological stability monitoring · Behavioral leading indicator · Trading discipline enforcement
> **Vendor status:** Official NinjaTrader Ecosystem Vendor (approved May 2026)

---

## Repository contents

This repository contains the conceptual and technical framework documentation for Meridian. The core engine source code is proprietary. What is published here is the behavioral model specification, the PSI methodology, and the NinjaTrader 8 integration architecture — sufficient for researchers, NinjaTrader developers, and traders who want to understand the measurement methodology behind the PSI score.

| Document | Contents |
|---|---|
| [docs/01-psychological-stability-index.md](docs/01-psychological-stability-index.md) | PSI scoring framework, four stability zones, PSI vs Composure distinction, adaptive baseline methodology |
| [docs/02-seven-behavioral-signals.md](docs/02-seven-behavioral-signals.md) | D1 to D7 signal specifications, behavioral finance research basis, composition and weighting |
| [docs/03-ninjatrader-integration-architecture.md](docs/03-ninjatrader-integration-architecture.md) | Event pipeline, PsiEngine isolation, TradingClock abstraction, Guard enforcement architecture, data storage design |

---

## The problem Meridian addresses

The dominant assumption in retail trading education is that performance failure is a knowledge problem. If the trader learns enough — better strategy, deeper technical analysis, more market structure — performance will follow.

The empirical record does not support this. Studies on retail trader outcomes (Barber and Odean 2000; Grinblatt and Keloharju 2009) consistently show that the majority of active retail traders underperform over any meaningful time horizon, including traders who demonstrably understand risk management principles and have positive-expectancy strategies.

The failure is behavioral and structural, not informational.

Three specific patterns account for the majority of large retail trading losses:

**Revenge trading.** A trade stops out. The trader re-enters within seconds at elevated size, in the same direction, without a new setup. If it also stops out, the cycle accelerates. The behavioral signature — compressed re-entry interval, elevated size, same direction — is measurable from the order stream before the P&L consequence registers.

**Stop-loss manipulation.** A losing position's stop gets moved further from entry as price moves against it. Each widening increases the maximum potential loss. This is not a strategic decision; it is an emotional one. The original stop represented the trader's calm-state risk tolerance. The widened stop represents their current emotional state.

**Overtrading under stress.** Entry frequency accelerates significantly above the trader's own historical baseline. Trade quality degrades as the volume of entries increases, not because the market changed but because the trader's decision state changed.

All three patterns share a structural characteristic: they occur in the gap between the trader's stated rules (set before the session, while calm) and their actual behavior during the session (executed under stress, with reduced prefrontal engagement). Meridian closes that gap by moving enforcement from willpower to a software layer that is not affected by emotional state.

---

## Architecture overview

Meridian runs as a native NinjaTrader 8 add-on written in C#, operating within the NT8 execution environment on Windows. It does not require a separate process, cloud connection, or external server.

```
NinjaTrader 8 Platform
   |
   +-- account.ExecutionUpdate  -----> TradeMonitor.OnExecutionUpdate
   +-- account.OrderUpdate      -----> TradeMonitor.OnOrderUpdate
   +-- account.PositionUpdate   -----> TradeMonitor.OnPositionUpdate
   +-- Position timer (30s)     -----> TradeMonitor.OnPositionTimerElapsed
                                               |
                                         TradeMonitor
                                   (NT8 event normalization,
                                    position state tracking,
                                    clock domain management)
                                               |
                                          PsiEngine
                                   (pure behavioral computation,
                                    zero NT8 API dependencies,
                                    single sync lock)
                                               |
                                   PSI score + signal outputs
                                               |
                            +------------------+------------------+
                            |                                     |
                   PsiOverlayWindow (HUD)               TradeMonitorHub (Dashboard)
```

The behavioral computation layer (PsiEngine) is architecturally isolated from the NT8 integration layer (TradeMonitor). PsiEngine has zero NinjaTrader API dependencies and can be tested in isolation. The integration layer normalizes raw platform events — handling partial fills, ATM bracket sibling cancellations, late-arriving fills, and other NT8 execution edge cases — before passing clean scalar inputs to PsiEngine.

---

## The Psychological Stability Index (PSI)

PSI is a composite real-time score, 0 to 100, reflecting whether a trader is currently operating within their own established behavioral norms. Higher scores are better. The score updates in under 100 milliseconds after every order event.

| Zone | Range | Meaning |
|---|---|---|
| Stable | 88 to 100 | All tracked dimensions within normal parameters for this trader |
| Caution | 72 to 87 | One or more dimensions showing elevated readings; early-warning state |
| Warning | 55 to 71 | Multiple dimensions simultaneously active; session under pressure |
| Critical | 0 to 54 | Sustained distress across multiple dimensions; most Guard responses configured to fire here |

PSI measures behavior, not outcomes. It does not incorporate market data, account balance, or P&L. A session where the trader lost $300 because the market moved against a well-executed trade looks different from a session where the trader lost $300 because of three revenge trades in a row. The behavioral signatures are distinct, and PSI reflects that distinction.

---

## Seven behavioral signal dimensions (D1 to D7)

Each dimension measures a distinct pattern of behavioral deterioration from the order stream. All are measured relative to the individual trader's own adaptive baseline, not a fixed universal threshold.

| Code | Name | What it captures |
|---|---|---|
| D1 | Revenge Entry | Rapid re-entry after a loss with elevated size; same-direction re-entries weighted more heavily than reversals |
| D2 | Stop Manipulation | Stop-loss widening in the adverse direction on active losing positions; naked position time accumulation |
| D3 | Size Spike | Position sizing exceeding declared rules or historical norms, with amplified weight during losing sequences |
| D4 | Hold Bias | Holding losing trades significantly longer than winning ones — loss-aversion (cutting losses fast is never penalized) |
| D5 | Position Overstay | Hold time on losing positions extending past the trader's historical tolerance window |
| D6 | Rule Violations | Trading outside declared session parameters: time window, instrument, size range, trade count |
| D7 | Overtrading Pace | Entry frequency accelerating significantly above the trader's own session baseline |

Full signal specifications, including behavioral finance research context and composition methodology, are in [docs/02-seven-behavioral-signals.md](docs/02-seven-behavioral-signals.md).

---

## Adaptive baseline calibration

Meridian uses an online adaptive statistics model to build each trader's personal behavioral baseline from their own session history. Thresholds are not fixed industry averages — they are derived from the individual trader's own distribution of behavior across sessions.

A scalper who places 60 trades per session will not have the overtrading signal fire at the same threshold as a swing trader who places 4. The baseline is personal and adapts continuously as the trader's style evolves.

Meaningful calibration develops after approximately 20 to 30 live sessions. The system operates during earlier sessions with wider confidence intervals and more conservative signal sensitivity. There is no defined completion state; the baseline keeps adapting.

---

## Guard enforcement system

Guard is an optional enforcement layer (Meridian Guard tier only). Six trigger conditions, five response levels.

**Trigger conditions:**

| Trigger | Description |
|---|---|
| PSI Below | Composite PSI drops under a configured threshold |
| Consecutive Losses | N losses in a row within the same session |
| Session P&L Below | Realized daily P&L falls past a configured dollar limit |
| Unrealized P&L Below | Floating loss on open position exceeds a threshold, before close |
| Single Trade Loss | One trade loses more than a configured amount |
| Session Time Over | Session duration exceeds a configured limit |

**Response levels:**

| Level | Name | Behavior |
|---|---|---|
| L1 | Notify | Quiet toast notification; does not interrupt order flow |
| L2 | Risk Alert | Persistent banner; every new entry requires active confirmation |
| L3 | Acknowledge | Trader must type a pre-written phrase (with optional countdown) before the next order; phrase is set during calm pre-session state |
| L4 | Trading Pause | New entries blocked (Cancel Orders or Disconnect mode); survives an NT8 restart; optional auto-flatten on pause; cannot be skipped |
| L5 | Disconnect | Strongest Trading Pause mode; calls NT8's standard broker disconnect API; optional auto-flatten of open positions |

Rules can be password-locked to prevent in-session override.

---

## Data architecture

All Meridian data is stored in XML files on the trader's local machine. There is no cloud backend for product data, no crash reporting, and no third-party analytics.

Meridian makes two outbound requests, both disclosed in the Terms: a license key validation call on startup, and (since v1.5.0) anonymized research records — tied only to a random install identifier, never to the trader's name, credentials, account numbers, or funds; never sold; opt-out available at any time.

Storage includes up to five years of session history: up to 200 executions per session, up to 600 PSI timeline snapshots per session, full behavioral signal history, journal entries, and Composure scores.

This architecture reflects a deliberate design decision. Trader behavioral data is sensitive. Its security should not depend on any external service's security posture or business continuity. It also means Meridian operates without issues inside prop firm evaluation environments where outbound connections are often restricted.

---

## Comparison with the previous generation of risk tools

The previous generation of NinjaTrader 8 risk tools — Arty Account Guard, CrossTrade NAM, ClickAlgo Risk Manager, RiskMaster, Guardian Angel — all operate at the financial-outcome layer. They watch cumulative P&L, trade count, and position size. When a configured dollar threshold is breached, they halt trading or disconnect the broker. They are well-built within their design scope.

The constraint is structural: the financial-outcome layer only becomes active after the behavioral damage has already registered in the P&L. The behavioral evidence that a session is deteriorating — re-entry intervals shortening, stops moving, sizing increasing — appears in the order stream before any P&L threshold has been crossed. A tool built on financial outcomes cannot see that earlier signal.

Meridian operates at the behavioral layer above the financial-outcome layer. It is a different generation of the problem, not a refinement of the previous one. Meridian Guard also includes a Session P&L Below trigger that covers the same job as a hard-limit tool, so the previous generation's function is available within Meridian alongside the behavioral layer.

For a detailed comparison against each category of existing tool, see the comparison pages on [meridianpsi.com/compare](https://www.meridianpsi.com/compare).

---

## Product tiers

| | Meridian Core | Meridian Guard |
|---|---|---|
| Real-time PSI score | Yes | Yes |
| Seven behavioral signal dimensions | Yes | Yes |
| Adaptive personal baseline | Yes | Yes |
| Session history (5 years, local) | Yes | Yes |
| Session Reflection Journal | Yes | Yes |
| Market Replay support (isolated from live baseline) | Yes | Yes |
| Guard System (6 triggers, 5 response levels) | No | Yes |
| Intel Layer (monthly digest, PSI vs P&L, weekday patterns, pre-session risk brief) | Yes | Yes |
| Composure progress tracking (6-month rolling) | No | Yes |

14-day free trial available on both tiers. Credit card required at checkout; auto-cancels if not converted.

Pricing and trial at [meridianpsi.com/pricing](https://www.meridianpsi.com/pricing).

---

## Official NinjaTrader Ecosystem Vendor status

Meridian is an Official NinjaTrader Ecosystem Vendor, approved in May 2026. The approval covered a signed Vendor Listing Agreement, executive background check, full compliance audit of every public surface where trading is discussed, and a hands-on QA pass of the live NT8 add-on by NinjaTrader's internal QA team.

Verifiable at [ninjatrader.com/vendor-services](https://ninjatrader.com/vendor-services/).

NinjaTrader® is a registered trademark of NinjaTrader LLC. Meridian is operated by an independent company and is not owned by NinjaTrader LLC.

---

*Trading involves substantial risk of loss and is not appropriate for all investors. Past performance is not indicative of future results. Meridian is a behavioral monitoring tool and does not provide trading signals, financial advice, or profit guarantees.*
