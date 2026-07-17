# DLMM Use-Case Playbooks

Read this when the user has a stated intent — "I want to LP SOL/USDC", "DCA out of my bag", "farm rewards", "seed a launch" — and you need a concrete, doc-backed recipe rather than general theory. Find the playbook matching their intent; each one gives the pool / bin-step / shape / range / side to use, the rules for managing it, and how to exit plus the risks that kill it.

Two rules override every recipe below:

- **All fee expectations here are NET of the protocol cut.** The protocol skims its share off the total swap fee *before* LPs receive anything: 10% on standard DLMM pools (you keep 90%), 20% on Launch Pools (you keep 80%). Any APR you quote or model must already have this subtracted — see "Model true net LP fee yield" and `fees-and-economics.md`.
- **Always fetch live pool data before committing capital.** These recipes tell you *what shape* to use; only the live pool — bin step, volume, `fee_tvl_ratio["24h"]`, `has_farm`, `collect_fee_mode`, function mode, Token-2022 status — tells you *whether this pool is worth it*. Pull it first; see the data-driven playbooks at the end and `data-api.md`.

**Uniform template.** Every playbook uses the same five parts so you can lift a whole recipe into a conversation:

- **Name** — the intent it serves.
- **When** — the market view or user goal that selects this playbook.
- **Setup** — pool, bin step, shape, range, side, and any pool-mode prerequisite.
- **Management** — what to watch and what action to take while the position is live.
- **Exit & risks** — how you get out and what can go wrong.

Two hard limits several playbooks lean on, worth memorizing: the total swap fee is capped at 10% (`MAX_FEE_RATE`, stored in 1e9 precision) no matter how high volatility drives the variable fee; and a position spans a default 70-bin layout, up to 1,400 bins via direct min/max entry, growable by at most 91 added bins per resize instruction.

---

## Playbook: Stable / pegged-pair LP

- **When:** Both tokens track the same value (USDC/USDT, an LST vs SOL, any pegged pair) and you expect price to sit at or very near the peg. Goal: harvest a high volume of tiny swaps with minimal directional risk.
- **Setup:**
  - Smallest bin step available (1–5 bps) so each bin captures the sub-percent wiggle around the peg.
  - Shape: **Curve** or **Spot-Concentrated (1–3 bins)** parked tight on the peg — this puts the maximum share of capital exactly where the trading happens, the whole point on a pair that barely moves.
  - Balanced (two-sided) deposit at the active bin.
- **Management:**
  - Watch the peg, not the calendar. While the active bin stays inside your 1–3 bins you can leave it for days.
  - If the peg starts to drift, do **not** close and reopen — resize the position wider *in place* (increase length on the drifting side) so your liquidity keeps straddling the live price and keeps earning through the wobble.
- **Exit & risks:**
  - Highest out-of-range risk of any playbook if the peg *breaks* — a Spot-Concentrated position can go fully inactive in one candle, leaving you holding 100% of the depegging side (realized IL).
  - A depeg is exactly when you must decide fast: widen to stay active and average through it, or withdraw and eat the loss.
  - Cross-link: `positions-orders-and-rewards.md` for resize mechanics.

---

## Playbook: Volatile-pair LP

- **When:** A mid-cap or blue-chip pair (SOL/USDC, mid-cap vs SOL) that moves several percent a day, and you want fee capture without going inactive every few hours.
- **Setup:**
  - Larger bin step (25–80 bps) so a reasonable bin count covers a wide price band.
  - Shape: **Spot-Spread (20–30 bins)** for balanced coverage with breathing room, or **Spot-Wide (~50 bins)** if you would rather rebalance rarely. Use **Bid-Ask** when you specifically want to earn most on swings out to the edges (see the volatility-capture playbook).
  - Optional pattern: open wide to survive the first big move, then decrease length with a shrink mode to trim empty bins once volatility cools and concentrate capital back near price.
- **Management:**
  - Check that the active bin is still inside your range and roughly centered.
  - On a strong trend a Spot-Spread can still exit range — resize or re-center rather than letting it sit dead.
  - When the market calms, shrink empty outer bins (ShrinkBoth) to lift fee-per-dollar.
