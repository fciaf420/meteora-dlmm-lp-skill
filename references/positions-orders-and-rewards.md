# Positions, Orders, and Rewards — the machinery under a DLMM position

Read this when a strategy depends on the plumbing rather than the spread: the user wants to
**resize or rebalance a position without closing it**, **place native limit orders**, **farm
liquidity-mining rewards**, **delegate management** of a position to a bot or teammate, **lock
launch liquidity** (or is stuck unable to withdraw it), or **vet a Token-2022 pool** before
providing liquidity. These headers back the "beyond spread liquidity" material in SKILL.md; the
fee math those decisions feed into lives in `fees-and-economics.md`.

Everything here is grounded in the `lb_clmm` program (ID
`LBUZKhRxPF3XUpBCjp4YzTKgLccjZhTSDM9YuVaPwxo`, same on mainnet-beta and devnet). The current
on-chain position type is `PositionV2`; the legacy v1 `Position` type is gone as of the 0.12.0
release, so do not advise around it.

## A position is an account, not a token

Correct the most common mental-model error up front: when an LP adds liquidity they do **not**
receive a fungible LP token and they do **not** receive a position NFT. They create or update a
**dedicated on-chain account** — `PositionV2` — that records exactly where their liquidity sits.
There is nothing transferable in a wallet's token list to represent the position; the account
itself is the position.

Each `PositionV2` stores:

- the **pool** (`lb_pair`) it belongs to and the **owner** wallet — a position is bound to exactly
  **one pool and one owner**;
- the **bin range** as an inclusive `[lower_bin_id, upper_bin_id]`;
- **per-bin liquidity shares** — the LP's proportional claim on each bin's fees and eligible
  rewards;
- **pending and lifetime-claimed fees tracked separately for token X and token Y**
  (`fee_x_pending`/`fee_y_pending` are claimable now; `total_claimed_fee_x_amount`/
  `total_claimed_fee_y_amount` are cumulative);
- **reward checkpoints** for up to 2 reward slots.

A position starts with the **default 70-bin layout** (70 inline liquidity shares, fee checkpoints,
and reward checkpoints). Ranges wider than 70 bins append extra per-bin data (`PositionBinData`)
after that header, up to a **maximum of 1,400 bins** (the program constant `POSITION_MAX_LENGTH`). Note that a bin **array** is also 70 bins —
`MAX_BIN_PER_ARRAY = 70` — so the default position exactly fills one array's worth of bins. (The
commonly cited "69" is the UI's *default range width*, not the account's bin capacity; the account
number is 70.)

## Resizing and one-transaction rebalance

The old model — "rebalancing means close the position and open a new one" — is outdated. Dynamic
Positions can be **resized in place, without closing**. This changes the advice materially: an LP
who wants to widen a defensive range or tighten back up after volatility cools does not have to eat
a full teardown/rebuild.

Three operations:

- **Increase length** — add bins to the lower side, the upper side, or both. New allocation beyond
  the default 70-bin layout is **capped at 91 added bins per resize instruction**; a very wide
  target range is reached by chunking across multiple instructions (exceeding the per-instruction
  realloc budget throws `ReallocExceedMaxLengthPerInstruction`, error 6086).
- **Decrease length** — remove **empty** bins from the lower or upper side. Only empty bins can be
  shed; bins that still hold liquidity or unclaimed fees block the shrink (`NonEmptyPosition`,
  error 6030).
- **Rebalance** — the `rebalance_liquidity` primitive combines **claim + remove + resize +
  add-liquidity into a single position-management flow**. This is what the UI's one-click
  "Rebalance / re-center" button and one-transaction rebalance bots use.

When rebalancing, a **shrink mode** controls how aggressively the range contracts:

| Shrink mode | Behavior |
| --- | --- |
| `ShrinkBoth` | Remove empty bins from both sides when possible. |
| `NoShrinkLeft` | Keep the lower side; shrink only from the upper side. |
| `NoShrinkRight` | Keep the upper side; shrink only from the lower side. |
| `NoShrinkBoth` | Keep both sides; do not shrink. |

Two things to keep telling LPs regardless of resize convenience. First, **only in-range liquidity
earns** — a position whose active bin has left the range stays open but sits idle, earning neither
fees nor rewards until price returns or the LP resizes/rebalances around the new active bin.
Resizing does not rescue an out-of-range position on its own; you still have to move liquidity to
where trading is happening. Second, a rebalance is not free just because Solana gas is cheap:
beyond the low network fee it carries **SOL rent** — refundable position-creation and extension
rent (returned when the position closes) plus **non-refundable rent to create any new bin arrays**
your new range touches. That bin-array rent is a real, unrecoverable cost, so widening into
never-before-used far bins is more expensive than nudging an existing range.

