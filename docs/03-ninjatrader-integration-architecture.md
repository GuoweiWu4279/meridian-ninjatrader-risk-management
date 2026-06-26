# NinjaTrader 8 Integration Architecture

## Overview

Meridian integrates with NinjaTrader 8 through the platform's native add-on infrastructure. It is not a chart indicator. It does not appear in the chart indicator list. It is an Add-On, accessible through NinjaTrader's Control Center under New > Add-on > Meridian Dashboard.

The integration gives Meridian direct access to NinjaTrader 8's account execution feed: every fill, order modification, and position change is observed in real time. This live order pipeline access is what makes real-time behavioral signal detection possible. A post-session analytics tool does not require this; a real-time psychological stability monitor cannot function without it.

---

## The event pipeline

NinjaTrader 8 fires three primary account-level events that Meridian observes:

**ExecutionUpdate** fires when a fill is confirmed: an entry, exit, or partial fill. This is the primary event for behavioral signal computation. Every D1 (Revenge Entry), D3 (Size Spike), D4 (Hold Bias), and D7 (Overtrading Pace) signal begins with an ExecutionUpdate event.

**OrderUpdate** fires when an order's state changes: placement, modification, cancellation, or rejection. Stop-loss modifications that underlie D2 (Stop Manipulation) are captured here. The order modification event carries the new stop price; Meridian compares it against the position's entry price and the trader's baseline stop distance distribution to determine whether the modification constitutes a widening on an adverse position.

**PositionUpdate** fires when NinjaTrader's internal position tracking state changes. Meridian uses this primarily for D5 (Position Overstay) and D2 naked-position monitoring, where the continuous state of a position — its current unrealized P&L, its hold duration, its relationship to the trader's historical position management patterns — matters more than the discrete fill events.

A fourth event source, a 30-second position timer, fires continuously during live trading. It drives the continuous-state signal computation for D2 and D5, which require measurement at regular intervals rather than only on discrete order events.

---

## The PsiEngine separation

The behavioral computation layer (PsiEngine) is architecturally isolated from the NinjaTrader integration layer (TradeMonitor). This separation has practical significance: PsiEngine has zero NinjaTrader API dependencies. It is a pure computation engine that receives scalar inputs and produces PSI outputs. It does not make NT8 API calls, does not access account objects, and does not reference NinjaTrader-specific types.

This design means the core behavioral computation logic can be tested in isolation, without a running NinjaTrader instance. The NinjaTrader integration layer normalizes the raw platform events into clean scalar inputs before passing them to PsiEngine. Edge cases in NT8's execution semantics — partial fills, ATM bracket sibling cancellations, late-arriving fills, order rejection sequences — are handled at the integration layer and are transparent to the computation layer.

The architecture looks like this:

```
NinjaTrader 8 Platform
   |
   +-- account.ExecutionUpdate  -------> TradeMonitor.OnExecutionUpdate
   +-- account.OrderUpdate      -------> TradeMonitor.OnOrderUpdate
   +-- account.PositionUpdate   -------> TradeMonitor.OnPositionUpdate
   +-- Position timer (30s)     -------> TradeMonitor.OnPositionTimerElapsed
                                                |
                                          TradeMonitor
                                    (normalizes NT8 events,
                                     tracks position state,
                                     handles clock domain)
                                                |
                                           PsiEngine
                                    (pure behavioral computation,
                                     no NT8 dependencies,
                                     single sync lock)
                                                |
                                    PSI score output
                                                |
                             +------------------+------------------+
                             |                                     |
                    PsiOverlayWindow (HUD)              TradeMonitorHub (Dashboard)
```

---

## The TradingClock abstraction

NinjaTrader 8 supports Market Replay, which allows traders to replay historical market data through the platform's standard execution infrastructure. From the perspective of code that observes order events, a Market Replay session looks identical to a live session: the same ExecutionUpdate, OrderUpdate, and PositionUpdate events fire, carrying fill prices, timestamps, and position data.

Without explicit handling, a behavioral monitoring system would contaminate its live baseline data with Market Replay sessions, treating replay fills as if they were live trading decisions. Meridian uses a TradingClock abstraction to resolve this. All time-sensitive code goes through TradingClock rather than reading system time directly. TradingClock knows whether the current session is Live or Replay, and it adjusts accordingly.

The baseline admission system uses a DataProvenance label on every data point. Only data points with Provenance equal to Live are admitted to the adaptive baseline. Market Replay sessions are tracked in a separate data context, isolated from the live baseline, so replay activity cannot shift the thresholds that govern live signal sensitivity.

This isolation is particularly relevant for traders who use Market Replay for practice and strategy testing. Their practice sessions do not affect the behavioral baseline that governs their live trading.

---

## The Guard enforcement pipeline

Meridian Guard is an optional enforcement layer that sits above the PSI monitoring layer. Guard is not required for PSI monitoring to function. Traders on the Meridian Core tier have full PSI monitoring without any Guard configuration.

