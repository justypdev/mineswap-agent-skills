# MineSwap Agent Skills

Free AI agent skills for the [BaseGold](https://basegold.io) mining ecosystem on [Base Network](https://base.org).

**Buy a Gold Mine for $1.70. Mine gold, swap tokens, raid opponents, compete in PvP, and earn BG cashback — all autonomously.**

## Skills

| Skill | Description | Auth Required |
|-------|-------------|---------------|
| [mineswap-play](skills/mineswap-play/SKILL.md) | Mine gold, buy upgrades, send expeditions, train armies, raid other mines | Yes (NFT + session) |
| [mineswap-swap](skills/mineswap-swap/SKILL.md) | DEX aggregator routing across Uniswap V2/V3, SushiSwap, Aerodrome | No (quotes), Yes (swaps) |
| [mineswap-cashback](skills/mineswap-cashback/SKILL.md) | Claim smelter BG, DEX swap rebates, and fee processing rewards | Mixed |
| [mineswap-pvp](skills/mineswap-pvp/SKILL.md) | 1v1 ranked RTS matches with mBG wagers, server-authoritative winners | Yes (NFT + session) |

All skills are **free**. Revenue comes from on-chain activity (0.15% swap fee, shop burns), not skill fees.

## Quick Start

### For AI Agents (Claude, GPT, Cursor, any MCP client)

Read any SKILL.md file — it contains everything your agent needs: authentication flow, endpoint URLs, request/response formats, rate limits, and optimal play loops.

### For Developers

```bash
# Clone the skills
git clone https://github.com/justypdev/mineswap-agent-skills.git

# Or install via Smithery CLI
npx skills add justypdev/mineswap-agent-skills
```

## How It Works

1. **Buy a Gold Mine NFT** — Send 0.10 BG ($1.70) to the Gold Vein contract on Base
2. **Authenticate** — Sign a message with your wallet, verify NFT ownership, get a Bearer token
3. **Play** — Mine gold, upgrade, raid, compete — all through REST API calls
4. **Earn** — BG accumulates from airdrops, cashback, referrals, raids, and PvP
5. **Compound** — Use earned BG to buy more mines or swap on MineSwap

## Tokenomics

- **BG Token**: Fixed supply of 10,000 — no mint function, deflationary
- **Current Price**: ~$17.00 per BG
- **Mine Cost**: 0.10 BG ($1.70)
- **Swap Fee**: 0.15% split 50/50 → cashback + auto-burn
- **Every action burns BG** — supply only goes down, mine floor price only goes up

## Key Contracts (Base, chainId 8453)

| Contract | Address |
|----------|---------|
| BG Token | `0x36b712A629095234F2196BbB000D1b96C12Ce78e` |
| Gold Vein | `0x0520B1D4dF671293F8b4B1F52dDD4f9f687Fd565` |
| MineSwap Aggregator V4 | `0xeA55dC06AFd17F2105B175Be659d6CA4942D9D80` |
| InstantBurn | `0xF9dc5A103C5B09bfe71cF1Badcce362827b34BFE` |

## Links

- **Play**: [mineswap.app](https://mineswap.app)
- **Community**: [basegold.io](https://basegold.io)
- **Chart**: [DexScreener](https://dexscreener.com/base/0x36b712a629095234f2196bbb000d1b96c12ce78e)
- **Twitter**: [@BaseGold_BG](https://twitter.com/BaseGold_BG)
- **Email**: basegold@basegold.io

## Security

- 8 security audit rounds completed (Jay Anon)
- Server-authoritative game state with HMAC integrity
- On-chain NFT ownership verification on every session
- PvP winners determined by server from game event logs
- Rate limits identical for agents and humans

## License

MIT
