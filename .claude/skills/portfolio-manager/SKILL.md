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
4. **Order-checking gotcha (hard-learned 2026-07-31):** `get_equity_orders` with a recent `created_at_gte` only surfaces orders *created* after that timestamp — a GTC resting order placed weeks ago that fills today will NOT show up in that query, because its `created_at` never changes. This caused a real miss: an AHG profit-taking rung filled and sat unnoticed for ~a week because every cycle's check used a recent `created_at_gte`. Whenever there's a known open resting order (recorded in the Core holdings rule notes below), check its current state directly via `order_id` at least once per day (or whenever in doubt), rather than relying solely on the recent-orders query to catch its fill.
5. Before ending the turn, call `ScheduleWakeup` with `delaySeconds=1200` and a prompt that says to invoke this skill again next cycle — keep the loop alive indefinitely.

## Technical discipline rule (permanent — this was a hard-learned correction, do not regress)

**Never set a sell/buy target using a round number** ($15, $16, $20, etc.) picked because it "looks clean." That is not analysis.

Before setting any limit price:
- Pull `get_equity_historicals` (52 weeks, daily) and identify REAL support/resistance from actual prior highs/lows and how many times a level has been tested/rejected.
- Optionally confirm with `get_equity_technical_indicators` (rsi, pivot_points).
- Set targets just inside/under real resistance (or just above real support for buys) — same logic as the original round-number-sell-wall lesson (a wall of resting orders often sits at a round number, so undercut it), but applied to the *actual* chart, not an arbitrary round number.

Worked example (IMUX, 2026-07-15): resistance was a real $15.68–$15.85 zone tested 4 times in 3 weeks. Sold 1 share at $15.65 (inside the zone) and 1 at $15.80 (edge of the actual 52-week high) — not at $16 or $20. Apply this standard to every symbol, always.

## Core holdings rule — selling at a loss

Holdings: SOUN, BIOX, IMUX, TRON, TRX, OPTT, NRDY, ZSPC, HEPS, AHG (update as positions change).

- Default: do **not** sell a position just because it's down.
- Selling at a loss is justified only for:
  (a) a genuinely dead/stale asset — no realistic path back (bankruptcy, delisting, fraud, fundamental collapse), or
  (b) **a proven track record, not a one-off projection (tightened 2026-07-17 per user).** A single "this reallocation should recoup faster" argument is NOT enough on its own — it's too easy to talk myself into a plausible-sounding case. This exception requires a demonstrated track record of this specific kind of decision (sell a loser, reallocate, it actually recoups faster) working out in practice. No such track record exists yet, so exception (b) is currently dormant — do not invoke it until there's real, logged evidence to point to, not just reasoning in the moment.
- ZSPC's blanket exemption is under review — user is reconsidering it (2026-07-17) but hasn't given a final answer. **Do not sell or evaluate ZSPC for a loss-sale until the user confirms which way this goes.** Cost basis $35.20 (20 sh @ $1.76), current value ~$3.60 — remaining downside from here is small either way.
- On the upside, keep applying profit-taking using the technical discipline rule above (real resistance, not round numbers).
- IMUX status: both resting sells filled 2026-07-17 (1sh @ $15.65, 1sh @ $15.80, realized gain $13.45 vs $9.00 cost). 1 share remains held uncapped as a tail, no resting orders on it currently. Update this note as new ladders get placed on other positions.
- AHG status: bought 2026-07-20 (22sh @ $1.48 limit, filled 2026-07-21) via the general capital rotation rule below — sector diversification pick (Retail Trade/Internet Retail). Profit ladder placed 2026-07-23 (both GTC, both gains vs $1.48 cost): 11sh @ $1.58 undercutting a heavily-tested $1.60 rejection wall, 11sh @ $1.65 undercutting the higher $1.66–$1.69 shelf. **The $1.58 rung FILLED 2026-07-30** (11sh, in two executions — 8sh same-day 2026-07-23, remaining 3sh near close 2026-07-30 — realized gain ~$1.10 vs $1.48 cost), freeing ~$17.67 settled cash; this fill wasn't caught until 2026-07-31 due to the order-checking gotcha above, now fixed. 11 shares remain, with the $1.65 rung still resting (unfilled as of 2026-07-31). The freed ~$17.67 was NOT redeployed on 2026-07-31 — the coiled-spring scan that cycle only turned up a crashed biotech (BIOA, gapped down ~65% overnight, a falling knife not a setup), a thin/declining microcap in a different sector (AUC, Finance) that didn't look like a real quality setup, and a same-sector pick (SOGP, Technology Services — already-heavy sector) — none cleared the "real, quality" bar, so cash was held per judgment call rather than forced into a weak trade. Re-attempt redeploying this ~$17.67 on a future cycle when the scan turns up something better; update this note once it's deployed.

