# Launch Pools & the Dynamic Terminal

Read this when you are advising on a token launch or memecoin LP position, or when the user is operating through the Meteora **Dynamic Terminal** UI on meteora.ag. This is the execution-and-due-diligence layer: how the current UI actually behaves, what a rebalance really costs, how to screen a fresh token before you touch it, and how launch-pool seeding parameters shape the position you are stepping into.

Launch LPing is the highest-variance thing an LP can do on DLMM. Do not frame "extremely high fee potential" as if it were free money — the protocol takes a bigger cut, fees do not compound themselves, and every rebalance into virgin price territory burns unrecoverable SOL. Advise net of all of that.

## Terminal defaults & the liquidity chart

The Dynamic Terminal is the revamped DLMM pool page: a left panel (pool stats + token risk), a TradingView chart top-center, your open positions bottom-center (live P&L), and the create/manage panel on the right.

- **Default position span is ~69 bins**, centered on the active price. Two ways to change it, and they move in opposite directions: dragging the range slider only ever *narrows* (down from 69 bins), while typing Min/Max prices directly *widens* — up to **1,400 bins**. You can also set Min%/Max% from the active bin and nudge with the per-bin +/− buttons.
- **Read the liquidity chart by color:** purple bins hold the base token (e.g. SOL), cyan/blue bins hold the quote token (e.g. USDC), grey bins are existing pool liquidity from other LPs. Hover any bin for its price and token breakdown. A deposit that is all purple on one side and all cyan on the other is the normal two-sided shape; all-one-color is single-sided.
- **Width is the core tradeoff:** a narrower range is more capital efficient with higher fee capture per dollar, but higher risk of going out of range; a wider range captures less fee per dollar but is more resilient and rebalances less often. At launch, resilience usually wins — see TA-driven placement below.
- **Liquidity Slippage** governs how much price drift you tolerate *while the add-liquidity tx lands*. On a fast-moving launch pool, raise it or the tx fails with `ExceededBinSlippageTolerance` (error 6004) when the active bin jumps mid-transaction.
- **Auto-Fill** (default ON) fills the matching other-token amount at the current rate. Toggle it OFF to go single-sided — enter an amount in only one field. This is the mechanism behind DCA-out (deposit only the token you are selling, Bid-Ask shape, over a range above spot).

## Position-management toolkit (right panel)

The Terminal exposes one-click operations that map to real on-chain flows — know what each does before recommending it:

- **Rebalance** — one click to re-center the range around the current active price. Under the hood this is a claim + remove + resize + add flow (see `fees-and-economics.md` for the in-place resize model); it is not free (see rent below).
- **Ape In** — swap + create a position in a single transaction from one token. For moving fast on an opportunity; you skip manual balancing but eat swap slippage.
- **Zap Out** — withdraw and convert everything to one preferred token in a single step. **Only available for positions of ≤ 250 bins** — a very wide launch position cannot Zap Out and must be removed the normal way.
- **Withdraw and Close** — exit entirely and reclaim the refundable rent (below).
- **Add Liquidity / Remove Liquidity** — top up or partially pull within the existing range.
- **Batch actions:** **Claim All Fees** collects fees from every position in one tx; **Close All Positions** exits them all at once.
- **Hard constraint:** you **cannot withdraw single-sided from the active bin**. The active bin holds both tokens, so any withdrawal touching it returns both — you cannot cherry-pick just the quote or just the base out of the active price point.

The bottom-center **Your Positions** panel shows, per position: open Date & Time, **Your Liquidity** (current value of deposited tokens), **Claimable Fees** (accumulated, unclaimed), **P&L** in $ and %, and **24h Fee/TVL %** (short-term fee-generation rate). When the active price leaves your range, the position flips **inactive** and stops earning — the fees you already accrued sit there unclaimed until you either wait for price to return or rebalance. Read these numbers to the user before recommending hold vs rebalance; do not advise from a hypothetical.

## SOL rent — the cost most LPs miss

The prior skill called rebalancing "gas fees on Solana are low." That is only half true and it hides the real cost. Each position touches two kinds of SOL rent:

- **Refundable rent** — position-creation rent and position-extension rent. This is returned in full when you close the position. Opening and closing a position is roughly rent-neutral over its life.
- **Non-refundable rent** — SOL spent to **create new bin arrays** that do not yet exist on-chain. Bin arrays hold 70 bins each; when your range reaches into price territory no one has initialized, you pay to create those arrays and **that SOL is not recoverable.**

Why this matters for launch LPing specifically: rebalancing or widening into fresh price territory — exactly what you do when a launch token is running or dumping through undiscovered price levels — keeps opening new bin arrays. Frequent re-centering on a trending launch token burns non-refundable bin-array rent every time. Before recommending an aggressive rebalance cadence, tell the user to click **"Show cost details"** in the Terminal, which breaks out refundable vs non-refundable before they confirm. A strategy that rebalances ten times through price discovery is paying real, gone SOL each hop, on top of any IL it locks in.