- **Exit & risks:**
  - IL is the main cost — concentrated liquidity amplifies it versus a full-range AMM.
  - Each rebalance is a transaction carrying non-refundable bin-array rent for any new bins you touch (refundable position rent comes back on close).
  - Frequent rebalancing on an oscillating pair locks in IL repeatedly; only re-center when the move looks durable, not on noise.

---

## Playbook: Capital-efficient tight vs resilient wide (the core tradeoff)

- **When:** The user asks "should I go narrow or wide?" This is the decision underneath most other playbooks, so make the tradeoff explicit rather than defaulting.
- **Setup:**
  - **Tight** = Spot-Concentrated (1–3 bins) or Curve: maximum fee-per-dollar because nearly all capital sits at the active price.
  - **Wide** = Spot-Wide (~50 bins): durable coverage that survives big moves.
  - **Middle** = Spot-Spread (20–30 bins).
- **Management:**
  - Tight positions earn more per dollar *while in range* but go inactive fast and demand frequent rebalancing (and its rent).
  - Wide positions rarely go out of range and need little attention, but each dollar is spread thinner so fee-per-dollar is lower.
  - Match the pole to how much active management the user will actually do.
- **Exit & risks:**
  - The failure mode of "tight" is going out of range — missed fees plus locked IL.
  - The failure mode of "wide" is disappointing yield that underperforms a tighter competitor in the same pool group.
  - No free lunch: narrower earns more and dies faster. Cross-link: SKILL.md (Core Concepts -> Liquidity shapes) for the shape definitions.

---

## Playbook: DCA INTO a token (accumulate as price falls)

- **When:** You want to buy a base token gradually at progressively lower prices instead of a single market buy.
- **Setup:**
  - Single-sided **quote** token only — toggle **Auto-Fill OFF** and enter an amount in just the quote field.
  - Place it in **Bid-Ask** (or hand-selected bins) laddered **below** the current active price.
  - As price falls through each lower bin, quote converts into the base token at that bin's fixed price — a mechanical buy-the-dip ladder.
- **Management:**
  - Watch how far price has descended into your ladder; the lower bins fill last and hold the most quote.
  - If your thesis changes you can pull the unfilled bins.
  - Constraint: you **cannot withdraw single-sided from the active bin** — once price sits in one of your bins you must take both tokens out of that active bin.
- **Exit & risks:**
  - If price never drops, nothing fills and you simply hold quote (opportunity cost, not loss).
  - If price craters through your whole ladder you end fully converted to a base token now cheaper than your average fill — real IL / directional risk.
  - Alternative: native **bid limit orders** (see the limit-order playbook) if the pool supports them, for cleaner per-bin fill tracking. Cross-link: `positions-orders-and-rewards.md`.

---

## Playbook: DCA OUT of a token (sell as price rises)

- **When:** You hold a base token and want to sell it gradually into strength rather than dumping at one price.
- **Setup:**
  - Single-sided **base** token only — Auto-Fill OFF, enter only the base amount.
  - Place it in **Bid-Ask** laddered **above** the current active price.
  - As price rises through each higher bin, base converts to quote at that bin's price — a mechanical take-profit ladder.
- **Management:**
  - Track how far price has climbed into the ladder; higher bins fill last.
  - Worked example (from the docs): you bought SOL around 236.6 USDC and want to sell between 240 and 280 USDC — turn Auto-Fill off, enter only SOL, set the range 240–280, choose **Bid-Ask**, and SOL spreads across that band and progressively swaps to USDC as price rises through it.
- **Exit & risks:**
  - If price never rises, nothing sells and you keep the base token.
  - If price spikes past 280 and keeps going you are fully sold and miss upside above your top bin (the classic "sold too early" outcome).
  - Same active-bin single-sided withdrawal constraint applies. Alternative: laddered native **ask limit orders**.

---

## Playbook: Native limit orders (buy-the-dip bid / take-profit ask)

- **When:** You want true on-chain limit orders — discrete fills at chosen prices with clean per-order accounting — rather than the "each bin is like a limit order" LP metaphor.
- **Setup:**
  - **Requires a pool whose function mode supports limit orders.** This is mutually exclusive (XOR) with liquidity mining — a pool is EITHER LM/farm OR limit-order, never both. Confirm the mode first.
  - **Bid**: place token Y at or below the active bin to buy token X when price reaches those bins (`is_ask_side = false`).
  - **Ask**: place token X at or above the active bin to sell X for Y (`is_ask_side = true`).
  - Up to **50 bins per order**. Place by absolute bin ID or relative offset from the observed active bin; relative placement carries a max active-bin slippage guard so the order won't create if the live active bin has moved too far.
