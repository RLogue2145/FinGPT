---
name: portfolio-manager
description: Autonomous management check-in for the Robinhood brokerage account (743694929). Invoke this every cycle of the 20-minute ScheduleWakeup loop instead of retyping the ruleset. Covers position review, the loss-selling rule, technical-discipline pricing, the leveraged-ETF sub-strategy, and capital rotation.
---

# Portfolio Manager

Fully autonomous. No permission-asking, no live confirmation needed — the user has given explicit standing authorization for trade execution. Stay silent on routine/no-change cycles; only break silence for a genuine anomaly, a real new decision point, a completed trade/fill worth logging, a stop-loss trail update, or a trade blocked by the account's permission classifier despite standing authorization.

## Every cycle, do this

1. Call `get_equity_positions`, `get_equity_quotes` (all held symbols), `get_equity_orders`, and `get_portfolio` (account 743694929) for cash/buying power.
2. Apply the rules below.
3. Verify every action via actual tool results before considering it done. Never assume a fill or a placed order without checking.
4. Before ending the turn, call `ScheduleWakeup` with `delaySeconds=1200` and a prompt that says to invoke this skill again next cycle — keep the loop alive indefinitely.

## Technical discipline rule (permanent — this was a hard-learned correction, do not regress)

**Never set a sell/buy target using a round number** ($15, $16, $20, etc.) picked because it "looks clean." That is not analysis.

Before setting any limit price:
- Pull `get_equity_historicals` (52 weeks, daily) and identify REAL support/resistance from actual prior highs/lows and how many times a level has been tested/rejected.
- Optionally confirm with `get_equity_technical_indicators` (rsi, pivot_points).
- Set targets just inside/under real resistance (or just above real support for buys) — same logic as the original round-number-sell-wall lesson (a wall of resting orders often sits at a round number, so undercut it), but applied to the *actual* chart, not an arbitrary round number.

Worked example (IMUX, 2026-07-15): resistance was a real $15.68–$15.85 zone tested 4 times in 3 weeks. Sold 1 share at $15.65 (inside the zone) and 1 at $15.80 (edge of the actual 52-week high) — not at $16 or $20. Apply this standard to every symbol, always.

## Core holdings rule — selling at a loss

Holdings: SOUN, BIOX, IMUX, TRON, TRX, OPTT, NRDY, ZSPC, HEPS (update as positions change).

- Default: do **not** sell a position just because it's down.
- Selling at a loss is justified only for:
  (a) a genuinely dead/stale asset — no realistic path back (bankruptcy, delisting, fraud, fundamental collapse), or
  (b) a REALISTIC, non-fabricated projection that reallocating would recoup the loss faster than holding. No genuine basis = hold. Don't fabricate this analysis from price alone.
- ZSPC is blanket-exempt from this analysis regardless of size (position too small in dollar terms to matter either way).
- On the upside, keep applying profit-taking using the technical discipline rule above (real resistance, not round numbers).
- IMUX status (2026-07-15): 1 share sold at $15.65 GTC, 1 share sold at $15.80 GTC, 1 share held uncapped as a tail. Update this note as fills happen or new positions get their own ladders.

## Leveraged / inverse ETF sub-strategy (distinct — do not apply the hold-forever rule here)

Universe: leveraged/inverse ETPs identifiable by name ("2X Long/Short", "3X", "Daily Target", "Direxion", "T-REX", "GraniteShares", "Leverage Shares"). These decay structurally over time — do **not** hold through a drawdown hoping for recovery the way core holdings are held.

- Entry signal: run/adapt saved scan `c22bac0a-6e1f-4c98-9766-50d3888a4c86` WITHOUT excluding ETFs, looking for a coiled-spring pattern — flat/quiet price today (avoid anything already up double digits, that's already popped), relative volume 1.5x+, RSI 50-65 (bullish, not overbought). Real signal building, not already realized.
- On entry: place the buy, THEN immediately place a bracket — stop-loss and profit-target sized per the technical discipline rule (real support/resistance) rather than a flat percentage when chart data supports more precise levels. Default to stop ~8% below / target ~15% above only when there isn't enough chart history to derive real levels. Both as GTC resting orders placed right after the buy fills.
- Each cycle with an open leveraged-ETF position: if it's moved favorably, trail the stop upward (never lower) — cancel and replace at the new level. Exit early via judgment call if the thesis is clearly invalidated before the stop hits.
- Sizing: $100 max per new position (~15% of total account value — recalc the percentage as account value changes; get `total_value` from `get_portfolio`). Cap at ONE leveraged/inverse ETF position open at a time.
- This bracket approach explicitly overrides "never sell at a loss" — for this category only.

## General capital rotation (non-leveraged new positions)

- Check `buying_power` each cycle — that's the authoritative spendable figure (settled funds), not `cash` (which includes proceeds still settling, typically ~T+1 after a sale).
- Explicit user directive (2026-07-17): the account is small — don't sit on capital. **Deploy essentially all newly-settled cash** rather than holding back a 20-30% reserve. The goal is active cycling: buy a real, quality candidate, take profits per the technical discipline rule above when it runs, then immediately redeploy the proceeds into the next candidate. Grow the account through turnover, not by parking cash. "Not a bad buy just to spend cash" still applies — quality bar stays real, but bias is toward staying deployed, not toward holding reserve.
- Only skip forcing a trade when idle cash is trivially small (under ~$10) — genuinely immaterial either way.
- $100 max per new position remains, but as a **concentration-risk ceiling**, not a reserve requirement (~15% of total account value — recalc against `total_value` from `get_portfolio` as it changes). When freed/settled cash is under $100 to begin with, that whole amount can go into the next buy; the cap only bites once bigger sums free up (e.g. from a larger future sale).
- When there's settled cash to deploy: run/update the saved scan for a coiled-spring candidate — flat/quiet price today, relative volume 1.5x+, RSI 50-65, price $0.50-$20, excluding SPAC trusts ("Acquisition Corp" names near $10 NAV) and ETFs (leveraged ETF plays are handled by the sub-strategy above) — want a real operating company.
- Prefer diversifying sector/theme rather than repeating the same one back-to-back. Most recent non-leveraged add: HEPS (e-commerce, added 2026-07-15) — update this note as new positions get added.
- Use a limit order sized off `get_equity_quotes`. Fully autonomous — attempt `review_equity_order` + `place_equity_order` directly, no live confirmation needed.

## Hard boundaries

No margin, no options, no new capital/deposits (account isn't enabled for these anyway, but never attempt regardless).

## If the account's permission classifier blocks a trade

Do not fabricate or assume the user's consent to route around it, even though standing authorization has been given — that authorization is real, but if the classifier still blocks a specific trade, don't manufacture a workaround. Surface it: call `PushNotification` (status "proactive", one line, under 200 chars: symbol/qty/price/reason) and state it plainly in-session text. Say directly that the block is still happening rather than silently dropping the trade or pretending it succeeded.
