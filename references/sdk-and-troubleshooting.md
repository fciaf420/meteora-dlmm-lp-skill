# DLMM SDK & Troubleshooting Reference

Read this when the LP wants to **execute** a position change through the TypeScript SDK or the on-chain program, or when a **transaction failed** and you need to decode a `custom program error: 0x…` code. The advisory `SKILL.md` stays strategy-only on purpose — this file is the one hop away that carries the program IDs, constants, account model, SDK function names, and error table. Pull the exact identifier from here rather than paraphrasing from memory; these are load-bearing and an LP will paste them straight into code or a block explorer.

## Program identity

- On-chain program: **`lb_clmm`**, program ID **`LBUZKhRxPF3XUpBCjp4YzTKgLccjZhTSDM9YuVaPwxo`** — the same ID on **mainnet-beta and devnet**. The SDK defaults to it; pass `{ cluster: "devnet" }` (or a custom `programId`) to target another deployment.
- The program **source is closed**. Integrate through: the published Anchor **IDL** (`github.com/MeteoraAg/dlmm-sdk/blob/main/idls/dlmm.json`), the **`@meteora-ag/dlmm`** TypeScript SDK, direct reads of on-chain account data, and the **DLMM Data API**. There is no source to audit, so treat the IDL + SDK as the contract.

## Core constants

Quote these exactly — they are the precise on-chain constants behind the rounded figures used elsewhere in this skill.

| Constant | Value | Meaning |
|---|---|---|
| `MAX_BIN_PER_ARRAY` | **70** | Bins per `BinArray` account |
| `DEFAULT_BIN_PER_POSITION` | **70** | Bins stored inline in `PositionV2` before extension |
| `MAX_BIN_PER_LIMIT_ORDER` | **50** | Max bins one limit-order account can span |
| `BIN_ARRAY_BITMAP_SIZE` | **512** | Default bitmap covers bin-array indexes **−512 … 511** |
| `DEFAULT_OBSERVATION_LENGTH` | **100** | Oracle sample capacity (extendable) |
| `NUM_REWARDS` | **2** | Reward slots per pool and per position |

Bin-array index for a bin ID = **`floor(bin_id / 70)`** (each array covers 70 consecutive bin IDs, `index×70 … index×70+69`). The advisory skill's older "69" is the UI's default *bin count minus one* confusion — the array size and default position width are both **70**. A default position holds **70 bins inline**; the max supported position length is **1,400 bins**, and any single resize may add at most **91 bins** beyond the default layout.

## Account model

- **`PositionV2`** is the current position account. It stores the fixed 70-bin layout inline (`liquidity_shares[70]`, `fee_infos[70]`, `reward_infos[70]`) and, for wider ranges, appends `PositionBinData` entries after the header. You do **not** get an LP token or NFT — the position account itself is the record.
- Grow/shrink in place with **`increase_position_length` / `decrease_position_length`** (and their v2 variants) — no close required. Decrease only removes **empty** bins.
- PDA seeds use **`width` (bin count), not `upper_bin_id`**: `derivePosition` = `["position", lb_pair, base, lower_bin_id, width]`. Re-deriving with `upper_bin_id` yields the wrong address.
- Legacy **`Position`** uses a smaller share integer type; migrate with `migrate_position_from_v1` / `migrate_position_from_v2`. New positions use `initialize_position` / `initialize_position2`.
- **`lock_release_point`** (slot or timestamp) gates locked launch liquidity — removing before it fails with `LiquidityLocked` (6055). **`fee_owner`** can redirect where claimed fees land (bootstrap positions).

## Bin-array cost mechanics — why wide/far positions cost more

This is the on-chain backing for range-width guidance:

- **`initialize_bin_array`** creates the array *and* initializes 70 bin prices — it is **compute-heavy**. If you build the tx manually (not via the SDK), prepend a compute-budget instruction or it will exceed the default limit. It also pays **non-refundable SOL rent** for each new bin array your range touches — a real, unrecoverable cost distinct from the refundable position/extension rent.
- The pool's own bitmap only tracks bin arrays in indexes **−512 … 511**. Operating on anything farther out requires the **`BinArrayBitmapExtension`** account (PDA `["bitmap", lb_pair]`); omitting it returns **`BitmapExtensionAccountIsNotProvided` (6036)**.
- Net effect: a wide or far-from-active position touches more (possibly uninitialized) bin arrays → more accounts, more compute, more non-refundable rent, and a higher chance of hitting the bitmap-extension or compute-budget walls. That's the concrete reason to keep ranges no wider than the strategy needs.

## Key SDK entry points (`@meteora-ag/dlmm`)

Load a pool with `DLMM.create(connection, poolAddress, { cluster })`; batch with `DLMM.createMultiple([...])`.