- **Management:**
  - Not guaranteed to execute — an order fills only if swap flow reaches the bin and consumes the liquidity, so it may sit unfilled or partially fill.
  - Monitor `filled_pct` and `nearest_unfilled_bin_price` (distance from current price).
  - While an order's liquidity sits in bins it earns a bonus: the program allocates **50% of the limit-order portion of the fee** to limit-order participants.
  - Cancel orders that no longer match your view — a cancel pays out accrued bonus plus the unfilled remainder immediately.
- **Exit & risks:**
  - Never-reached orders are dead capital until cancelled.
  - This is a distinct economic path from the Bid-Ask LP metaphor — do not conflate them.
  - Cross-link: `positions-orders-and-rewards.md` and `data-api.md` (the `/wallets/{wallet}/limit_orders/...` endpoints track live orders and realized bonus).

---

## Playbook: Volatility capture

- **When:** You expect price to swing hard and want to earn the most precisely on those swings out to the extremes, whether on a volatile pair or a wobbling pegged pair.
- **Setup:**
  - **Bid-Ask** shape — it concentrates liquidity at the *edges* of your range (the inverse of Curve), so the big fees land when price lurches out to a boundary bin.
  - Size the range around where you expect the swings to reach.
  - A larger bin step lets the variable fee escalate faster — the variable fee scales with the square of both volatility and bin step.
- **Management:**
  - Bid-Ask sits relatively idle near the active price and only lights up when the market moves into the edge bins — that's the design, not a bug.
  - Watch for price actually reaching your edges; that's when you earn most and also approach the end of your range.
- **Exit & risks:**
  - More advanced to manage than Spot.
  - If price blows straight through an edge you go inactive on that side and hold the converted token; if price never swings you underearn versus a centered shape.
  - Dynamic fees offset IL during the swing but cannot eliminate it.

---

## Playbook: In-place dynamic resizing / migrate tight <-> wide

- **When:** The market regime changed and you want to move a live position between a tight high-efficiency range and a wide defensive one *without* closing and reopening (which churns rent and can lock IL).
- **Setup:**
  - Any existing Dynamic Position.
  - **Increase length** to add bins on either side — new allocation beyond the default 70-bin layout is capped at **91 added bins per resize instruction**, so a big widen may take several.
  - **Decrease length** to remove *empty* bins.
  - The **rebalance** flow combines claim + remove + resize + add-liquidity into a single position-management instruction.
- **Management:**
  - Choose a shrink mode to control how aggressively rebalance trims: **ShrinkBoth** (trim empties both sides), **NoShrinkLeft** (keep lower, trim upper), **NoShrinkRight** (keep upper, trim lower), **NoShrinkBoth** (keep both).
  - Typical moves: a stable position that started tight around a peg widens as the pair drifts; a volatile position that started wide shrinks back to tight once volatility cools.
- **Exit & risks:**
  - Resizing avoids close-and-reopen churn and avoids IL-locking, but it is not free — the cost is non-refundable rent for **new bin arrays only** (existing arrays and refundable position rent are unaffected).
  - Resizing does not remove IL or out-of-range risk; it just makes moving the range cheaper. Cross-link: `positions-orders-and-rewards.md`.

---

## Playbook: Liquidity-mining reward farming

- **When:** You want token rewards on top of swap fees, and there's a farm-enabled pool for your pair.
- **Setup:**
  - Pick a pool with `has_farm = true` (check `farm_apr` / `farm_apy`).
  - Liquidity mining is a pool function mode **mutually exclusive with limit orders** — an LM pool cannot host limit orders and vice versa.
  - A pool tracks up to **2 reward tokens**, emitted at a fixed rate over a set period (docs example: 50,000 USDC over 4 weeks ≈ 1,785.71 USDC/day).
  - Place liquidity so it stays in or very near the active bin — that's the only liquidity that earns.
