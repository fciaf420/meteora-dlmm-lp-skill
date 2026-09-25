# DLMM Fees & Economics — the math behind every fee number

Read this when you're computing a fee value, modeling net LP yield, or explaining why fees spike or decay. The SKILL.md headlines ("10% cap", "LPs keep 90%", "composition fee trap") are shorthand for the exact math below — pull the real formula whenever a user wants a number rather than a vibe.

**Precision key — internalize this before touching any formula.** Fee rates are stored on-chain as integers in **1e9 precision** (`FEE_DENOMINATOR = 1,000,000,000`). A stored `10,000,000` means **1%** (`fee_rate / 1e9 × 100 = percent`). Basis points are a *separate* unit used for shares and bin spacing: **10,000 bps = 100%**. Never mix the two — a `protocol_share` of `1,000` is 1,000 bps (10%), not a 1e9 fee rate. The program does integer arithmetic with explicit rounding, so on-chain results differ slightly from naive decimal math; when a user's claimed fees are "a bit off" from your estimate, that rounding (outputs floored, required inputs and fees ceiled — always favoring the pool) is usually why.

## Bin pricing and the constant-sum invariant

Bin prices form a geometric ladder, computed as Q64.64 fixed-point:

```
P_i = (1 + bin_step / 10,000) ^ i        # i = bin ID, can be negative
```

Bins are evenly spaced in *percentage* terms, not absolute price. Even though bin_step is quoted in bps, 25 bps means each neighbor is **1.0025×** the last (moving bin 0 → bin 1 multiplies price by 1.0025). Concretely: SOL at $20 with a 25 bps step gives next bins ≈ $20.05, $20.10. N bins from the active bin = price ratio `(1 + bin_step/10,000)^N`.

Inside a single bin, liquidity obeys a **constant-sum** invariant (not the constant-product `x*y=k` of a classic AMM):

```
L = P · x + y
```

This fixed-price design is *why* there is **zero price impact inside a bin** — a swap trades at the bin's exact price until that bin's output-side reserve is fully consumed, then price steps to the next bin. Use this when explaining slippage: the trader eats price impact only at bin boundaries, and IL arrives in discrete jumps at those same boundaries, not continuously.

## Base fee — the pool minimum (CORRECTED)

```
base_fee_rate = base_factor × bin_step × 10 × 10^base_fee_power_factor    # stored in 1e9 precision
```

The **× 10** constant is load-bearing. A common mis-statement (`bin_step × base_factor × 10^base_fee_power_factor`) drops the × 10, making every base-fee estimate **10× too low** — never reproduce it. Divide the result by 1e9 to get the fractional rate.

The base fee is the *minimum* fee any swap in the pool pays; it is set at creation in the pool's `StaticParameters` (standard permissionless pools pick from an allowed preset, `PresetParameter2`). It is **not** fixed for life: the program bounds it to **`MIN_BASE_FEE` = 100,000 (0.01%)** through **`MAX_BASE_FEE` = 100,000,000 (10%)** at 1e9 precision, and a pool operator can change it after creation with the `update_base_fee_parameters` instruction. So read the current on-chain parameters rather than assuming the creation-time value. Because bin_step is a direct multiplier, larger-bin-step pools carry higher base fees — which fits their purpose (each bin is a bigger price move, so LPs need more per-trade compensation). The creator's tradeoff, worth surfacing when advising on pool choice or creation: a **lower base fee may attract more volume**, while a **higher base fee earns more per unit of volume**. There is no single right answer; it depends on the pair's expected flow.

