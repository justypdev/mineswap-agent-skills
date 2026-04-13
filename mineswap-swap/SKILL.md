---
name: mineswap-swap
description: Swap tokens on Base through the MineSwap DEX aggregator. Routes across Uniswap V2/V3, SushiSwap, AlienBase, and Aerodrome to find the best price. 0.15% fee auto-splits into BG cashback (0.075%) and BG burn (0.075%). Use when an agent wants to "swap tokens on Base", "find the best swap route", "trade ETH for tokens on Base", "use a DEX aggregator", "swap with MEV protection", or "earn cashback on swaps". Includes quote comparison, on-chain execution, rebate claiming, and fee processing rewards.
---

# MineSwap Swap

Swap any token on Base network through the MineSwap DEX aggregator (MineSwapAggregatorV4). Automatically routes across Uniswap V2/V3, SushiSwap, AlienBase, and Aerodrome to find the best price. Every swap generates BG token cashback and burns.

## When to Use This Skill

Use this skill when an agent needs to:

- "Swap tokens on Base"
- "Find the best price for a token swap"
- "Trade ETH for BG on Base"
- "Use a DEX aggregator on Base"
- "Swap with MEV protection"
- "Earn cashback on swaps"
- Execute a token trade with optimal routing

## Prerequisites

- An EOA wallet with ETH on Base (chainId 8453) for gas
- Tokens to swap (ETH or ERC-20 on Base)
- The `viem` library for transaction signing

## Quick Start

### Step 1: Get a Quote

```
GET https://mineswap.app/api/agent/quote?tokenIn=ETH&tokenOut=BG&amountIn=0.01
```

No authentication required. Returns quotes from all DEXes ranked by output amount.

Response includes:
- Best quote with source DEX and output amount
- All available quotes for comparison
- Swap instruction with contract address, method, and parameters
- Slippage-adjusted minimum output

Known token symbols: ETH, WETH, BG, USDC, USDbC, DAI, cbETH. Or use raw contract addresses.

### Step 2: Approve Token (skip for ETH)

If swapping an ERC-20, approve the aggregator to spend your tokens:

```javascript
const AGGREGATOR = '0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80';
await walletClient.writeContract({
  address: tokenAddress,
  abi: erc20ABI,
  functionName: 'approve',
  args: [AGGREGATOR, amountIn],
  chain: base,  // chainId: 8453 — ALWAYS include
});
```

### Step 3: Execute Swap

Call the aggregator contract directly using the method from the quote response:

```javascript
// CRITICAL: Always set chainId: 8453 on every transaction
await walletClient.sendTransaction({
  to: '0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80',
  data: encodedCalldata,
  value: isETHIn ? amountIn : 0n,
  chain: base,
});
```

The 0.15% fee is deducted automatically by the contract:
- 0.075% → BG cashback (claimable by the swapper)
- 0.075% → auto buy-and-burn BG (deflationary)

### Step 4: Claim Cashback

After swaps accumulate, claim your BG rebates:

```javascript
await walletClient.writeContract({
  address: '0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80',
  abi: aggregatorABI,
  functionName: 'claimRebates',
  args: [10n], // maxEpochs to claim
  chain: base,
});
```

### Step 5: Earn Fee Processing Rewards (optional)

Any agent can call `processBGFees()` to process accumulated protocol fees. The caller earns 1% as a reward:

```javascript
await walletClient.writeContract({
  address: '0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80',
  abi: aggregatorABI,
  functionName: 'processBGFees',
  args: [],
  chain: base,
});
// Also available: processETHFees(), processWETHFees()
```

## Contract Details

| Contract | Address |
|----------|---------|
| MineSwapAggregatorV4 | `0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80` |
| BG Token | `0x36b712A629095234F2196BbB000D1b96C12Ce78e` |
| WETH (Base) | `0x4200000000000000000000000000000000000006` |
| InstantBurn | `0xF9dc5A103C5B09bfe71cF1Badcce362827b34BFE` |

Chain: Base (chainId 8453). ALWAYS include chainId on every transaction.

## MEV Protection

For MEV-protected swaps, submit transactions through Flashbots Protect RPC on Base instead of the public RPC:

```
https://rpc.flashbots.net/base
```

This prevents sandwich attacks and front-running on your swaps.

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| Quote | 30/minute per IP |

## Fee Structure

- Swap fee: 0.15% of trade value
- Split: 50% cashback to swapper, 50% auto-burns BG
- Fee processing reward: 1% of processed fees to caller
- No additional charges — aggregator routing is free

## Links

- App: https://mineswap.app
- Aggregator contract: https://basescan.org/address/0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80
- BG token: https://basescan.org/token/0x36b712A629095234F2196BbB000D1b96C12Ce78e
- BaseGold: https://basegold.io