- **Management:**
  - Rewards accrue only to in-range / active-bin liquidity — a v2 swap splits elapsed rewards across at most **15 bins** it crosses, and a position out of range or a bin with no liquidity earns nothing (an empty active bin pays no rewards for that period).
  - Your reward share = your liquidity in eligible bins ÷ total liquidity in eligible bins, so more TVL crowding your bin dilutes your cut.
  - Rewards **do not auto-compound** and must be claimed manually.
  - Rebalance to stay eligible; drifting out of range silently zeroes your farm yield.
- **Exit & risks:**
  - Fixed emission means reward APR falls as more TVL piles in.
  - Upside is real: rewards can flip a position that is fee-negative (fees < IL) into net positive. Model fees and rewards together, net of the protocol cut.
  - Cross-link: `data-api.md` (`unclaimedRewardTokenX/Y` on the pnl endpoint, `/total_claims` for realized).

---

## Playbook: Token launch / memecoin LP

- **When:** Seeding or LPing a brand-new token through price discovery. Highest fee potential and highest chance of near-total loss — only disposable capital.
- **Setup:**
  - Pre-activation seeding uses the LFG curve over a min/max price with a **curvature** factor 0–1 (lower = more concentrated), or single-bin seeding to park all base token at one price.
  - High bin step (up to the **400 bps** program cap) with **Spot** or **Bid-Ask** over a broad discovery range; single-sided is possible since projects can bootstrap with only their token.
  - Before depositing, sync the pool price with Jupiter's reference — a fresh low-liquidity pool's price often doesn't match market, and there's a "Sync with Jupiter's price" action for it.
- **Management:**
  - Account for the **doubled 20% protocol cut** on Launch Pools (you keep 80%, not 90%) and the 10% total-fee cap.
  - Know the controls that can bite: `lockReleasePoint` lockups prevent withdrawal until they release; `creatorPoolOnOffControl` lets the creator pause trading; an Alpha Vault (FCFS caps vs Prorata buying cap, with vesting windows) governs pre-activation deposits; seeding only works *before* the activation point (slot or timestamp).
  - After discovery settles, rebalance/resize around the new market level.
- **Exit & risks:**
  - The token can drop 80%+ from launch price — plan to be fully converted to it.
  - Non-refundable bin-array rent on a wide launch range is a real sunk cost.
  - Dynamic fees during the opening volatility spike can be large but are never guaranteed. Cross-link: `launch-pools-and-terminal.md`.

---

## Playbook: Model true net LP fee yield

- **When:** The user quotes or asks about an APR, or you're comparing pools on fee yield. Never repeat a gross number as if the LP keeps it.
- **Setup:**
  - Net fee = gross swap fee × LP share, where LP share is **90%** on standard pools and **80%** on Launch Pools.
  - The pool object's `apr` / `apy` are 24h scalars; treat them as inputs to sanity-check, not gospel.
- **Management:**
  - Worked adjustment — forget the cut and you overstate real yield by 10% of the figure on a standard pool (a 50% gross-fee APR is ~45% net) and by 20% on a Launch Pool (50% gross → 40% net).
  - Always state the number you give as net, and say which share you applied.
- **Exit & risks:**
  - The bigger error is treating past fee APR as a promise — fee earnings depend on volume through *your* bins, and volume can dry up.
  - Present net yield as conditional on the pool staying active and price staying in range. Cross-link: `fees-and-economics.md`.

---

## Playbook: Avoid composition fees on entry / rebalance

- **When:** Depositing or rebalancing at or across the active price, and you want to avoid a silent extra cost.
- **Setup:**
  - A **composition fee** is charged only when you add liquidity **to the active bin** in a token mix that differs from that bin's current X:Y ratio — because an off-ratio deposit is effectively forcing a mini-swap.
  - There is **no** composition fee when you deposit into an empty bin or any non-active bin.
- **Management:**
  - To avoid it, either match the active bin's current X:Y ratio when depositing there (Auto-Fill helps by filling the equivalent other-token amount at the current rate), or add your liquidity outside the active bin entirely.
  - This matters most on rebalances that re-center onto the live price.
- **Exit & risks:**
  - Ignoring it means a real, often-overlooked haircut on every off-ratio active-bin deposit, on top of the swap-fee math. Cross-link: `fees-and-economics.md`.

---

## Playbook: Pick a pool by fee-token (Collect Fee Mode)