## Leveraged / inverse ETF sub-strategy (distinct — do not apply the hold-forever rule here)

Universe: leveraged/inverse ETPs identifiable by name ("2X Long/Short", "3X", "Daily Target", "Direxion", "T-REX", "GraniteShares", "Leverage Shares"). These decay structurally over time — do **not** hold through a drawdown hoping for recovery the way core holdings are held.

- Entry signal: run/adapt saved scan `c22bac0a-6e1f-4c98-9766-50d3888a4c86` WITHOUT excluding ETFs, looking for a coiled-spring pattern — flat/quiet price today (avoid anything already up double digits, that's already popped), relative volume 1.5x+, RSI 50-65 (bullish, not overbought). Real signal building, not already realized.
- On entry: place the buy, THEN immediately place a bracket — stop-loss and profit-target sized per the technical discipline rule (real support/resistance) rather than a flat percentage when chart data supports more precise levels. Default to stop ~8% below / target ~15% above only when there isn't enough chart history to derive real levels. Both as GTC resting orders placed right after the buy fills.
- Each cycle with an open leveraged-ETF position: if it's moved favorably, trail the stop upward (never lower) — cancel and replace at the new level. Exit early via judgment call if the thesis is clearly invalidated before the stop hits.
- Sizing: no fixed dollar cap — size to available settled cash for this one position. The real risk control here is structural: cap at ONE leveraged/inverse ETF position open at a time, since this category is the highest-risk bucket (decay + volatility) and shouldn't be stacked.
- This bracket approach explicitly overrides "never sell at a loss" — for this category only.

## General capital rotation (non-leveraged new positions)

- Check `buying_power` each cycle — that's the authoritative spendable figure (settled funds), not `cash` (which includes proceeds still settling, typically ~T+1 after a sale).
- Explicit user directive (2026-07-17): the account is small — don't sit on capital. **Deploy essentially all newly-settled cash** rather than holding back a 20-30% reserve. The goal is active cycling: buy a real, quality candidate, take profits per the technical discipline rule above when it runs, then immediately redeploy the proceeds into the next candidate. Grow the account through turnover, not by parking cash. "Not a bad buy just to spend cash" still applies — quality bar stays real, but bias is toward staying deployed, not toward holding reserve.
- Only skip forcing a trade when idle cash is trivially small (under ~$10) — genuinely immaterial either way.
- **No fixed dollar cap per position.** (Correction, 2026-07-17: an earlier version of this rule stated "$100 max, ~15% of account value" — that number was never something the user specified, it was invented unilaterally and mis-stated as agreed-upon. Removed.) Whether settled cash on a given cycle goes into a single position or is split across a few small ones is a judgment call each cycle, based on what's actually available and what candidates the scan turns up — a single cycle's buy doesn't need to be diversified on its own.
- **Portfolio-level diversification is the real, explicit goal (user-confirmed 2026-07-17):** across the account as a whole, don't concentrate everything into one place. If a scenario ever calls for deploying a large freed sum (e.g. after a full liquidation), do NOT put all of it into one single asset — spread it. Also: don't keep buying into the *same* asset cycle after cycle without a sound reason to add to that specific position again (a real thesis reason is fine — chasing habit isn't).
- **Diversification explicitly includes sector balance (user-confirmed 2026-07-17).** Context from the user: their main (non-agentic) account got over-concentrated in tech, and because tech and crypto tend to sell off together, a single bad day hits everything at once and has caused margin calls there — this agentic account has no margin so a margin call specifically can't happen here, but the underlying problem (a whole portfolio moving as one correlated block) is exactly what sector diversification is meant to prevent, and it applies here too. Before adding a new position: check the sector of existing holdings (`get_equity_fundamentals` returns `sector`) and avoid stacking further into a sector that's already heavily represented (current holdings already lean tech/software-adjacent: SOUN, NRDY, ZSPC) unless there's a sound, specific reason. Prefer a genuinely different sector/theme when the scan offers a reasonable choice.
- The one real, explicit hard constraint from the user: **don't risk the whole account (~$600-700 total) going to zero in one day.**
- When there's settled cash to deploy: run/update the saved scan for a coiled-spring candidate — flat/quiet price today, relative volume 1.5x+, RSI 50-65, price $0.50-$20, excluding SPAC trusts ("Acquisition Corp" names near $10 NAV) and ETFs (leveraged ETF plays are handled by the sub-strategy above) — want a real operating company. Also screen out a stock that's merely *drifted* into the RSI/relative-volume band via an overnight crash (check `get_equity_historicals` for a large gap-down before buying) — that's a falling knife, not a coiled spring, even though it can pass the raw filter values. Most recent non-leveraged add: AHG (Retail Trade/Internet Retail, added 2026-07-20, filled 2026-07-21) — update this note as new positions get added. ~$17.67 freed by the AHG $1.58 rung fill (2026-07-30) is still sitting undeployed as of 2026-07-31 pending a real candidate — see the AHG status note above.
- Use a limit order sized off `get_equity_quotes`. Fully autonomous — attempt `review_equity_order` + `place_equity_order` directly, no live confirmation needed.

## Sector relative-strength signal (probabilistic tilt, not prediction — added 2026-07-17)

User's idea, refined: use historical data across multiple lookback windows to gauge which sectors currently have the strongest/weakest momentum, and use that as a bias for capital decisions — NOT as a claim that it predicts the next move. Be honest with the user about this distinction if it ever comes up: this follows a trend a bit after it starts, it does not call tops/bottoms or foresee news.

- **Basket:** sector ETFs as proxies — XLK (tech), XLF (financials), XLE (energy), XLV (healthcare), XLP (staples), XLU (utilities), XLY (discretionary), XLI (industrials), XLB (materials), XLRE (real estate), XLC (communication services).
- **Data:** `get_equity_historicals` per symbol, daily bars, going back ~400 days (covers the 365-day window with buffer).
- **Compute:** % return over the last 30, 90, 180, and 365 days for each sector ETF. Composite score = weighted toward recent momentum, e.g. 40% (30d) + 30% (90d) + 20% (180d) + 10% (365d). Rank sectors strongest → weakest.
- **Refresh cadence:** NOT every 20-minute cycle — sector trends don't move that fast and it's wasteful to recompute constantly. Refresh once per day, or right before a capital-deployment decision, whichever comes first. Cache the ranking with the date it was computed.
- **Use 1 — new capital:** when the coiled-spring scan turns up a candidate, prefer one in a sector ranked in the stronger half of the list. This is a tiebreaker/bias on top of the existing scan criteria, not a replacement for them — and the sector-diversification rule (don't stack an already-heavy sector) still wins if the two conflict.
- **Use 2 — exit bias on existing winners:** if a held position's sector has fallen to the weak end of the ranking (broad sector deterioration, not just one stock dipping), that raises the priority to actually execute the resting profit-taking targets from the technical discipline rule rather than letting it ride for more — and can justify biasing a new sell target toward the nearer edge of the real resistance zone rather than the far edge, since the sector backdrop is turning less favorable. This does NOT authorize selling at a loss — the core holdings rule is untouched; it only affects how eagerly gains get locked in on winners.

## Crypto (added 2026-07-23 — currently unactionable, capability gap)

User gave the account crypto holdings (~$34.23 as of 2026-07-23, visible only as an aggregate `crypto_value` figure in `get_portfolio`). Standing rule from the user: **never sell crypto at a loss; otherwise use judgment** — same spirit as the core equity holdings rule.

**Real limitation, checked and confirmed 2026-07-23:** no crypto-specific tools exist in this session's Robinhood MCP toolset — no `get_crypto_positions`, no crypto quote tool, no crypto order-placement tool. `get_portfolio` shows the aggregate dollar value only; there's no way to see the coin, quantity, cost basis, current price, or place a trade. This was searched for directly (including the exact names hinted at in other tools' docs) and confirmed absent, not just overlooked.

Because of this, the rule above is currently **dormant/unactionable** — do not fabricate a position, price, or action. Each cycle, note in passing whether crypto tools have appeared (a quick ToolSearch for crypto-related names costs little); the moment they do, start checking crypto holdings alongside equities each cycle and apply this same never-sell-at-a-loss rule to them.

## Hard boundaries

No margin, no options, no new capital/deposits (account isn't enabled for these anyway, but never attempt regardless).

## If the account's permission classifier blocks a trade

Do not fabricate or assume the user's consent to route around it, even though standing authorization has been given — that authorization is real, but if the classifier still blocks a specific trade, don't manufacture a workaround. Surface it: call `PushNotification` (status "proactive", one line, under 200 chars: symbol/qty/price/reason) and state it plainly in-session text. Say directly that the block is still happening rather than silently dropping the trade or pretending it succeeded.