Worked arithmetic to internalize the × 10 (plug the pool's real `StaticParameters` — the inputs below are illustrative only). With `base_factor = 5,000`, `bin_step = 25`, `base_fee_power_factor = 0`: correct = `5,000 × 25 × 10 × 10^0 = 1,250,000` stored → `1,250,000 / 1e9 = 0.125%`. The mis-stated formula (missing × 10) gives `5,000 × 25 × 10^0 = 125,000` → `0.0125%` — exactly 10× too low. Any time your base-fee estimate looks suspiciously tiny, check that the × 10 is present.

## Variable (dynamic) fee — volatility surge pricing

```
variable_fee_rate = ceil( variable_fee_control × (volatility_accumulator × bin_step)^2 / 1e11 )
```

The denominator is **100,000,000,000 (1e11)**. If `variable_fee_control = 0`, the variable fee is **0** — not every pool has one; it is parameter-driven. The critical property for advice: the fee scales with the **square of (volatility × bin_step)**. That squared term is why high-bin-step pools escalate fees fastest under volatility, and why a launch pool at 100+ bps can jump toward the cap in seconds while a 1 bps stable pool barely moves.

The `volatility_accumulator` grows as the active bin moves and decays over time:

```
volatility_accumulator = min( volatility_reference + |index_reference − active_id| × 10,000,
                              max_volatility_accumulator )
```

Each bin of price movement adds `× 10,000` to the accumulator, and it is hard-capped by `max_volatility_accumulator`. The reference decays on a timer — this is the "cool-down" that makes the surge fade:

| Time since last update | Behavior |
|---|---|
| < `filter_period` | Keep existing `volatility_reference` **and** `index_reference` (rapid trades stay "hot"). |
| between `filter_period` and `decay_period` | `index_reference = active_id`; `volatility_reference = volatility_accumulator × reduction_factor / 10,000` (partial decay). |
| ≥ `decay_period` | `index_reference = active_id`; `volatility_reference = 0` (fully cooled). |

The `index_reference = active_id` reset is easy to miss but required to reproduce the accumulator: once `filter_period` has elapsed, the next swap measures bin movement from the bin that was active at that moment, not from the older reference.

So when a user asks "why did fees spike then drop?": swaps crossing bins in quick succession pump the accumulator (fees rise), and once trading quiets past `decay_period` the reference resets to 0 (fees normalize). All three knobs — `filter_period`, `decay_period`, `reduction_factor` — live in the pool's `StaticParameters` and govern how fast the surge fades.

## Total fee and the 10% hard cap

```
total_fee_rate = min( base_fee_rate + variable_fee_rate, MAX_FEE_RATE )
```

`MAX_FEE_RATE = 100,000,000` at 1e9 precision = **10%**. This is an absolute ceiling: no matter how violent the launch or how high volatility drives the variable component, an LP will never see a total swap fee above 10% on-chain. Use this to bound any "substantial returns during volatility" framing — the per-swap fee is capped, so headline fee APR comes from *volume × 10% at most*, not from an unbounded fee.

## Applying the total fee — inclusive vs exclusive

The `total_fee_rate` is applied in one of two forms depending on swap direction and the pool's Collect Fee Mode. Know both so a fee figure reconciles against on-chain amounts:

```
# Form A — fee is carved out of an amount the trader already provided (fee-inclusive):
fee_amount       = ceil( amount_with_fee × total_fee_rate / 1e9 )
amount_excluding = amount_with_fee − fee_amount

# Form B — fee is added on top of a fee-exclusive amount:
fee_amount       = ceil( amount_excluding_fee × total_fee_rate / (1e9 − total_fee_rate) )
amount_including = amount_excluding_fee + fee_amount
```

Fee amounts round **up** (ceil), favoring the pool. Form B's `(1e9 − total_fee_rate)` denominator is why grossing a fee back up from a net amount isn't just "× rate" — mind it when reverse-engineering a trader's fee from an output amount.

**Price-impact guard** (program-only, per Meteora's formulas doc; the SDK exposes only the input, `swapWithPriceImpact({ priceImpact })` in bps, not the math). Some swap flows carry a max price impact, checked against a guard price derived from the active bin:

```
X→Y: minimum price         = P_active × (10,000 − max_price_impact_bps) / 10,000
Y→X: maximum effective price = P_active × 10,000 / (10,000 − max_price_impact_bps)
```

## Protocol and host split — model this or your net yield is wrong

The protocol takes its cut **before LPs see anything**. This is the single most common omission in naive fee-APR math, so always net it out — **but only once**. The Data API's `fees`, `fee_tvl_ratio`, and `apr` are **already LP-net** (`fees + protocol_fees = volume × total_fee_rate`; live check: `protocol_fees / (fees + protocol_fees)` ≈ 0.10 on standard pools). Apply the LP share only to a **gross** figure you computed yourself (`volume × fee_rate`, or `fees + protocol_fees`), never to API fee/APR fields.

| Pool type | Typical protocol share | LP keeps |
|---|---|---|
| Standard DLMM pool | **10%** (`protocol_share` = 1,000 bps) | **90%** |
| Launch Pool | **20%** (`ILM_PROTOCOL_SHARE` = 2,000 bps) | **80%** |

`protocol_share` is a **per-pool** parameter capped at **2,500 bps (25%)**; the table shows typical values, not fixed ones. For an exact number, read it on-chain: `lbPair.parameters.protocolShare` via the SDK (`DLMM.create(...)` then `pool.lbPair.parameters.protocolShare`, or `pool.getFeeInfo().protocolFeePercentage`). Do **not** trust the Data API's `pool_config.protocol_fee_pct` (it reports 5 on pools that are 1,000 bps = 10% on-chain) or the IDL constant `PROTOCOL_SHARE = 500`. Without SDK access, the realized ratio `protocol_fees / (fees + protocol_fees)` from the Data API is a good empirical check. The split:

```
protocol_fee = floor( trading_fee × protocol_share / 10,000 )
LP_fee       = trading_fee − protocol_fee
```

The launch-pool double-cut matters: launch LPing is often pitched as "extremely high fee potential" without mentioning that the protocol takes **20%** there vs 10% on standard pools. Launch APR expectations must be computed net of that larger share.

**Referral / host fee** is carved from the *protocol's* slice, never from the LP's. When a swap routes through a referral account, the host gets **20%** of the protocol fee (`HOST_FEE_BPS` = 2,000 bps); with no referral account the host fee is 0 and the whole protocol slice stays with the treasury. Because it comes out of the protocol portion, it does **not** change LP take.

```
host_fee                 = floor( protocol_fee × 2,000 / 10,000 )     # 0 without a referral account
protocol_fee_remainder   = protocol_fee − host_fee
trading_fee              = LP_fee + protocol_fee_remainder + host_fee  # liquidity-mining pools (all fills are MM)
```

On limit-order pools the host fee also takes a capped slice of the limit-order protocol fee — see "Native limit-order fee split" below.

**Net-yield formula the advisor should use:**

```
LP_take = gross_swap_fee × LP_share          # LP_share = 1 − protocol_share/10,000 (typically 0.90 standard, 0.80 launch)
```

`gross_swap_fee` here means a figure you built yourself from `volume × total_fee_rate`. If you started from Data API `fees` / `apr` / `fee_tvl_ratio`, you already have `LP_take` — multiplying again understates yield by 10% (standard) or 20% (launch).

## Composition fee — the off-ratio entry/rebalance trap

When you deposit into the **active bin** with a token mix that differs from that bin's current X:Y ratio, the program treats the imbalance like a mini-swap and charges a composition fee:

```
composition_fee = floor( Δamount × total_fee_rate × (1e9 + total_fee_rate) / 1e9^2 )
```

Protocol share is then skimmed from the composition fee just like a swap fee (`floor(composition_fee × protocol_share / 10,000)`). Critically, there is **no composition fee** when depositing into an **empty bin** or any **non-active bin** — only the active bin, and only when the deposit shifts its composition.

Advise LPs to avoid it two ways: **match the active bin's current X:Y ratio** when depositing at the active price, or **place liquidity outside the active bin** (e.g., single-sided ranges above/below current price). This is a real, easily-overlooked cost on entry and on any rebalance that tops up the active bin — flag it whenever a user proposes an off-ratio deposit at the current price.

## Liquidity shares and withdrawal rounding

Deposits mint per-bin liquidity shares; withdrawals burn them pro rata. Both round **down**, favoring the pool:

```
L_in  = P · x_in + y_in                     # y is Q64.64-scaled (× 2^64) in share math
share = L_in                                 # bin has no liquidity supply yet
share = floor( L_in × liquidity_supply / L_bin )   # bin already has liquidity

out_x = floor( share × bin_amount_x / liquidity_supply )   # withdrawal
out_y = floor( share × bin_amount_y / liquidity_supply )
```

That floor on each bin, summed across a wide position, is why a full withdrawal can come back a few base units short of a naive pro-rata estimate.

## Collect Fee Mode — which token your fees arrive in

A **pool-level** setting fixed at creation. An LP **cannot change it**; existing pools already have one configured. It decides the *denomination* of accrued fees, not how much you earn.

| Mode | Value | Fees collected in | Effect |
|---|---|---|---|
| `InputOnly` | **0** | The token *entering* the swap | X→Y charges in X, Y→X charges in Y → balanced fee exposure in both assets over time. |
| `OnlyY` | **1** | Always token Y | Predictable single-token fee accrual, typically when Y is the quote asset (SOL/USDC). |

Under `OnlyY`, even an X→Y swap denominates its fee in token Y (accounted from the output side). Strategy implication: for LPs accumulating a **quote token** or running a DCA plan, an `OnlyY` pool concentrates all fee income in the quote asset (clean accounting, easy DCA math); an `InputOnly` pool spreads fee income across both sides. **Warning to always pass through**: Collect Fee Mode does **not** remove market/IL risk — choosing `OnlyY` does not protect you from ending up with a lopsided token composition after price moves through your bins.

## Liquidity-mining reward accrual

For reward-enabled (liquidity-mining) pools, rewards accrue per token of liquidity in the active bin:

```
Δreward_per_token = floor( seconds_elapsed × reward_rate / active_bin_liquidity_supply )
reward_rate       = (new_funded_amount + remaining_reward_amount) / reward_duration
```

Two mechanics drive the "stay in range" advice:

- **Empty-active-bin time is recorded, not paid.** If the active bin has no liquidity, the program logs the elapsed time instead of paying LPs; the funder can carry those unvested rewards forward. Rewards only land in bins that hold liquidity **at the active price**.
- **Rewards split across crossed bins**, but a **v2 swap splits across at most 15 bins** with liquidity. Combined with the fee rule that only bins the price actually trades through earn anything, this is the mechanistic backing for "concentrate near where price trades and stay in range." More TVL in your bin also *dilutes* your per-share reward, since rewards are emitted per token of liquidity.

## Native limit-order fee split

On limit-order-mode pools a fill can come from MM positions, limit orders, or both in the same bin. The program first splits the trading fee **by liquidity source** (rounding favors MM — the MM share is rounded **up**), then splits each slice by recipient. The limit-order slice splits **50/50** between the order participant and the protocol (`LIMIT_ORDER_FEE_SHARE` = 5,000 bps), whatever the pool's `protocol_share`:

```
total_MM_fee                = ceil( trading_fee × MM_amount_in / total_amount_in )
total_limit_order_fee       = trading_fee − total_MM_fee

MM_protocol_fee             = floor( total_MM_fee × protocol_share / 10,000 )
MM_LP_fee                   = total_MM_fee − MM_protocol_fee

limit_order_participant_fee = floor( total_limit_order_fee × 5,000 / 10,000 )
limit_order_protocol_fee    = total_limit_order_fee − limit_order_participant_fee

# host fee (referral swaps only; otherwise 0)
host_fee_on_limit_order     = min( floor( floor(total_limit_order_fee × protocol_share / 10,000) × 2,000 / 10,000 ),
                                   limit_order_protocol_fee )
host_fee                    = floor( MM_protocol_fee × 2,000 / 10,000 ) + host_fee_on_limit_order
protocol_fee_remainder      = MM_protocol_fee + limit_order_protocol_fee − host_fee

# full identity
trading_fee = MM_LP_fee + limit_order_participant_fee + protocol_fee_remainder + host_fee
```

The source split and the MM/LO recipient splits match the SDK's quote math (`commons/src/quote.rs` `split_fee`). The **host-fee cap on the limit-order slice is documented by Meteora but not verifiable in the public SDK** (the SDK has no limit-order host-fee path, and the program source is not public). With no limit-order fill, `total_limit_order_fee = 0` and everything reduces to the liquidity-mining formulas above.

This is a distinct economic path from MM LPing — a limit-order participant keeps 50% of the fee on the slice their order fills, not the 90%/80% an MM LP keeps. Only relevant on pools whose function mode is limit-order (a pool is liquidity-mining XOR limit-order, never both).

## Worked net-yield skeleton (fill from live API data — never invent APRs)

Never quote an APR from memory. Compute it from API-sourced inputs, running them through this chain:

1. **LP fee** = pool LP fee income attributable to your bins over the window (`fees["24h"]` is already LP-net — scale it to your share of the active/crossed bins — or a position's `allTimeFees`).
2. **Protocol cut: already excluded from API fields — do not deduct again.** Only when you started from a *gross* figure (`volume × total_fee_rate`, or `fees + protocol_fees`) multiply by the LP share, `1 − protocol_share/10,000` (typically `× 0.90` standard, `× 0.80` launch).
3. **± LM reward** → add farm rewards for reward pools (`farm_apr`/`farm_apy`), but only for the fraction of time your liquidity sat in the active/paying bins.
4. **vs IL** → subtract realized/unrealized IL (from position P&L). Net LP result = fees net of protocol + rewards − IL − any composition fee paid on entry − non-refundable bin-array rent on new ranges.

Every dollar figure in that chain must come from a digest-backed API field or the user's own numbers. If you lack an input, say so and pull it — do not backfill with an assumed rate.

## Quick constant reference

| Constant | Value |
|---|---|
| `FEE_DENOMINATOR` (fee precision) | 1,000,000,000 (1e9) |
| 1% fee rate, stored | 10,000,000 |
| `MAX_FEE_RATE` (total fee cap) | 100,000,000 = 10% |
| `MIN_BASE_FEE` / `MAX_BASE_FEE` | 100,000 (0.01%) / 100,000,000 (10%) |
| Variable-fee denominator | 100,000,000,000 (1e11) |
| bps for 100% | 10,000 |
| Standard `protocol_share` (typical; per-pool — read on-chain) | 1,000 bps = 10% (LP keeps 90%) |
| Launch `ILM_PROTOCOL_SHARE` (typical) | 2,000 bps = 20% (LP keeps 80%) |
| Max `protocol_share` | 2,500 bps = 25% |
| Referral `HOST_FEE_BPS` | 2,000 bps = 20% of protocol slice |
| `LIMIT_ORDER_FEE_SHARE` | 5,000 bps = 50/50 |
| v2 swap reward-bin split limit | 15 bins |
| Collect Fee Mode | `InputOnly` = 0, `OnlyY` = 1 |
| Bin price base (25 bps) | 1.0025 per bin |

**Source docs:** Formulas https://docs.meteora.ag/core-products/dlmm/formulas — Collect Fee Mode https://docs.meteora.ag/core-products/dlmm/collect-fee-mode — Program accounts (StaticParameters, protocol_share, collect_fee_mode) https://docs.meteora.ag/developer-guides/dlmm/program/accounts
