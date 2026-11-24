# Wickproof

**Manipulation-resistant gold (`XAU-PERP`) and silver (`XAG-PERP`) perpetual futures on Solana, powered by an on-chain order book on MagicBlock Ephemeral Rollups.**

[![Solana](https://img.shields.io/badge/Solana-Devnet-9945FF?logo=solana&logoColor=white)](https://solana.com)
[![Anchor](https://img.shields.io/badge/Anchor-Framework-blue)](https://www.anchor-lang.com/)
[![MagicBlock](https://img.shields.io/badge/MagicBlock-Ephemeral_Rollup-black)](https://www.magicblock.gg/)
[![Pyth](https://img.shields.io/badge/Oracle-Pyth_Network-purple)](https://pyth.network/)

---

## Overview

Traditional perpetual exchanges rely on a single oracle price to trigger liquidations—meaning a single bad print or flash wick can unfairly liquidate healthy positions. 

**Wickproof** solves this by computing the **Mark Price** as the median of three independent sources:
1. **Pyth Oracle Price** (`XAU/USD` or `XAG/USD`)
2. **Order Book Midpoint** (halfway between best bid and best ask)
3. **Last Trade Price**

Because the median of three values is always used, **one wrong price is always outvoted**. Meanwhile, custody of USDC remains strictly on Solana Layer 1 while high-speed, zero-fee order matching and liquidations execute inside a **MagicBlock Ephemeral Rollup (ER)**.

---

## Architecture & Execution Flow

Wickproof separates **asset custody (Solana L1)** from **trade execution (MagicBlock ER)**:
### User Lifecycle
1. **Deposit (`IX-03`):** Deposit USDC into the market's Solana vault to credit your `TraderAccount`.
2. **Delegate (`IX-04`):** Delegate your `TraderAccount` onto the MagicBlock Ephemeral Rollup.
3. **Trade (`IX-05`, `IX-06`):** Place or cancel limit orders (`XAU-PERP` or `XAG-PERP`) with up to **10x leverage**. Orders match best-price-first, then oldest-first (FIFO), up to 8 fills per instruction.
4. **Undelegate (`IX-08`):** Once all open positions are closed and resting orders are cancelled, commit state and undelegate your account back to Solana.
5. **Withdraw (`IX-09`):** Withdraw your updated USDC balance from the Solana vault.

---

## Core Mechanisms & Risk Engine

### 1. Three-Price Median Mark (`REQ-27`)
To prevent oracle-wick liquidations, both margin checks and liquidations evaluate positions against the median of `[oracle_price, book_midpoint, last_trade_price]` (falling back to the oracle price if the book or last trade is uninitialized).
* *Example:* If gold trades at `$1,800` (book midpoint `$1,800`, last trade `$1,801`) and Pyth glitches to `$1,700`, the mark price resolves to **`$1,800`**, protecting traders from false liquidation.

### 2. Margin & Trading Guardrails
| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **Max Leverage / Initial Margin** | `10x` (`1000 bps` / 10%) | Equity must cover 10% of max potential position value including open orders (`REQ-19`). |
| **Maintenance Margin** | `5%` (`500 bps`) | Positions with `equity < position_value * 5%` become eligible for liquidation (`REQ-29`). |
| **Liquidation Penalty** | `1%` (`100 bps`) | Paid from the liquidated trader's remaining collateral to the liquidator (`REQ-34`). |
| **Price Band** | `±2%` (`200 bps`) | Orders outside ±2% of the oracle price are rejected to prevent book manipulation (`REQ-20`). |
| **Oracle Staleness Limit** | `60 seconds` | Orders and liquidations pause if Pyth data is stale (e.g., weekend market closures) (`REQ-21`, `REQ-30`). |
| **Order Book Capacity** | `32 orders / side` | Bounded on-chain order book with self-trade prevention (`REQ-22`, `REQ-23`). |

### 3. Permissionless Liquidation & Bad Debt Accounting (`UC-07`)
* **Position Takeover:** Any trader with sufficient initial margin can liquidate an unhealthy account, assuming the target's entire position at the mark price (`REQ-28`, `REQ-31`, `REQ-32`).
* **Order Cleanup:** All resting orders belonging to the liquidated trader are automatically cancelled (`REQ-35`).
* **Transparent Bad Debt:** If an extreme price gap (e.g., Monday market open) causes losses exceeding a trader's collateral, collateral is floored at `0` and the shortfall is recorded openly in `market.bad_debt` (`REQ-36`), preserving the global accounting invariant: `vault_balance + bad_debt >= sum(trader_collateral)` (`REQ-45`).

---

## Program Instructions

| ID | Instruction | Execution Layer | Signer | Description |
| :--- | :--- | :--- | :--- | :--- |
| **IX-01** | `initialize_market` | Solana L1 | Admin | Creates `Market`, empty `OrderBook`, and USDC `Vault` PDA. |
| **IX-02** | `delegate_market` | Solana L1 | Admin | Delegates `Market` and `OrderBook` PDAs to MagicBlock ER. |
| **IX-03** | `deposit` | Solana L1 | Trader | Transfers USDC into the vault and credits `TraderAccount`. |
| **IX-04** | `delegate_trader` | Solana L1 | Trader | Delegates `TraderAccount` PDA to MagicBlock ER. |
| **IX-05** | `place_order` | MagicBlock ER | Trader | Matches incoming limit order against the book (up to 8 fills) and rests remainder. |
| **IX-06** | `cancel_order` | MagicBlock ER | Trader | Removes an open order from the book (allowed even when oracle is stale). |
| **IX-07** | `liquidate` | MagicBlock ER | Liquidator | Transfers an unhealthy position to the liquidator at mark price and collects 1% fee. |
| **IX-08** | `undelegate_trader` | MagicBlock ER | Trader | Commits state and returns `TraderAccount` to Solana (requires 0 positions & 0 orders). |
| **IX-09** | `withdraw` | Solana L1 | Trader | Transfers available USDC collateral from the vault back to the trader's wallet. |

---

## On-Chain Account State

* **`Market` (`["market", name]`):** Stores admin pubkey, Pyth `feed_id`, margin/risk parameters (in bps), `last_trade_price`, `bad_debt`, and `next_order_id`.
* **`OrderBook` (`["book", market]`):** Fixed-size arrays for `bids` and `asks` (`MAX_ORDERS_PER_SIDE = 32`), sorted by price-time priority.
* **`TraderAccount` (`["trader", market, owner]`):** Tracks `collateral` (6 decimals), signed `position` (in lots of `0.001 oz`), `entry_price`, and open order lot exposure.
* **`Vault` (`["vault", market]`):** SPL Token account holding USDC collateral on Solana L1.

---

## Testing & Verification

Wickproof is built with a requirement-driven test suite covering all 50 specification requirements (`REQ-01` through `REQ-50`):

* **Unit & Integration Tests (LiteSVM):** Deterministic tests mapped directly to requirement IDs (e.g., `req_27_mark_ignores_one_bad_price`, `req_36_monday_gap_bad_debt`) using mocked Pyth price accounts to simulate flash crashes, stale weekend feeds, and Monday open gaps.
* **Rollup E2E Tests (MagicBlock Devnet):** Verifies cross-layer delegation, order matching, and state commitment (`REQ-05`, `REQ-09`, `REQ-40`).
* **Stateful Fuzz Testing (`REQ-45` – `REQ-50`):** Randomized sequences of deposits, orders, cancels, price shocks, liquidations, and withdrawals asserting core invariants:
  * `vault + bad_debt >= total_collateral`
  * `trader_collateral >= 0`
  * `sum(long_positions) == sum(short_positions)`
  * `best_bid < best_ask`

### Running Tests

```bash
# Build the Anchor program
anchor build

# Run LiteSVM unit and invariant fuzz tests
cargo test

# Run MagicBlock devnet integration tests
anchor test --skip-local-validator