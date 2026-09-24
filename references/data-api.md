# DLMM Data API Reference

Read this when you need exact endpoints, required vs. optional params, pagination rules, or the real response-field names to build a precise call — and to answer from actual pool, position, and portfolio numbers instead of hypotheticals. Everything here is the read-only indexed Data API; the protocol math lives elsewhere: bin-price formula and dynamic-fee decay in `references/fees-and-economics.md`, resize caps in `references/positions-orders-and-rewards.md`.

## Global facts

- **Production base URL:** `https://dlmm.datapi.meteora.ag`
- **Development base URL:** `https://dlmm.dev.metdev.io` (same schema; switch via the Swagger server dropdown)
- **Swagger UI:** `https://dlmm.datapi.meteora.ag/swagger-ui/` (dev: `https://dlmm.dev.metdev.io/swagger-ui/`)
- **Spec:** OpenAPI 3.1.0, title "DLMM API", version 0.1.0.
- **Auth:** none (`security: []`) — public, no key. Use WebFetch, curl, or any HTTP tool.
- **Rate limit:** 30 requests/second shared across ALL DLMM endpoints — budget your calls, especially when fanning out across a wallet's pools.
- **Errors:** every non-200 (typically `400`) returns `ErrorResponse { message: string }`. If a call comes back with `message`, read it — it usually names the bad param.

Fetch proactively. If the user gives you a token, pool address, or wallet, pull real data before advising rather than reasoning from hypotheticals. Build URLs like `https://dlmm.datapi.meteora.ag/pools?query=SOL`. Example calls you'll reach for constantly:

- Best SOL/USDC-style pools: `GET /pools?query=SOL&sort_by=fee_tvl_ratio_24h:desc&filter_by=is_blacklisted=false%20%26%26%20volume_24h>=50000`
- One pool's live state: `GET /pools/5hbf9JP8k5zdrZp9pokPypFQoBse5mGCmW6nqodurGcd`
- A user's positions in a pool: `GET /positions/{pool_address}/pnl?user={wallet}&status=open`
- A wallet's open-position pools: `GET /portfolio/open?user={wallet}&page=1&page_size=50`

## DELETED ENDPOINTS — do not call these (they 404)

Three endpoints that older versions of this skill referenced **do not exist anywhere in the API**. Never emit them; use the replacement instead.

| Non-existent endpoint | What to call instead |
|---|---|
| `GET /wallets/{wallet}/open_positions` | `GET /portfolio/open?user=<wallet>` — pools with open positions; each `PoolOpenPortfolioItem.listPositions[]` holds the open position addresses. |
| `GET /wallets/{wallet}/closed_positions` | `GET /portfolio?user=<wallet>` (pools with closed positions), then drill into `GET /positions/{pool_address}/pnl?user=<wallet>&status=closed`. There is NO cursor pagination anywhere in this API. |
| `GET /positions/{address}/total_claim_fees` | Per-position total fees = `allTimeFees` on `GET /positions/{pool_address}/pnl`; live unclaimed = `unrealizedPnl.unclaimedFeeTokenX/Y` on the same call; wallet-level realized claims = `GET /wallets/{wallet}/pools/{pool_address}/total_claims`. |

## Shared list-query grammar

The `/pools` list endpoint accepts:

- `page` — 1-based page number.
- `page_size` — **pool lists max 1000; group lists max 100.**
- `query` — free-text search by name, tokens, or address.
- `sort_by` — `<field>:<dir>` (non-windowed) or `<metric>_<window>:<dir>` (time-windowed). `dir` is `asc|desc`. **Default `volume_24h:desc`.**
  - Time-windowed metrics: `volume_*`, `fee_*`, `fee_tvl_ratio_*`, `apr_*`.
  - Non-windowed fields: `tvl`, `fee_pct`, `bin_step`, `pool_created_at`, `farm_apy`.
- `filter_by` — `<field><op><value>` joined by `&&` (logical AND; whitespace around operators ignored).
  - Numeric fields (`tvl`, `volume_*`, `fee_*`, `fee_tvl_ratio_*`, `apr_*`): operators `= > >= < <=`.
  - Boolean `is_blacklisted`: `=true` / `=false`.
  - Text fields (`pool_address`, `name`, `token_x`, `token_y`): exact `=<value>` or multi-value OR `=[a|b|c]`.
- **Time windows:** `5m 30m 1h 2h 4h 12h 24h`. `5m` is valid for sort/filter but is NOT a guaranteed key in `TimeWindowData` response objects (which carry `30m 1h 2h 4h 12h 24h` as required keys).

## Pool discovery

