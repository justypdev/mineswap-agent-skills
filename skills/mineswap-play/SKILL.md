---
name: mineswap-play
description: Play the BaseGold mining game as an AI agent. Buy a Gold Mine NFT, mine gold, purchase upgrades, send expeditions, train armies, raid other mines, and earn BG token cashback. Use when an agent wants to "play MineSwap", "mine gold on Base", "buy a Gold Mine NFT", "earn BG tokens through gameplay", "raid other mines", "start an expedition", or "interact with the BaseGold ecosystem". Covers the full gameplay loop from NFT purchase through competitive PvP mining.
---

# MineSwap Play

Play the BaseGold mining game (mineswap.app) as an autonomous AI agent. Mine gold, buy upgrades, send expeditions, train armies, and raid other players — all through a server-authoritative REST API with the same anti-cheat protections as human players.

## When to Use This Skill

Use this skill when an agent needs to:

- "Play the MineSwap mining game"
- "Mine gold and earn BG tokens"
- "Buy a Gold Mine NFT on Base"
- "Raid other mines for BG"
- "Start an expedition in MineSwap"
- "Upgrade my mine"
- Participate in the BaseGold deflationary ecosystem
- Earn passive income through DeFi gaming

## Prerequisites

- An EOA wallet with a private key (for signing)
- ETH on Base network (chainId 8453) for gas
- 0.10 BG tokens to mint a Gold Mine NFT (current cost ~$1.70 at $17/BG)
- The `viem` library for wallet operations

## Quick Start

### Step 1: Buy a Gold Mine NFT

Send 0.10 BG to the Gold Vein contract to mint your mine. This is an on-chain transaction — the agent needs BG tokens and ETH for gas.

- Gold Vein contract: `0x0520B1D4dF671293F8b4B1F52dDD4f9f687Fd565`
- BG token: `0x36b712A629095234F2196BbB000D1b96C12Ce78e`
- Chain: Base (8453)
- CRITICAL: Always set `chainId: 8453` on every transaction

### Step 2: Authenticate

```
// 1. Sign a session message
const timestamp = Date.now();
const message = `BaseGold Miner Session\n${address.toLowerCase()}\n${timestamp}`;
const signature = await walletClient.signMessage({ message });

// 2. Create session
POST https://mineswap.app/api/session
{ "action": "create", "address": "0x...", "signature": "...", "message": "...", "timestamp": ..., "deviceInfo": "MineSwap Agent v1.0" }

// 3. Get Bearer token (verifies NFT ownership on-chain)
POST https://mineswap.app/api/war-auth
{ "wallet": "0x...", "tokenId": YOUR_MINE_NUMBER }
// Response: { "token": "bearer-token-here", "expiresIn": 86400 }
```

All subsequent API calls require: `Authorization: Bearer <token>`

### Step 3: Initialize Game State

```
GET https://mineswap.app/api/agent/state
// Auto-creates initial save if first time. Returns full game state.
```

### Step 4: Mine Gold

```
POST https://mineswap.app/api/agent/mine
{ "clicks": 25 }
// Server computes gold from your upgrades. Max 150 clicks/batch, 5s cooldown.
// Response: { "goldEarned": 1125, "gold": 50000, "level": 5, "goldPerSecond": 12 }
```

### Step 5: Buy Upgrades

```
POST https://mineswap.app/api/agent/upgrade
{ "upgradeId": "miner", "count": 5 }
// Server validates gold balance, level requirement, and maxOwned cap.
```

Available upgrades (in unlock order): pickaxe, miner, drill, geologist, dynamite, deepShaft, luckyStrike, goldmine, refinery, tunnelBorer, goldBoost, motherLode, quantumDrill, voidExtractor, cosmicForge.

### Step 6: Send Expeditions

```
POST https://mineswap.app/api/agent/expedition
{ "action": "start", "type": "shallow" }
// Types: shallow (1 day), deep (3 days), legendary (7 days)
// Claim after duration: { "action": "claim" }
```

### Step 7: Raid Other Mines

```
// Find targets
POST https://mineswap.app/api/agent/raid
{ "action": "targets" }

// Deploy troops (requires trained army)
POST https://mineswap.app/api/agent/raid
{ "action": "deploy", "targetTokenId": 15, "deployArmy": { "scout": 5, "soldier": 3 }, "raidType": "gold" }
// Limits: 5 raids/day, 1 per target per 24h
```

## Optimal Agent Loop

```javascript
// Mine every 10 seconds
setInterval(() => POST('/api/agent/mine', { clicks: 25 }), 10000);

// Auto-upgrade every 30 seconds (buy cheapest affordable)
setInterval(async () => {
  const state = await GET('/api/agent/state');
  const affordable = findCheapestAffordable(state.upgrades, state.gold);
  if (affordable) POST('/api/agent/upgrade', { upgradeId: affordable });
}, 30000);

// Manage expeditions every minute
setInterval(async () => {
  const exp = await POST('/api/agent/expedition', { action: 'status' });
  if (!exp.expedition) POST('/api/agent/expedition', { action: 'start', type: 'shallow' });
  else if (exp.expedition.isComplete) POST('/api/agent/expedition', { action: 'claim' });
}, 60000);
```

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| mine | 12 calls/minute |
| upgrade | Standard write limit |
| expedition | Standard write limit |
| raid targets | 10/minute |
| raid deploy | 5/day + 1/target/24h |
| state | 60/minute |

## Security

- All gold is computed server-side from upgrade state — agent cannot spoof
- HMAC integrity on every save — tamper detection
- withLock on all mutations — no race condition exploits
- Browser session conflict check — can't play agent + browser simultaneously
- On-chain NFT ownership verified before session creation

## Revenue Streams from Playing

1. **Season airdrops**: 0.2% of BG supply distributed to mine holders each season
2. **Smelter cashback**: BG earned from swap fee rebates (30-day smelt lock)
3. **Raid winnings**: Steal BG from other mines' smelters
4. **PvP match stakes**: Wager mBG in competitive matches
5. **Referral income**: 7-level deep referral cuts via Gold Vein (5% of 0.10 BG per referred mine)

## Links

- App: https://mineswap.app
- API docs: https://mineswap.app/docs/AGENT_API.md
- BaseGold: https://basegold.io
- BG token: https://basescan.org/token/0x36b712A629095234F2196BbB000D1b96C12Ce78e
- Gold Mine NFT: https://basescan.org/address/0x4F8f97e10E2D89Bc118d6fdfe74d1C96A821E4e3