- **When:** You care which token your fees accumulate in — e.g. you want to build a predictable quote (USDC/SOL) balance.
- **Setup:**
  - Collect Fee Mode is a **pool-level** setting fixed at creation; an LP cannot change it.
  - **OnlyY (value 1)** collects fees in token Y (the quote) regardless of swap direction — pick this to accumulate quote predictably.
  - **InputOnly (value 0)** collects fees in whichever token enters the swap, so you earn both sides over time depending on flow.
- **Management:**
  - Read `collect_fee_mode` on the pool object before entering if fee-token denomination matters to your accounting.
  - On limit-order pools this also determines which token your order bonus arrives in.
- **Exit & risks:**
  - Collect Fee Mode does **not** remove market risk — choosing OnlyY does not protect you from IL or from ending up with a different token composition after swaps pass through your bins.
  - It only shapes the fee-token denomination. Cross-link: `fees-and-economics.md`.

---

## Playbook: Delegated / managed LP

- **When:** A bot, team, or manager needs to run positions on someone else's behalf, or you want fees routed to a separate address.
- **Setup:**
  - Assign an **operator** to a position to let another address perform position-management actions (modify liquidity, rebalance).
  - Set a separate **fee owner** to control where claimed fees are routed.
  - Optionally flip permission bits to make specific operations (e.g. fee claiming) **permissionless** so anyone can trigger them for that position.
- **Management:**
  - Useful for automated strategies and multi-person desks — the operator manages, the fee owner collects.
  - Keep the roles scoped to what each actor actually needs.
- **Exit & risks:**
  - Only delegate operator rights to addresses you trust: an operator can perform any position-management action the program authorizes, which includes moving your liquidity.
  - Revoke or reassign if trust changes. Cross-link: `positions-orders-and-rewards.md`.

---

## Playbook: Token-2022 pair LPing

- **When:** One side of the pair is an SPL Token-2022 mint. Extra due diligence before committing.
- **Setup:**
  - Confirm the mint's extensions are DLMM-safe. Permissionless-supported extensions include `TransferFeeConfig`, `MetadataPointer`, `TokenMetadata`, and `TransferHook` **only when both the hook program and hook authority are revoked**; anything more sensitive needs a token badge review.
  - Prefer wrapped SOL over the native Token-2022 mint (DLMM rejects the native Token-2022 mint outright to avoid fragmenting SOL liquidity).