| Endpoint | Purpose |
|---|---|
| `GET /pools` | Paginated pool list. Params: `page`, `page_size` (1–1000), `query`, `sort_by`, `filter_by`. Returns `{ total, pages, current_page, page_size, data[] }`. |
| `GET /pools/{address}` | Single pool, full `PoolResponse`. No query params. |

**Docs-only, not live:** the docs also describe `GET /pools/groups` and `GET /pools/groups/{lexical_order_mints}`, but the deployed API does not serve them — the router parses `groups` as a pool address and returns `invalid_pubkey`. When docs and API disagree, trust the live OpenAPI spec at `GET /api-docs/openapi.json` (which also exposes live-only `/stats/daily/volume`, `/stats/daily/trading_fees`, `/stats/daily/protocol_fees`).

Recipe for "which pool for pair X?": call `/pools?filter_by=token_x=<mintA> %26%26 token_y=<mintB>&sort_by=fee_tvl_ratio_24h:desc` and repeat with the mints swapped (pairs exist in both orientations), or use `query=<symbol>` and filter client-side. Then compare each pool's `volume["24h"]`, `fees["24h"]`, `fee_tvl_ratio["24h"]`, `apr`, and `pool_config.bin_step`. Field-tested `filter_by` behavior on the live API: text filters match **exact mint addresses only** (`token_x=<mint>` works; `name=SOL-USDC` and symbol values return 0 rows), and the documented multi-value OR syntax `=[a|b]` has been observed returning 0 rows — prefer two separate calls. One more field-tested note: bare `urllib`-style clients can get HTTP 403 (user-agent filtering) — `curl` works.

## Pool metrics shape (READ THIS — the field names people get wrong)

On the pool object, `volume`, `fees`, `fee_tvl_ratio`, and `protocol_fees` are **`TimeWindowData` objects keyed by `30m/1h/2h/4h/12h/24h`** — not scalars. Read `volume["24h"]`, `fee_tvl_ratio["24h"]`, `fees["24h"]`.

**`fees` is the LP share, already net of the protocol cut** — `fees + protocol_fees = volume × fee rate`, so `fee_tvl_ratio` (= `fees / tvl`) and `apr` are net too. Never apply the 90%/80% LP share to these fields.

There is **NO `trade_volume_24h` and NO `fees_24h` field** — those names do not exist. When you quote a fee/TVL ratio, always name its window, e.g. `fee_tvl_ratio["24h"] = 0.8%`, because the same object also carries `fee_tvl_ratio["1h"]` etc.

`apr` and `apy` ARE scalars — both are 24-hour figures. `farm_apr`/`farm_apy` (LM rewards) exist and are scalars too.

Full `PoolResponse` top-level fields (this is the canonical pool shape, also returned by `/pools/{address}` and inside group listings):

- `address` — pool (LB pair) address; `name` — pair name.
- `token_x`, `token_y` — `TokenMetrics` objects (see below).
- `token_x_amount`, `token_y_amount` — token amounts the pair holds (double).
- `reserve_x`, `reserve_y` — reserve *account addresses* (strings), NOT amounts.
- `created_at` — pool creation unix timestamp (int64).
- `reward_mint_x`, `reward_mint_y` — farming reward mint addresses.
- `pool_config` — `PoolConfig` (fees + bin step + fee mode).
- `dynamic_fee_pct` — current rate = base fee + variable fee.
- `tvl`, `current_price` — doubles.
- `apr`, `apy` — 24-hour scalars.
- `has_farm` (bool), `farm_apr`, `farm_apy` — LM reward scalars.
- `volume`, `fees`, `protocol_fees`, `fee_tvl_ratio` — all four are `TimeWindowData` objects.
- `cumulative_metrics` — `{ volume, fees }`, all-time doubles.
- `is_blacklisted` (bool), `tags[]` (string array).
- `launchpad` (optional, string|null) — identifies launch-pool origin.

### PoolConfig (fee + bin config — LP-critical)

- `bin_step` — bin step in basis points (int32).
- `base_fee_pct` — base fee rate (double).
- `max_fee_pct` — the cap the dynamic fee cannot exceed.
- `protocol_fee_pct` — the protocol's cut skimmed from the trade fee. LPs do NOT keep 100% of the swap fee; net LP fee is after this cut (10% standard pools, 20% launch pools — see mechanics).
- `collect_fee_mode` — `0 = InputOnly` (fees flow to the deposited/input token), `1 = OnlyY` (fees always paid in token Y). Affects which token you accumulate.
- Relationship: `dynamic_fee_pct` = `base_fee_pct` + variable fee, bounded above by `max_fee_pct`.

