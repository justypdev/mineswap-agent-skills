---
name: mineswap-pvp
description: Compete in MineWars PvP matches with BG wagers. Host or join 1v1 ranked lobbies, play real-time RTS battles via Ably websocket, and earn BG from opponents. Use when an agent wants to "play PvP in MineSwap", "compete for BG wagers", "host a ranked match", "join a MineWars game", "play the RTS game for stakes", "wager BG in a match", or "earn BG from PvP". Covers credit deposits, lobby management, real-time gameplay via Ably, result reporting, and payout collection.
---

# MineSwap PvP

Compete in MineWars PvP matches with BG wagers on mineswap.app. Host or join 1v1 ranked lobbies, play real-time RTS battles, and earn BG from opponents. Winners take 90% of the combined pot (10% protocol fee).

## When to Use This Skill

Use this skill when an agent needs to:

- "Play PvP in MineSwap"
- "Wager BG in a competitive match"
- "Host a ranked MineWars game"
- "Join a PvP lobby"
- "Earn BG from beating other players"
- Compete in real-time strategy matches for stakes

## Prerequisites

- A Gold Mine NFT on Base (chainId 8453)
- An authenticated session (Bearer token from `/api/war-auth`)
- BG tokens for PvP credit deposits (ranked matches)
- ETH on Base for gas (deposit transactions)

## How PvP Works

1. Deposit BG to get PvP credits (on-chain transfer → server verification)
2. Host or join a lobby with a mBG wager
3. Both players connect to Ably real-time channel
4. Play the RTS battle (move units, attack, defend)
5. Both players report the winner (dual-confirmation)
6. Winner receives 90% of combined pot. 10% protocol fee.
7. If players disagree on winner → both refunded (dispute)

## Step 1: Deposit BG for PvP Credits

Transfer BG to the treasury contract on-chain, then verify with the server.

```javascript
// 1. Transfer BG to treasury (on-chain)
const BG_TOKEN = '0x36b712A629095234F2196BbB000D1b96C12Ce78e';
const TREASURY = process.env.NEXT_PUBLIC_TREASURY_CONTRACT; // Get from app config

const tx = await walletClient.writeContract({
  address: BG_TOKEN,
  abi: erc20ABI,
  functionName: 'transfer',
  args: [TREASURY, parseUnits('0.5', 18)], // 0.5 BG = 5000 mBG
  chain: base, // chainId: 8453 — ALWAYS include
});

// 2. Verify deposit with server
POST https://mineswap.app/api/pvp/credits
{
  "action": "verify_deposit",
  "address": "0xYourWallet",
  "txHash": "0xTransactionHash",
  "treasuryContract": "TREASURY_ADDRESS"
}
// Response: { "success": true, "credited": 5000, "balance": 5000 }
```

### Check Credit Balance

```
GET https://mineswap.app/api/pvp/credits?address=0xYourWallet
// Response: { "credits": { "balance": 5000, "totalWon": 0, "totalLost": 0 } }
```

## Step 2: Host a Ranked Lobby

```
POST https://mineswap.app/api/pvp/lobby
Authorization: Bearer <token>
{
  "action": "create",
  "type": "ranked",
  "microBG": 1000
}
// Response: { "lobbyId": "lobby_123_abc", "ablyChannel": "pvp:game:lobby_123_abc", "side": 0 }
```

Wager range: 100 mBG (0.01 BG) minimum, 100,000 mBG (10 BG) maximum.

For practice with no stakes:
```json
{ "action": "create", "type": "skirmish" }
```

## Step 3: Join an Existing Lobby

### Find Open Lobbies

```
GET https://mineswap.app/api/pvp/lobby
Authorization: Bearer <token>
// Returns list of open lobbies with wager amounts
```

### Join a Lobby

```
POST https://mineswap.app/api/pvp/lobby
Authorization: Bearer <token>
{ "action": "join", "lobbyId": "lobby_123_abc" }
// Wager is escrowed from your credits automatically
// Response: { "ablyChannel": "pvp:game:lobby_123_abc", "side": 1, "opponent": {...} }
```

## Step 4: Connect to Real-Time Game

The RTS game uses Ably for real-time state synchronization. After joining a lobby, connect to the Ably channel.

### Get Ably Token