There is a third, easily-missed cost on top of rent: the **composition fee**. When you add liquidity *to the active bin* with a token mix that differs from the bin's current X:Y ratio, the program charges a fee because that off-ratio deposit behaves like a mini-swap. Re-centering a position onto the active price is exactly when this bites. It is avoided only when you deposit into an empty bin or a non-active bin, or when you match the active bin's current ratio. So the true cost of a rebalance at the active price is: non-refundable bin-array rent + composition fee + any IL locked in by closing the old range — not "low gas."

## Sync price before you deposit

A newly created or low-liquidity pool can sit at a price that does not match the market — you can seed a mispriced pool and get instantly arbitraged. Meteora uses **Jupiter's price API as the market-price reference**. Before depositing into any fresh pool:

- Compare the pool's active-bin price against Jupiter's price (shown in the Terminal).
- Use the **"Sync with Jupiter's price"** button when it is available — it appears when there is 0 liquidity between the active bin and the Jupiter-price bin, or the liquidity is already close enough to that bin.
- If liquidity sits in between and is too far to auto-sync, either wait for arbitrage to correct it or make a few tiny manual swaps in the correcting direction, then deposit.

Skipping this on a low-liquidity pool means you are the one providing the mispriced liquidity that arbitrageurs eat.

## Fees at launch — model them net, not gross

Launch fees look enormous during the opening volatility spike, but three facts cut into the headline number:

- **Dynamic fees surge and cap.** The fee is base floor + a volatility surge, hard-capped at **MAX_FEE_RATE = 10%** total. No matter how wild the launch, on-chain total swap fee never exceeds 10%. The **base floor = base_factor × bin_step × 10 × 10^base_fee_power_factor** (stored in 1e9 precision) — note the literal `× 10`; leaving it out understates the base fee tenfold. The surge scales with the *square* of volatility × bin_step, so a high-bin-step launch pool escalates fees much faster under the same price movement — one reason launch pools use wide bin steps.
- **The protocol cut doubles on launch pools.** Standard DLMM pools take **10%** of the trading fee for the protocol (LP keeps 90%); **Launch Pools take 20%** (`ILM_PROTOCOL_SHARE` = 2,000 bps — LP keeps only 80%). Compute any launch APR estimate net of the 20% haircut, not the gross fee.
- **Fees do NOT auto-compound.** They accumulate and must be claimed manually. A position that goes out of range stops earning entirely (it is inactive until price returns), so unclaimed launch fees just sit there while the token dumps past your bins.

The honest question for a launch LP is: will the fees earned *while price is actually trading through my bins* — net of the 20% protocol cut and net of non-refundable rebalance rent — exceed the IL from an 80%+ drawdown? Sometimes yes during the frenzy; often no once volume dries up. Cross-link the full fee split and IL math in `fees-and-economics.md`.

## Pre-entry risk screening (memecoin due diligence)

Before LPing a fresh token, pull the risk panel the Terminal surfaces on the left and read it out to the user — do not LP a token blind. Key fields:

- **Jupiter Organic Score** — Meteora's headline trust signal for the token.
- **Top 10 Holders %** and **Top 10 Dev Wallets %** — concentration; high dev-wallet supply is dump risk.
- **Mint Authority** and **Freeze Authority** status — an active mint authority can inflate supply; an active freeze authority can freeze token accounts. Both are red flags for a token you will hold via IL.
- **FDV**, **Market Cap**, **Token Age**, **Holders**, **Total Supply**, and 5m/1h/12h/24h price change.
- External deep-dives linked from the panel: **RugCheck**, **Bubblemaps**, Solscan.

If you have API access to a token-security source, corroborate with programmatic flags: `is_verified`, `freeze_authority_disabled`, `is_blacklisted`. A token failing these plus high dev-wallet concentration is one to decline outright, not to size down into.

## TA-driven range placement

Instead of the "cover the 7-day range plus a buffer" hand-wave, use the integrated TradingView chart, which sits on the same page as the create panel (desktop and tablet; 100+ indicators, Fibonacci tools, drawings persist across sessions). Concrete method:

- **Anchor Min price to support and Max price to resistance.** Use Fibonacci retracements to find the reversion zones for a range-bound thesis, and set your position edges to those levels rather than to an arbitrary percentage.
- **Enter during consolidation.** Tighter price action near your target range means more time in range; check momentum before committing to a concentrated range.
- **Widen the range in high volatility.** In an active launch, a wider range keeps you in range longer and avoids repeated (rent-burning) re-centers; tighten only once volatility settles.
- **Exit before a clear trend** pushes price far outside the position — do not passively watch a breakout carry price out of every bin you own.

## Launch-pool seeding & activation (what you are stepping into)