- **Management:**
  - On a transfer-fee token, the fee leaks from your realized amounts on both principal and fees — every deposit, withdrawal, and fee claim arrives smaller than the nominal figure, so factor that into yield math.
  - Avoid mints with a live **freeze authority** (DLMM won't permissionlessly accept mints that can freeze accounts) — a freeze can trap your position.
- **Exit & risks:**
  - The freeze-authority and native-mint cases are hard stops, not preferences.
  - Transfer-fee drag compounds over many claims. Cross-link: `launch-pools-and-terminal.md` and `positions-orders-and-rewards.md`.

---

## Playbook: Fast entry / exit via Dynamic Terminal

- **When:** You want to move quickly on the meteora.ag UI — enter in one transaction, exit to a single token, or batch-manage many positions.
- **Setup:**
  - **Ape In** swaps and creates a position from a single token in one transaction, using your preset settings — for seizing an opportunity without pre-splitting funds.
- **Management:**
  - **Zap Out** withdraws and converts everything to one preferred token in a single step, but only for positions with **≤ 250 bins**.
  - **Withdraw and Close** exits fully and reclaims the refundable rent.
  - Batch actions **Claim All Fees** and **Close All Positions** collect or exit across every position at once.
- **Exit & risks:**
  - Zap Out is unavailable on very wide (>250-bin) positions — those must be removed the normal way.
  - Closing reclaims refundable position/extension rent but never the non-refundable bin-array rent. Cross-link: `positions-orders-and-rewards.md`.

---

## Playbook: TA-driven range placement

- **When:** You want to anchor a position's boundaries to real chart structure rather than an arbitrary ± percentage.
- **Setup:**
  - Use the terminal's integrated TradingView chart.
  - Anchor **min price to support**, **max price to resistance**, and use **Fibonacci retracements** to find reversion zones for range-bound strategies.
  - Enter during consolidation near your target range — tighter price action means more time in range.
- **Management:**
  - Check momentum before committing to a concentrated range.
  - In high-volatility periods wait for action to settle, and set **wider ranges** to stay in range longer.
  - Watch for breakouts that push price out of your range and decide rebalance/hold/exit from what the chart shows.
- **Exit & risks:**
  - Avoid entering an already-extended range likely to revert against you, and exit before a clear trend pushes price far outside the position (that's when IL runs).
  - TA improves placement odds; it doesn't remove out-of-range risk.

---

## Playbook: Data-driven hold-vs-claim-vs-close & rebalance/exit review

- **When:** A position is live and the user asks "should I hold, claim, rebalance, or close?" Decide from data, not vibes.
- **Setup:**
  - Pull `GET /positions/{pool_address}/pnl?user=<wallet>` — path is the **pool** address, `user` is **required**, `status = open|closed|all`. It returns every position the user holds in that pool.
  - Also pull `GET /portfolio/open?user=<wallet>` for the pool-level `outOfRange` flag and `positionsOutOfRange[]`.
- **Management:**
  - Compare `allTimeFees` + unclaimed rewards (`unrealizedPnl.unclaimedRewardTokenX/Y`) against IL, and read overall `pnlUsd` — if fees plus rewards beat IL, the position is net positive and probably a hold.
  - Read `feePerTvl24h` to see whether it's still earning, and `isOutOfRange`.
  - Check `poolActiveBinId` / `poolActivePrice` against the position's `[lowerBinId, upperBinId]` to see exactly where the market sits relative to your range.
  - Live unclaimed fees are `unrealizedPnl.unclaimedFeeTokenX` / `unclaimedFeeTokenY`.
- **Exit & risks:**
  - If the active bin has left the range and price is trending away, rebalancing locks IL — weigh waiting for a return versus re-centering.
  - There is **no** `/positions/{address}/total_claim_fees` and **no** `/wallets/{wallet}/open_positions` endpoint; use the endpoints above. Cross-link: `data-api.md`.

---

## Playbook: Portfolio-wide review & claim accounting

- **When:** The user wants a wallet-level picture across all their DLMM positions, or an audit of what they've actually claimed.
- **Setup:**
  - `GET /portfolio/total?user=<wallet>` for all-time aggregate PnL.
  - `GET /portfolio/open?user=<wallet>` for the pools where they hold open positions (each item carries `listPositions[]` and `positionsOutOfRange[]`).
  - `GET /portfolio?user=<wallet>` for pools with **closed** positions — pass explicit `page_size` / `days_back`, since the doc's defaults are internally inconsistent.
- **Management:**
  - Drill from a portfolio pool into `GET /positions/{pool_address}/pnl?user=<wallet>&status=closed` for per-position history.
  - Use `GET /positions/{address}/historical` (path is the **position** address here) to reconstruct the add / remove / claim_fee / claim_reward timeline.
  - For realized fee + reward claims in one pool, use `GET /wallets/{wallet}/pools/{pool_address}/total_claims`.
- **Exit & risks:**
  - Don't invent cursor pagination — the API is page / page_size only.
  - When comparing pools, read `volume["24h"]`, `fees["24h"]`, `fee_tvl_ratio["24h"]` off the `TimeWindowData` objects (there are no `trade_volume_24h` / `fees_24h` scalars), plus `apr` / `apy` and `farm_apr` / `farm_apy`. Cross-link: `data-api.md`.

---

## Before you commit any playbook

Two checks decide which playbooks are even available on a given pool, both set at pool creation and unchangeable by an LP:

1. **Pool function mode is LM XOR Limit Order.** A farm-enabled pool cannot host limit orders; a limit-order pool has no farm rewards. So the "reward farming" and "native limit orders" playbooks are mutually exclusive on the same pool — check `has_farm` and the function mode before promising either.
2. **Token-2022 status.** If either mint is Token-2022, run the safe-extension / freeze-authority / transfer-fee checks first.

Each playbook links back to the reference that carries the parameter-level detail (SKILL.md Core Concepts, `fees-and-economics.md`, `positions-orders-and-rewards.md`, `launch-pools-and-terminal.md`, `data-api.md`) — read that file when the user needs the exact constants, formulas, or endpoint shapes behind the recipe.