### TimeWindowData (shape of volume/fees/protocol_fees/fee_tvl_ratio)

A single object whose keys are windows and whose values are doubles: `30m`, `1h`, `2h`, `4h`, `12h`, `24h` are required; `5m` may appear. So `fee_tvl_ratio` is per-window percentages — `fee_tvl_ratio["24h"]` and `fee_tvl_ratio["1h"]` are different numbers. Any single figure you cite must name its window.

### TokenMetrics (per-token due diligence)

`address`, `name`, `symbol`, `decimals`, plus the pre-deposit safety screen:

- `is_verified` (bool) — token verification status.
- `freeze_authority_disabled` (bool) — rug-safety signal (freeze authority renounced).
- `holders` (int) — holder count; thin holder base is a risk flag.
- `total_supply` (double), `price` (USD double), `market_cap` (double).

Combine with pool-level `is_blacklisted`, `tags[]`, and `launchpad` before advising anyone to deposit into a thin or unverified token.

## Pool analytics (timing entries/exits)

| Endpoint | Params | Response per bucket |
|---|---|---|
| `GET /pools/{address}/ohlcv` | `timeframe` (`5m 30m 1h 2h 4h 12h 24h`, default `24h`), `start_time`, `end_time` (unix seconds, inclusive) | `timestamp`, `timestamp_str`, `open`, `high`, `low`, `close`, `volume` |
| `GET /pools/{address}/volume/history` | same params | `timestamp`, `timestamp_str`, `volume`, `fees`, `protocol_fees` |

Range rules: both bounds given → `[start,end]`; one given → the other inferred from `timeframe`; neither → default range from `timeframe`. Use OHLCV to read realized price range/volatility before recommending range width or a rebalance; use volume/history to spot a trend (rising volume → healthier fee outlook; falling → consider exit). `protocol_fees` is broken out separately and `fees` is already LP-net: LP-realizable fee = `fees`, gross = `fees + protocol_fees`. For launch-spike detection, pull short windows (`5m`/`30m`/`1h`) — the launch fee/volume spike is invisible in the 24h number.

## Position P&L — `GET /positions/{pool_address}/pnl`

**The path param is the POOL address, not a position address, and `user` is a REQUIRED query param.** Passing a position address or omitting `user` fails or returns nothing. The call returns ALL of that user's positions in the pool.

**Params:** `pool_address` (path); `user` (query, **required**); `status` (`open|closed|all`, default `all`); `page` (default 1); `page_size` (min 1, **max 100**, default 20).

**Response `GetPoolPositionPnLResponse`:** `totalCount`, `page`, `pageSize`, `hasNext`, `tokenX`/`tokenY`, `tokenXPrice`/`tokenYPrice`, `rewardTokenX`/`rewardTokenY` (+ prices), `solPrice?`, `positions[]`.

**`PositionPnLData` per position (this is rebalancing gold — one call answers hold/claim/rebalance/exit):**
- `positionAddress`.
- `[lowerBinId, upperBinId]` (int32 bin range); `[minPrice, maxPrice]` (price range, strings).
- `poolActiveBinId`, `poolActivePrice` — where the market is NOW; compare against `[lowerBinId, upperBinId]` to see if you're in range.
- `isOutOfRange` (bool|null), `isClosed` (bool).
- `feePerTvl24h` — this position's own fee/TVL over rolling 24h (is it actually earning?).
- `pnlUsd`, `pnlPctChange`, `pnlSol`, `pnlSolPctChange`.
- `createdAt`, `closedAt` (nullable int64).
- `allTimeDeposits`, `allTimeWithdrawals`, `allTimeFees` — each `TokenPairWithTotal` (per-token X/Y `TokenAmount{amount, usd, amountSol?}` + `total{usd, sol?}`). **`allTimeFees` is the real cumulative fee figure for the position** (the replacement for the deleted `/total_claim_fees`).
- `unrealizedPnl` (open positions only): `balances` (USD) + `balancesSol?`, `balanceTokenX/Y`, `unclaimedFeeTokenX/Y` (live unclaimed fees), `unclaimedRewardTokenX/Y` (live unclaimed LM rewards) — each a `TokenAmount`.

To decide hold-vs-claim-vs-close: compare `allTimeFees` (fees earned) against `pnlUsd` (is fee income beating IL?), read `unrealizedPnl.unclaimedFeeTokenX/Y` for what's sitting unclaimed, and check `poolActiveBinId` vs `[lowerBinId, upperBinId]` plus `isOutOfRange` for whether the position is live.

## Position history — `GET /positions/{address}/historical`