If the user is creating a launch pool — or LPing into one and wants to understand its wiring — these are the levers a creator sets via the Meteora Invent toolkit (`pnpm studio ...`, config in `studio/config/dlmm_config.jsonc`). They shape the position and the risk:

- **`activationType` (0 = Slot, 1 = Timestamp) + `activationPoint`** gate when trading opens. Seeding must happen *before* activation — both `dlmm-seed-liquidity-lfg` and `dlmm-seed-liquidity-single-bin` only work while the pool is not yet activated.
- **Seeding shape:**
  - **LFG curvature** (`lfgSeedLiquidity`) distributes across a price curve over `minPrice`/`maxPrice`. `curvature` is `1/k`, range **0–1, lower = more concentrated** (example 0.6). This is how a creator front-loads liquidity near the launch price.
  - **Single-bin** (`singleBinSeedLiquidity`) dumps all base token into one bin at one `price` — maximal concentration, a classic launch shape. LP-facing implication: with all seeded liquidity at a single price point, the moment buying pushes price out of that bin there is little depth behind it, so early price impact is violent. If you are LPing into a single-bin-seeded launch, expect fast, discontinuous moves and size your range wider than you would for an LFG-curved seed.
- **`lockReleasePoint`** — the slot/timestamp before which seeded liquidity cannot be withdrawn; **0 = immediately unlocked.** Removing before it fails with `LiquidityLocked` (error 6055). If you are an outside LP, check whether the team's liquidity is locked or can be pulled at will.
- **`creatorPoolOnOffControl`** — on a customizable permissionless pool, the creator retains a switch to **enable/disable trading** (`dlmm-set-pool-status`). Real LP risk: the creator can pause the pool. A disabled pool is withdraw-only (`PoolDisabled`, error 6042) — you can pull liquidity but not add or swap.
- **`pre_activation_duration` + `pre_activation_swap_address`** — a single designated wallet may swap *before* public activation. That is a sanctioned front-run window; the first public price you see may already reflect that wallet's buy.
- **Restricted launch flows require the quote token to be SOL or USDC** — a non-SOL/USDC quote reverts with `InvalidQuoteToken` (error 6061).

## Alpha Vault — the real anti-bot mechanism

The Alpha Vault is more than a vague "anti-bot mechanism." Concretely, it is a pre-activation deposit vault that reserves the first buys for genuine participants ahead of snipers, in two flavors:

- **FCFS (First-Come-First-Serve):** `maxDepositCap` (total quote across all users) and `individualDepositingCap` (per-user quote cap). Everyone who gets in before the cap fills gets their allocation.
- **Prorata:** `maxBuyingCap` (total quote bought across all users); deposits above the cap are refunded pro rata.

Both use `depositingPoint` (when deposits open), `startVestingPoint`/`endVestingPoint` (when purchased tokens vest and become claimable), an `escrowFee` (quote-token cost to create a stake escrow account), and a `whitelistMode` (`permissionless`, `permissioned_with_merkle_proof`, or `permissioned_with_authority`, with optional CSV/merkle-proof whitelist plumbing). The vault can front any pool type via `poolType` (`dlmm`, `dynamic`/DAMM v1, or `damm2`). For an LP, a pool with an Alpha Vault means the earliest, most volatile buying is partly pre-allocated and then subject to a vesting schedule — the opening print, and the selling pressure as vesting unlocks, both behave differently from a raw permissionless launch. Check the vesting window before assuming the first-day price is the clearing price.

## Version note — lb_clmm 0.12.0

The current program release adds LP-facing capabilities worth knowing when advising launch/active LPs:

- **Native limit orders are live** — `place_limit_order` / `cancel_limit_order` / `close_limit_order_if_empty` on-chain; `placeLimitOrder`, `cancelLimitOrder`, `quoteCreateLimitOrder` in the SDK. This is real limit-order infrastructure (up to 50 bins per order), not just the "each bin is a limit order" analogy — a cleaner tool for DCA-in/out than a single-sided Bid-Ask position for some users. Note the different economics: a limit-order participant earns **50%** of the fee on the portion their order fills (the other 50% goes to protocol), a distinct path from MM LPing where you keep 90%/80%. A pool is **LimitOrder XOR LiquidityMining** — a pool that supports limit orders cannot also run LM/farm rewards, and vice versa.
- **Quote Token Fee / `collect_fee_mode`** — pools can now collect fees single-sided in token Y instead of on the input token. Default for all existing accounts is `InputOnly` (value 0); `OnlyY` (value 1) concentrates fees in the quote asset — launch-relevant when a new token is paired against SOL/USDC and the team wants predictable quote-side fee capture.
- **Max bins swappable per instruction dropped 280 → 260** (limit-order logic costs more compute) — matters only for very large routed swaps across many bins.
- **v1 `Position` is removed** — positions now use the `DynamicPosition` model, which is what makes in-place resize (rather than close-and-reopen) possible. Any tooling still referencing v1 positions is stale.
