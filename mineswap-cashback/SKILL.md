---
name: mineswap-cashback
description: Claim all earned BG from the BaseGold ecosystem — smelter cashback, DEX swap rebates, and fee processing rewards. Use when an agent wants to "claim BG cashback", "claim swap rebates", "check smelter balance", "withdraw BG earnings", "process DEX fees for reward", "claim MineSwap rewards", or "collect BaseGold earnings". Covers both off-chain smelter claims (HMAC-signed) and on-chain aggregator rebate claims.
---

# MineSwap Cashback

Claim all earned BG tokens from the BaseGold ecosystem. There are three revenue streams to claim, each with its own mechanism:

1. **Smelter cashback** — BG from season airdrops, raid winnings, allocations (off-chain, HMAC-signed claims)
2. **DEX swap rebates** — BG earned from the 0.075% cashback on every MineSwap aggregator swap (on-chain, call contract)
3. **Fee processing rewards** — 1% reward for processing accumulated protocol fees (on-chain, permissionless)

## When to Use This Skill

Use this skill when an agent needs to:

- "Claim my BG cashback"
- "Claim swap rebates from MineSwap"
- "Process DEX fees for a reward"
- "Check my smelter balance"
- "Withdraw my BG earnings"
- "Collect all BaseGold rewards"
- Harvest accumulated BG from all sources

## Prerequisites

- A Gold Mine NFT on Base (chainId 8453) — for smelter claims
- An authenticated session (Bearer token from `/api/war-auth`) — for smelter claims
- A wallet with ETH on Base for gas — for on-chain rebate claims
- The `viem` library for contract interactions

## Revenue Stream 1: Smelter Cashback (Off-Chain)

BG accumulates in your smelter from season airdrops, raid steals, and allocations. Each batch smelts for 30 days before becoming claimable.

### Check Smelter Status

```
POST https://mineswap.app/api/agent/claim
Authorization: Bearer <token>
{ "action": "status" }
```

Returns claimable batches (with HMAC tickets) and smelting batches (still locked).

### Claim Smelted BG

Use the `batchId` and `hmac` from the status response:

```
POST https://mineswap.app/api/agent/claim
Authorization: Bearer <token>
{ "action": "claim", "batchId": "alloc_1234_abc", "hmac": "a1b2c3d4e5f6..." }
```

Claims are queued for admin payout via Disperse.app batch.

### Quick-Check via State

```
GET https://mineswap.app/api/agent/state?include=smelter
Authorization: Bearer <token>
```

## Revenue Stream 2: DEX Swap Rebates (On-Chain)

Every swap through the MineSwap aggregator accumulates 0.075% cashback in BG for the swapper. Claim it by calling `claimRebates` on the aggregator contract.

### Claim Swap Rebates

```javascript
import { createWalletClient, http } from 'viem';
import { base } from 'viem/chains';

const AGGREGATOR = '0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80';
const AGGREGATOR_ABI = [
  {
    name: 'claimRebates',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [{ name: 'maxEpochs', type: 'uint256' }],
    outputs: [{ name: 'totalAmount', type: 'uint256' }],
  },
  {
    name: 'totalBGRebated',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }],
  },
];

// Claim accumulated rebates (up to 10 epochs at a time)
const tx = await walletClient.writeContract({
  address: AGGREGATOR,
  abi: AGGREGATOR_ABI,
  functionName: 'claimRebates',
  args: [10n],
  chain: base,  // CRITICAL: Always include chainId 8453
});
```

The claimed BG goes directly to the caller's wallet — no intermediary, no 30-day lock.

## Revenue Stream 3: Fee Processing Rewards (On-Chain)

Anyone can call `processBGFees()`, `processETHFees()`, or `processWETHFees()` on the aggregator to process accumulated protocol fees. The caller earns 1% of the processed amount as a reward.

```javascript
const FEE_ABI = [
  { name: 'processBGFees', type: 'function', stateMutability: 'nonpayable', inputs: [], outputs: [] },
  { name: 'processETHFees', type: 'function', stateMutability: 'nonpayable', inputs: [], outputs: [] },
  { name: 'processWETHFees', type: 'function', stateMutability: 'nonpayable', inputs: [], outputs: [] },
  { name: 'processTokenFees', type: 'function', stateMutability: 'nonpayable',
    inputs: [{ name: 'token', type: 'address' }, { name: 'router', type: 'address' }], outputs: [] },
  { name: 'ethFeesCollected', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ name: '', type: 'uint256' }] },
  { name: 'feesCollected', type: 'function', stateMutability: 'view',
    inputs: [{ name: 'token', type: 'address' }], outputs: [{ name: '', type: 'uint256' }] },
];

// Check if fees have accumulated
const ethFees = await publicClient.readContract({
  address: AGGREGATOR, abi: FEE_ABI,
  functionName: 'ethFeesCollected',
});

// Process fees and earn 1% reward
if (ethFees > 0n) {
  await walletClient.writeContract({
    address: AGGREGATOR, abi: FEE_ABI,
    functionName: 'processETHFees', args: [],
    chain: base,
  });
}
```

This is competitive — multiple agents may race to process fees. First caller wins the reward.

## Optimal Harvest Loop

```javascript
// Every hour: check smelter + claim rebates + process fees
setInterval(async () => {
  // 1. Smelter claims (off-chain)
  const status = await POST('/api/agent/claim', { action: 'status' });
  for (const batch of status.claimable) {
    await POST('/api/agent/claim', { action: 'claim', batchId: batch.batchId, hmac: batch.hmac });
  }

  // 2. DEX rebates (on-chain)
  try {
    await walletClient.writeContract({
      address: AGGREGATOR, abi: AGGREGATOR_ABI,
      functionName: 'claimRebates', args: [10n], chain: base,
    });
  } catch (e) { /* No rebates to claim */ }

  // 3. Fee processing reward (on-chain, competitive)
  try {
    const ethFees = await publicClient.readContract({
      address: AGGREGATOR, abi: FEE_ABI, functionName: 'ethFeesCollected',
    });
    if (ethFees > 100000000000000n) { // Only if >0.0001 ETH accumulated
      await walletClient.writeContract({
        address: AGGREGATOR, abi: FEE_ABI,
        functionName: 'processETHFees', args: [], chain: base,
      });
    }
  } catch (e) { /* Another agent processed first */ }
}, 3600000);
```

## Contract Reference

| Contract | Address | Use |
|----------|---------|-----|
| MineSwapAggregatorV4 | `0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80` | Rebates + fee processing |
| BG Token | `0x36b712A629095234F2196BbB000D1b96C12Ce78e` | Earned token |
| InstantBurn | `0xF9dc5A103C5B09bfe71cF1Badcce362827b34BFE` | Shop burn target |

Chain: Base (chainId 8453). ALWAYS include chainId on every transaction.

## Security

- Smelter claims use HMAC-signed tickets — unforgeable without server secret
- Monotonic claimed flags prevent replay
- DEX rebates are per-wallet on-chain — can only claim your own
- Fee processing is permissionless by design — first caller wins
- All state backed up to Supabase for audit trail

## Links

- App: https://mineswap.app
- Aggregator: https://basescan.org/address/0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80
- BG token: https://basescan.org/token/0x36b712A629095234F2196BbB000D1b96C12Ce78e
- BaseGold: https://basegold.io