Here the path IS a **position address** (contrast with the P&L endpoint). Params: `address` (path); `event_type` (`add | remove | claim_fee | claim_reward`, optional → all); `order_direction` (`asc | desc`, default `desc` = newest first). Response `{ events[] }`; each `PositionEvent`: `signature`, `ixIndex`, `eventType`, `positionAddress`, `blockTime` (int64 sec), `slot`, `poolAddress`, `userAddress`, `tokenX`, `tokenY`, `amountX`, `amountY`, `amountXUsd`, `amountYUsd`, `totalUsd`, `createdAt` (ISO). Use it to reconstruct the deposit/withdraw/fee-claim/reward-claim timeline and audit claim cadence.

## Portfolio — lifecycle split (`user` REQUIRED on all three)

These are split by position lifecycle, NOT a single overview. Choose deliberately.

- **`GET /portfolio/open?user=<wallet>`** — pools where the user holds OPEN positions. Params: `user` (required), `page` (default 1), `page_size` (default 20, **max 50**), `sort_direction` (default `desc`), `sort_by` (prefer enum values `current_balances | unclaimed_fee | fee_per_tvl24h`). Each `PoolOpenPortfolioItem`:
  - `poolAddress`, `binStep` (int), `baseFee`, `collectFeeMode`.
  - token X/Y + `rewardX`/`rewardY` metadata.
  - `balances`(+Sol), `unclaimedFees`(+Sol), `totalDeposit`(+Sol), `poolPrice`.
  - `feePerTvl24h` — the user's OWN fee/TVL over rolling 24h.
  - `pnl`/`pnlPctChange`(+Sol variants) — live.
  - `openPositionCount`, **`listPositions[]`** — open position addresses.
  - **`outOfRange`** (bool|null) — true if ANY of the user's positions in the pool is out of range; null = undetermined.
  - **`positionsOutOfRange[]`** — the out-of-range position addresses (your primary rebalance signal).
  - Top-level response also carries `total?` (`TotalMetrics`) and `totalPositions`.
- **`GET /portfolio?user=<wallet>`** — pools where the user has CLOSED positions ONLY, sorted `last_closed_at` DESC. This is NOT a general overview. Params: `user` (required), `page`, `page_size`, `days_back`. The doc contradicts itself on defaults (prose: `page_size` default 120/max 365, `days_back` default 90; OpenAPI schema: `page_size` default 20/max 50, `days_back` default 120) — so **pass explicit `page`/`page_size`/`days_back`** rather than trusting defaults. Each `PoolPortfolioItem` gives realized `totalDeposit`/`totalWithdrawal`/`totalFee` (+Sol), `pnlUsd`, `pnlSol`, `pnlPctChange`, `lastClosedAt`. Drill into `/positions/{pool_address}/pnl?user=<wallet>&status=closed` for per-position detail.
- **`GET /portfolio/total?user=<wallet>`** — all-time total PnL across the portfolio (aggregates closed positions). Params: `user` (required). Response: `totalPnlUsd`, `totalPnlSol`, `totalPnlPctChange`, `totalPnlSolPctChange` (all strings).

## Realized claims — `GET /wallets/{wallet}/pools/{pool_address}/total_claims`

Combined realized claimed fees + rewards for a wallet in one pool. Params: `wallet`, `pool_address` (both path); no query params. Response `GetWalletTotalClaimsResponse`: fees `total_fee_x/y` (+`_usd`,`_sol`), `fee_claim_count`, `last_fee_claim_time` (ISO, nullable); rewards `total_reward_x/y` (+`_usd`,`_sol`), `reward_claim_count`, `last_reward_claim_time` (nullable); totals `total_claims_usd`, `total_claims_sol`. This is the realized-claims endpoint (`/positions/{address}/total_claim_fees` does not exist).

## Native limit orders (6 endpoints)

DLMM limit orders are on-chain buy/sell orders placed as liquidity at chosen bins in LO-enabled pools. An order deposits one token and fills bin-by-bin as price crosses, converting at each bin's fixed price, AND accrues a **bonus** (a pro-rata share of trade fees while its liquidity sits in bins).

- `is_ask_side` — **true = ask (selling X for Y); false = bid (buying X with Y).** `input_token` = deposited; `output_token` = expected fill token.
- **"Open"/"live"** = no `close_limit_orders` row. **"Closed"** = that row exists. **Cancel is NOT terminal** — a cancelled-but-unclosed order still shows under `/open`; cancel pays out bonus + unfilled remainder immediately.
- Monitor: `filled_pct` (100 ⟺ fully filled), `nearest_unfilled_bin_price` (`None` only when fully filled; distance = `(nearest_unfilled_bin_price − current_pool_price)/current_pool_price`), and `bin_distribution[]` (per-bin `fill_status`: `not_filled | partial_filled | fulfilled`).