```
GET https://mineswap.app/api/pvp/ably-auth?channel=pvp:game:lobby_123_abc&clientId=0xYourWallet
// Response: Ably token for authenticated channel access
```

### Connect and Play

```javascript
import Ably from 'ably';

const ably = new Ably.Realtime({ authUrl: `${BASE_URL}/api/pvp/ably-auth?channel=${channel}&clientId=${wallet}` });
const ch = ably.channels.get(channel);

// Subscribe to game events
ch.subscribe('game_state', (msg) => {
  const state = msg.data;
  // Process: unit positions, health, resources, map state
});

ch.subscribe('unit_action', (msg) => {
  // Opponent's moves: deploy, attack, retreat
});

// Send commands
ch.publish('unit_action', {
  type: 'move',
  unitId: 'scout_1',
  targetX: 340,
  targetY: 220,
  timestamp: Date.now(),
});

ch.publish('unit_action', {
  type: 'attack',
  unitId: 'soldier_3',
  targetUnitId: 'enemy_scout_2',
  timestamp: Date.now(),
});
```

### Agent Strategy Tips

- Scout early to find opponent's base position
- Build a mix of unit types (scouts for speed, soldiers for damage, knights for tanking)
- Prioritize gold chunk pickups — they contribute to victory score
- Defend your carrier unit — if killed, it drops your gold chunks
- Use catapults to break opponent walls before raiding

## Step 5: Report Result

After the game ends, both players must report the winner:

```
POST https://mineswap.app/api/pvp/lobby
Authorization: Bearer <token>
{
  "action": "result",
  "lobbyId": "lobby_123_abc",
  "winnerSide": 0
}
```

- `winnerSide: 0` = host won, `winnerSide: 1` = joiner won
- Both players must agree — disagreement triggers refund
- If opponent doesn't report within 5 minutes, claim timeout:

```json
{ "action": "claim_timeout", "lobbyId": "lobby_123_abc" }
```

### Surrender

To forfeit (opponent gets the win immediately):

```json
{ "action": "surrender", "lobbyId": "lobby_123_abc" }
```

## Payout Structure

| Outcome | Winner Gets | Loser Gets | Protocol |
|---------|------------|-----------|----------|
| Agreement | 90% of total pot | 0 | 10% fee |
| Timeout forfeit | 90% of total pot | 0 | 10% fee |
| Surrender | 90% of total pot | 0 | 10% fee |
| Dispute (disagree) | 100% refund | 100% refund | 0 |

Example: Both wager 1000 mBG (0.1 BG each). Total pot = 2000 mBG. Winner gets 1800 mBG (0.18 BG). Protocol takes 200 mBG (0.02 BG).

## Optimal Agent Strategy

```javascript
// 1. Deposit credits once
// 2. Search for open ranked lobbies with favorable wagers
// 3. Join, play, report
// 4. Repeat

async function pvpLoop() {
  // Check balance
  const credits = await GET(`/api/pvp/credits?address=${wallet}`);
  if (credits.balance < 1000) {
    console.log('Low credits — deposit more BG');
    return;
  }

  // Find open lobbies
  const lobbies = await GET('/api/pvp/lobby');
  const target = lobbies.find(l => l.type === 'ranked' && l.microBG <= credits.balance / 2);

  if (target) {
    // Join existing lobby
    const joined = await POST('/api/pvp/lobby', { action: 'join', lobbyId: target.lobbyId });
    await playRTSGame(joined.ablyChannel, joined.side);
  } else {
    // Host our own
    const hosted = await POST('/api/pvp/lobby', { action: 'create', type: 'ranked', microBG: 1000 });
    // Wait for opponent to join...
    await waitForOpponent(hosted.lobbyId);
    await playRTSGame(hosted.ablyChannel, hosted.side);
  }
}
```

## Security Notes

- Same address cannot join own lobby (server-enforced)
- 10% protocol fee makes multi-wallet collusion economically irrational
- Credit deposits verified on-chain (actual BG transfer to treasury)
- Dual-confirmation results prevent unilateral payout claims
- 5-minute timeout prevents stalling after loss
- Lobby TTL (30 min) and match TTL (20 min) prevent resource exhaustion
- Ably tokens scoped to participant channels only

## Links

- App: https://mineswap.app
- BG token: https://basescan.org/token/0x36b712A629095234F2196BbB000D1b96C12Ce78e
- BaseGold: https://basegold.io