## Native limit orders (full mechanics)

DLMM has **first-class on-chain limit orders** (program feature since `lb_clmm` 0.12.0, mainnet
2026-05-21) — not merely the "each bin behaves like a limit order" analogy. A limit order is its
own account that parks liquidity in specific bins so it fills like a resting buy or sell order when
normal swap flow reaches those bins.

- **One limit-order account can cover up to 50 bins** (`MAX_BIN_PER_LIMIT_ORDER = 50`).
- **Bid side** places **token Y at or below** the active bin to **buy token X** when price falls to
  those bins. **Ask side** places **token X at or above** the active bin to **sell token X for
  token Y** when price rises to those bins. (At the account level, `is_ask = 1` is a sell of X for
  Y; `is_ask = 0` is a buy of X with Y.)
- Placement is by **absolute bin ID** or by **relative offset** from the observed active bin.
  Relative placement carries a **max active-bin slippage guard** — if the live active bin has moved
  too far from the LP's expected active bin, the order is not created (`ExceededBinSlippageTolerance`,
  error 6004), which protects against placing into a stale book on a fast-moving launch pool.
- **Fee share:** when a swap fills both market-maker liquidity and limit-order liquidity, the
  program **allocates 50% of the limit-order portion of the fee to the limit-order participants**,
  with the remainder handled by the pool's fee-split logic. Filled limit-order liquidity and
  market-maker liquidity are **accounted separately, per bin** (ask-side and bid-side fees tracked
  independently), so a limit order is part of the tradable liquidity surface, not a passive
  off-book instruction.
- **Execution is not guaranteed.** An order fills only if swap flow reaches the bin and consumes
  its liquidity; it can sit unfilled indefinitely, fill partially, or fill during a volatile spike.
  Tell LPs to monitor open orders and cancel (`cancel_limit_order`) those that no longer match
  their view, then reclaim the empty account (`close_limit_order_if_empty`).

Product framing to offer: bid-side below price = "buy the dip" or DCA-in; ask-side above price =
"take profit" or DCA-out; split across several bins for a gradual entry or exit.

## Pool function modes are mutually exclusive (this gates farm APR)

A DLMM pool runs in exactly one of two higher-level function modes — **Liquidity Mining XOR Limit
Order — never both at the same time.** A pool that supports LM rewards does not support limit-order
placement, and a pool that supports limit orders does not distribute LM rewards.

This has a consequence that changes reward advice: since the 0.12.0 release, `FunctionType` no
longer defaults to `LiquidityMining`. Every existing `LbPair` **without liquidity-mining rewards
already initialized is defaulted to a limit-order-supported pool and can no longer add LM — and
this is not reversible.** So farm rewards are an increasingly **legacy** consideration: new and
non-LM pools cannot bolt LM on. Before you tell an LP to expect farm APR, confirm the pool is
actually an LM pool (`function_type = 1`, `concrete_function_type = LiquidityMining = 1`, or a
non-empty `farm_apr`/`farm_apy` from the Data API); on a limit-order pool that farm APR structurally
does not exist.

## Liquidity mining mechanics (when a pool is an LM pool)

- A pool can track **up to 2 reward tokens** (`NUM_REWARDS = 2`).
- Rewards reach only liquidity where trading is actually happening: the **active bin**, or **bins
  crossed by a swap**, subject to the program's **15-bin reward-split limit** when a single v2 swap
  moves the active price across many bins. Liquidity sitting far from the market earns nothing.
- **Reward share = your liquidity in eligible bins / total liquidity in eligible bins.** An
  out-of-range position earns **no** rewards; a bin with no liquidity pays **no** rewards — the same
  in-range principle as trading fees.
- Emissions are **fixed over a reward period** and streamed to eligible bins. Worked example from
  the docs: a pool distributing **50,000 USDC over 4 weeks ≈ 1,785.71 USDC/day**, split each moment
  across whatever liquidity is in the eligible (active/crossed) bins.
- If the active bin has **no liquidity** during a period, rewards for that period are **not
  distributed** to LPs; the program accrues that time into
  `cumulative_seconds_with_empty_liquidity_reward`, and **operators can later withdraw those
  ineligible rewards** (`withdraw_ineligible_reward`).
- Rewards **do not auto-compound** — they accrue to the position and must be **claimed manually**
  (`claim_reward` / `claim_reward2`).

Strategy implication: LM rewards only lift returns when your liquidity is active, so a reward
program does not change the core rule that you must keep liquidity near where price trades. For a
token team, it argues for incentivizing **depth around the expected trading zone**, not just total
TVL.

## Delegation and custody

`PositionV2` separates ownership from operation, which is what managed and automated setups rely on:

- **Operator** — the owner can authorize another address (`update_position_operator`, or create the
  position via `initialize_position_by_operator`) to perform position-management actions such as
  modifying liquidity. This is how an LP bot or a fund manager runs a position it does not own.
  **Warn the LP that an operator can perform any position-management action the program authorizes**
  — delegate only to addresses you trust.
- **Fee owner** — a position can carry a separate `fee_owner` that controls **where claimed fees are
  routed**, distinct from the position owner. Common on bootstrap/launch positions where the project
  seeds liquidity but directs fees elsewhere.
- **Permission bits** — `permissionless_operation_bits` (set via `set_permissionless_operation_bits`)
  can make selected operations, notably **fee claiming, permissionless** so a non-owner can trigger
  them. Useful for a keeper that claims fees on a schedule without holding owner authority.

## Locked liquidity (a real launch/bootstrap exit risk)

A position can carry a **`lock_release_point`** — a slot or timestamp (matching the pool's
activation type) before which the program **refuses to remove liquidity**. Attempting to remove or
rebalance locked liquidity early throws **`LiquidityLocked` (error 6055)**. A value of `0` means
immediately unlocked.

This is not academic: launch and bootstrap positions are frequently seeded with a lock so the team
cannot pull liquidity out from under buyers. If an LP is evaluating a launch pool — or is themselves
seeding one — surface the lock explicitly. Capital under a `lock_release_point` is **stuck until the
release point regardless of how far underwater the position goes**, which is a material exit risk to
price in before committing.

## Token-2022 due diligence

DLMM accepts SPL Token-2022 mints, but only a specific set of extensions passes **permissionlessly**;
anything else needs a Meteora-approved **token badge** before a pool can be created. When an LP is
about to provide into a new-token or launch pool, walk this checklist.

**Permissionless (no Meteora approval needed):**

- `TransferFeeConfig`
- `MetadataPointer`
- `TokenMetadata`
- `TransferHook` — **only when both the hook program and the hook authority are revoked** (a live
  custom hook is not permissionless);
- plus `MemoTransfer`, which is an **account-level** extension on token accounts, not a mint
  extension.

**Hard flags that block permissionless use:**

- A mint with a **freeze authority** requires additional review — DLMM does not permissionlessly
  accept mints that can freeze accounts (a freeze authority means the issuer could freeze the pool's
  or an LP's token account).
- The **native Token-2022 mint is rejected** outright (to avoid fragmenting SOL liquidity) — use the
  standard **wrapped SOL** mint for SOL-side liquidity.
- **Any other extension** (authority-based, pausable, confidential, non-transferable, scaled-UI,
  live transfer hooks, etc.) requires a **token badge**: the creator files Meteora's form and opens a
  Discord ticket for review before pool creation. A valid badge lets the mint pass DLMM's mint
  validation despite off-allowlist extensions.

The one that silently costs LPs money is **transfer fees**. On a `TransferFeeConfig` mint, the token
skims a fee on every transfer, so the amount **actually deposited, withdrawn, and claimed is less
than the nominal amount** — transfer fees erode both principal and claimed fees on the way in and
out. Factor that leakage into any yield estimate for a transfer-fee token, and confirm wallets,
routers, bots, and analytics can display and transfer the token before providing. (Unsupported
extensions surface at instruction time as `NotSupportMint`, error 6069, or `UnsupportedMintExtension`,
error 6070.)

## Claiming reality and Collect Fee Mode

Two facts govern the end of a position's life:

- **Nothing auto-compounds.** Accrued fees and LM rewards sit as claimable balances on the position
  until the LP claims them; they never fold back into liquidity on their own. Reinvesting means
  claim, then add liquidity — two deliberate steps.
- **A position closes only after it is fully cleared.** `close_position` succeeds only once all
  liquidity is removed and all fees and rewards are claimed; otherwise it reverts with
  `NonEmptyPosition` (error 6030). The SDK's `removeLiquidity({ ..., shouldClaimAndClose: true })`
  bundles the remove-claim-close sequence, and wide positions may return **more than one
  transaction**.

Which token those fees arrive in is a **pool-level** setting, **Collect Fee Mode**: `InputOnly`
(the default — fees accrue in whichever token was swapped in, so a position can accumulate both X
and Y) or `OnlyY` (single-sided "Quote Token Fee" — the pool collects fees only in token Y). This
determines the *composition* of what an LP claims; the *magnitude* — how the base fee
(`base_factor × bin_step × 10 × 10^base_fee_power_factor`, stored in 1e9 precision), the dynamic
fee, and the protocol cut (typically **10% on standard DLMM pools, 20% on Launch Pools**, per-pool, taken before LPs) —
combine into what LPs actually earn is worked out in `fees-and-economics.md`.