- **`initializePositionAndAddLiquidityByStrategy({ positionPubKey, user, totalXAmount, totalYAmount, strategy: { minBinId, maxBinId, strategyType } })`** — create + fund in one tx. `StrategyType.Spot | Curve | BidAsk` over `minBinId … maxBinId`. The SDK auto-initializes the bin arrays and token accounts the range needs. Add to an existing position with `addLiquidityByStrategy` (use the `...Chunkable` variant for wide ranges).
- **`increase_position_length` / `decrease_position_length`** — resize the range in place (the ≤91-bins-per-instruction cap applies; chunk larger growth).
- **`rebalance_liquidity` / `rebalance_position`** — the atomic primitive that folds **claim + remove + resize + add-liquidity + shrink** into one position-management flow, with selectable shrink modes **`ShrinkBoth`, `NoShrinkLeft`, `NoShrinkRight`, `NoShrinkBoth`**. This is why "rebalance" no longer implies close-and-reopen.
- **Limit orders** (`function_type` = 2 pools only; LM and limit-order modes are mutually exclusive): `placeLimitOrder` / `cancelLimitOrder` / `quoteCreateLimitOrder`, backed by `place_limit_order` / `cancel_limit_order` / `close_limit_order_if_empty`. `is_ask=1` sells token X for Y (ask side, bins at/above active); `is_ask=0` buys X with Y (bid side, bins at/below active). **≤50 bins per order account.**
- **Fees & rewards:** `claimSwapFee` (one position), `claimAllSwapFee` (wallet-wide), `claimAllRewardsByPosition`, `claimAllLMRewards`, backed by `claim_fee` / `claim_reward`. Fees/rewards never auto-compound — they stay claimable until claimed.
- **`removeLiquidity({ position, user, fromBinId, toBinId, bps, shouldClaimAndClose })`** — **returns one OR MORE transactions** (wide positions chunk, so iterate and send all of them). `bps: 10_000` = remove 100%; `shouldClaimAndClose: true` claims fees/rewards and closes the account in the same flow.
- **Portfolio/batch:** `DLMM.getAllLbPairPositionsByUser(connection, user)` for every position across pools; `DLMM.createMultiple` to load many pools at once.

## Fee & collect-mode flags (on-chain)

- **`protocol_share`** (basis points, on `StaticParameters`) is skimmed from the total swap fee **before** LPs receive anything — 10% on standard DLMM pools, 20% on Launch Pools. Compute LP fee APR net of this cut (Data API `fees` / `apr` already are; only a self-computed `volume × fee_rate` needs it).
- **`collect_fee_mode`**: `0` = fees collected in the **input token(s)** (either side possible); `1` = fees collected **only in token Y**. This decides which token accrued fees arrive in.
- Each bin stores `amount_x` / `amount_y` **already net of protocol fees**, and fees accrue **per bin** (`fee_amount_x/y_per_token_stored`) — bins price never trades through earn nothing.

## Launch / lock flags (on `LbPair`)

- **`activation_type`** — activation measured in slot or timestamp. **`activation_point`** — when a permission pool opens trading. **`pre_activation_duration`** — window before activation during which one designated wallet may swap. **`pre_activation_swap_address`** — that single pre-activation wallet (a launch front-running vector to flag).
- Restricted launch/creation flows require the **quote token = SOL or USDC**; anything else fails with **`InvalidQuoteToken` (6061)**.

## Pool status & type flags

- **`status`**: `0` enabled / `1` disabled. A disabled pool is **withdraw-only except for whitelisted wallets** — you can pull liquidity but adds/swaps fail with **`PoolDisabled` (6042)**. A pool you're in *can* be disabled; that's real tail risk.
- **`pair_type`**: `0` permissionless / `1` permission / `2` customizable permissionless / `3` permissionless v2 (Token-2022).

## LP-facing error table (decode a failed tx)

Anchor custom errors start at **6000**. Logs show `custom program error: 0x<hex>` — convert hex → decimal (e.g. `0x1774` = 6004). The subset an LP will actually hit:

| Code | Name | LP-visible cause → fix |
|---|---|---|
| **6004** | `ExceededBinSlippageTolerance` | Active bin moved outside the allowed range mid-instruction (common on fast/launch pools). Refresh the quote **and** active bin, or widen bin slippage. |
| **6030** | `NonEmptyPosition` | Tried to close/shrink a position that still holds liquidity, fees, or rewards. **Remove + claim first**, then close. |
| **6036** | `BitmapExtensionAccountIsNotProvided` | Range touches a bin array outside −512…511. Pass the bitmap-extension account. |
| **6042** | `PoolDisabled` | Pool `status` = disabled; add/swap blocked. Only withdrawal is allowed. |
| **6054** | `InvalidStrategyParameters` | Strategy params inconsistent, out of range, or incompatible with the active bin. Recompute `minBinId/maxBinId` around the live active bin. |
| **6055** | `LiquidityLocked` | Remove/rebalance attempted before `lock_release_point`. Wait for the lock to release. |
| **6061** | `InvalidQuoteToken` | Restricted flow used a non-SOL/USDC quote token ("Quote token must be SOL or USDC"). |

Adjacent codes worth knowing when they surface: `6003 ExceededAmountSlippageTolerance` (refresh quote/widen tolerance), `6017 InvalidBps` (bps > 10000), `6086 ReallocExceedMaxLengthPerInstruction` (chunk the resize). The full 6000–6110 table lives on the program errors page.

## Variable-fee accumulator internals

The dynamic fee rides on `VariableParameters.volatility_accumulator`: it **rises as the active bin moves**, then **decays** over time per `filter_period` / `decay_period` / `reduction_factor`, and is hard-**capped by `max_volatility_accumulator`**. `variable_fee_control` scales it into the fee. So the "surge pricing" the advisory layer describes is literally this accumulator climbing during rapid bin crossings and bleeding off when trading calms — see `fees-and-economics.md` for how it combines with the base fee (`base_factor × bin_step × 10 × 10^base_fee_power_factor`, stored in 1e9 precision).

## Doc links (dev hop lands live)

- DLMM TypeScript SDK Reference: https://docs.meteora.ag/developer-guides/dlmm/typescript-sdk/reference (siblings: `getting-started`, `examples`)
- DLMM Data API Overview: https://docs.meteora.ag/developer-guides/dlmm/api-reference/overview (individual endpoints under `https://docs.meteora.ag/api-reference/dlmm/<group>/<page>`)

The old `developer-guide/guides/...` SDK path and the `api-reference/dlmm/overview` path both 404 — use the ones above.