When Guard is enabled (Meridian Guard tier only), the trader configures trigger conditions and response levels in a dedicated Guard section of the Dashboard. Guard rules are entirely separate from the Settings configuration, which handles the trading profile (session time window, position sizes, signal weights, response presets). This separation is intentional: settings that govern measurement and settings that govern enforcement are managed in different places.

**The six trigger conditions:**

1. PSI Below a configured threshold
2. N Consecutive Losses within the session
3. Session P&L Below a configured dollar amount
4. Unrealized P&L Below a configured threshold (fires before the position closes)
5. Single Trade Loss exceeding a configured amount
6. Session Time Over a configured duration

Each trigger fires once per condition entry, then resets. The edge-trigger design prevents repeated firing on a condition that remains true: if PSI drops below the threshold and stays there, the trigger fires once, not continuously. A cooldown mechanism prevents rapid re-triggering between different conditions.

**The five response levels:**

Guard responses escalate in friction. The trader assigns each rule to one of five levels:

Level 1 (Notify): A quiet visual notification. Does not interrupt order flow.

Level 2 (Risk Alert): A persistent banner appears across the interface and stays up while the condition holds. It is non-blocking — it keeps the risk in front of you but does not gate entries; Level 3 (Acknowledge) is the level that blocks the next order.

Level 3 (Acknowledge): The trader must type a pre-written phrase — with an optional countdown timer — before placing the next order. The phrase is defined during the pre-session configuration, when the trader is calm. It is typed during the triggered state, when the trader may not be. Examples: "I am not ready to trade" or "Check your PSI before the next entry." The friction is the mechanism.

Level 4 (Trading Pause): New entries are blocked entirely, in Cancel Orders mode (stay connected, new entry orders auto-cancelled) or Disconnect mode. The pause survives a NinjaTrader restart or crash until its window expires. Orders that close or reduce a position are never blocked.

Level 5 (Disconnect): The strongest Trading Pause mode. NinjaTrader's standard broker disconnect API is called — the same operation as manually clicking Disconnect in NinjaTrader's interface. For open positions, Guard can be configured to submit closing orders before disconnecting (auto-liquidation), or to leave open positions under manual management.

Rules can be password-locked to prevent in-session override. The recommended configuration for traders who want the enforcement to hold under pressure: use a password that will not be remembered during a stressed session.

---

## Data storage and the no-cloud design

All Meridian data is stored in XML files on the trader's local machine. Session history, behavioral baselines, configuration, journal entries, and PSI timeline snapshots are all local. Retention is up to five years of session history.

Meridian makes two kinds of outbound requests, both disclosed in the product's Terms: a license key validation call on startup, and (since v1.5.0) anonymized research records — trading activity and computed behavioral signals tied only to a random install identifier, never to the trader's name, credentials, account numbers, or funds. The research contribution is never sold and can be opted out of at any time. There is no crash reporting and no third-party analytics.

This architecture was chosen specifically to support traders in prop firm evaluation environments, where outbound connections from the trading platform are often restricted or prohibited. Meridian operates within these environments without modification.

The per-session storage includes: up to 200 executions per session, up to 600 PSI timeline snapshots per session, the full behavioral signal history, journal entries, and Composure scores. Older entries are pruned automatically when the storage window exceeds the configured retention period.

---

## Multi-account handling and Replikanto

Traders who run multiple NinjaTrader accounts simultaneously, or who use Replikanto to copy fills across accounts, would produce double-counted behavioral signals without explicit handling. A single trading decision that results in fills in two accounts would appear as two Revenge Entries, two Size Spikes, and so on.

Meridian deduplicates Replikanto copy-fills within a 250-millisecond window. Fills that arrive within this window across accounts with matching instrument, direction, and quantity are recognized as copies of a single trading decision and counted once. The behavioral signals fire once for the decision, not once for each copy.

For traders running genuinely independent decisions across multiple accounts simultaneously, all connected accounts are monitored and their events are pooled into a single PSI computation. This reflects the behavioral reality: the trader is making decisions across all accounts, and the aggregate behavioral pattern is what matters for PSI.

---

## Further reading

- [The Psychological Stability Index (PSI)](01-psychological-stability-index.md)
- [Seven Behavioral Signal Dimensions (D1 to D7)](02-seven-behavioral-signals.md)
- [Meridian Guard product page](https://www.meridianpsi.com/guard)
- [Meridian features overview](https://www.meridianpsi.com/features)
- [Installation guide](https://www.meridianpsi.com/installation-guide)

---

*This document describes the integration architecture of Meridian, a real-time psychological stability monitor for NinjaTrader 8, developed by an Official NinjaTrader Ecosystem Vendor. NinjaTrader® is a registered trademark of NinjaTrader LLC. Meridian is independently operated and is not owned by NinjaTrader LLC. Trading involves substantial risk of loss.*
