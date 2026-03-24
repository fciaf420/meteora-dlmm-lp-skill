# Meteora DLMM LP — Claude Code Skill

A Claude Code skill that provides expert advisory on **Meteora DLMM (Dynamic Liquidity Market Maker)** liquidity provision on Solana.

## What It Does

This skill turns Claude into a Meteora DLMM strategist. It provides guidance on:

- **Bin selection & bin steps** — choosing the right granularity for your pair's volatility
- **Liquidity shapes** — Spot, Curve, and Bid-Ask strategies and when to use each
- **Fee optimization** — understanding dynamic fees and maximizing fee capture
- **Impermanent loss management** — practical approaches to IL in concentrated liquidity
- **Position sizing & rebalancing** — when to hold, when to rebalance, and how to think about it
- **Token launch LP strategies** — navigating DLMM Launch Pools with high volatility
- **Single-sided liquidity & DCA** — using DLMM for dollar-cost averaging in/out of positions
- **Live data** — fetches real-time pool data from the Meteora DLMM Data API to ground advice in actual numbers

## Installation

### As a Claude Code Skill

Copy the `SKILL.md` file into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/meteora-dlmm-lp
cp SKILL.md ~/.claude/skills/meteora-dlmm-lp/
```

### As a Cowork Plugin Skill

Place the skill folder inside your plugin's `skills/` directory.

## Usage

Once installed, Claude will automatically activate this skill when you ask about:

- Meteora or DLMM
- Providing liquidity on Solana
- Bin steps, liquidity shapes, or concentrated liquidity strategies
- Impermanent loss in DLMM contexts
- Rebalancing DLMM positions
- Token launch LP strategies

### Example Prompts

- *"I want to LP on a new memecoin/SOL pair on Meteora. What bin step and shape should I use?"*
- *"My SOL/USDC Curve position is at the edge of its range. Should I rebalance?"*
- *"Can I use DLMM to DCA sell my token bag over the next few weeks?"*
- *"What are the best SOL/USDC pools on Meteora right now?"*

## Evals

The `evals/` directory contains test cases to validate the skill's output quality. Run them with Claude Code's skill evaluation tooling.

## License

MIT