| Endpoint | Purpose |
|---|---|
| `GET /wallets/{wallet}/limit_orders/open/pools` | Per-pool summary of live LOs. `page_size` default 20, max 1000. |
| `GET /wallets/{wallet}/limit_orders/open/pools/{pool_address}` | Per-order live detail; header carries `current_active_bin_id`, `current_pool_price`. |
| `GET /wallets/{wallet}/limit_orders/closed/pools` | Per-pool summary of closed LOs, sorted `last_closed_at` DESC. |
| `GET /wallets/{wallet}/limit_orders/closed/pools/{pool_address}` | Per-order lifecycle for closed LOs (`terminal_signature`, `terminal_slot`). |
| `GET /wallets/{wallet}/limit_orders/summary` | Wallet-wide counts + `total_deposit_usd/sol`, `total_bonus_usd/sol`. |
| `GET /wallets/{wallet}/limit_orders/pools/{pool_address}/bonus_claimed` | Realized bonus over every cancel event: `total_bonus_x/y`(+usd/sol), `total_bonus_usd/sol`, `cancel_event_count`. |

Shared `LimitOrderPoolDetails` header: `pool_address`, `pair_name`, `bin_step` (bps), `base_fee` (percent), token X/Y mint+symbol, `collect_fee_mode` (0 InputOnly / 1 OnlyY).

## Reward-farming (LM) fields

Rewards can flip a losing position positive, so surface them: pool-level `has_farm`, `farm_apr`, `farm_apy`, `reward_mint_x`, `reward_mint_y`; per-position live `unrealizedPnl.unclaimedRewardTokenX/Y`; realized `reward_claim_count` + `total_reward_x/y` from `/total_claims`. When you filter pools for a reward-farming strategy, add `farm_apr`/`farm_apy` to the comparison alongside `fee_tvl_ratio["24h"]`.

## Protocol stats — `GET /stats/protocol_metrics`

No params. Response `ProtocolMetricsResponse`: `total_tvl` (TotalTVL), `volume_24h` (Volume24h), `fee_24h` (Fee24h), `total_volume` (all-time), `total_fees` (all-time), `total_pools` (int64). Use for macro/protocol-health context.

## Recipe index (cross-links to SKILL.md workflows)

- **Pool selection:** `/pools` with `filter_by` on the two mint addresses (both orderings) or `query=<symbol>`, sorted `fee_tvl_ratio_24h:desc` with `filter_by=is_blacklisted=false && volume_24h>=<x>`; compare bin-step variants by `volume["24h"]`, `fee_tvl_ratio["24h"]`, `apr`, `pool_config.bin_step`.
- **Token due diligence before deposit:** `TokenMetrics.is_verified`, `freeze_authority_disabled`, `holders`, `market_cap` + pool `is_blacklisted`, `tags`, `launchpad`.
- **Data-driven rebalance / exit:** `/portfolio/open` → `positionsOutOfRange[]`/`outOfRange`; `/positions/{pool}/pnl` → `poolActiveBinId` vs `[lowerBinId, upperBinId]`, `isOutOfRange`, `feePerTvl24h`.
- **Hold vs claim vs close:** `/positions/{pool}/pnl` → `allTimeFees` vs `pnlUsd`, `unrealizedPnl.unclaimedFeeTokenX/Y`, `unclaimedRewardTokenX/Y`.
- **Launch-pool scan:** pool `launchpad` + `created_at`; short-window `volume_5m`/`volume_30m`/`fee_1h` to catch the launch fee/vol spike the 24h number hides.
- **Portfolio review:** `/portfolio/total` (all-time), `/portfolio` (closed history), `/positions/{pool}/pnl?status=closed` (per-position), `/positions/{address}/historical` (event timeline).
- **Reward-farming selection:** filter/compare on `has_farm`, `farm_apr`, `farm_apy` alongside fee/TVL.
- **Entry/exit timing:** `/pools/{address}/ohlcv` (realized range/volatility) + `/pools/{address}/volume/history` (trend) to set range width and judge whether volume supports staying in.

## Reference URLs

- DLMM Data API overview: `https://docs.meteora.ag/developer-guides/dlmm/api-reference/overview.md` (individual endpoints under `https://docs.meteora.ag/api-reference/dlmm/<group>/<page>.md`).
- DLMM TypeScript SDK reference: `https://docs.meteora.ag/developer-guides/dlmm/typescript-sdk/reference.md` (siblings `getting-started.md`, `examples.md`).
- Docs index for LLMs: `https://docs.meteora.ag/llms.txt`.
