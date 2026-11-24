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