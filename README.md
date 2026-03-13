# X Layer Expert Skill

A Claude Code skill that provides deep expertise for building on [X Layer](https://www.okx.com/xlayer) — OKX's Layer 2 blockchain built on OP Stack.

## What it does

When triggered, this skill gives Claude Code specialized knowledge about:

- **Network configuration** — RPC endpoints, chain IDs (196 mainnet / 1952 testnet), re-genesis block
- **Smart contract security** — 18 Golden Rules covering reentrancy, L2-specific risks, signature replay, oracle safety, and more
- **Contract patterns** — Hardhat & Foundry config, deploy scripts, proxy/upgrade (UUPS), contract verification via OKLink
- **Gas optimization** — OKB economics, L1 data fee structure, calldata compression
- **Bridge & cross-chain** — OP Stack predeploys, L2→L1 withdrawals, L1→L2 deposits, AggLayer
- **Flashblocks** — Sub-second pre-confirmations, reorg handling
- **OnChain Data API** — OKLink REST API with HMAC authentication for querying blocks, transactions, tokens, and event logs
- **Testing** — Mainnet forking, security testing patterns, stress testing

## Installation

Copy the skill directory into your Claude Code skills folder:

```bash
cp -r xlayer-expert ~/.claude/skills/
```

Or clone this repo directly:

```bash
git clone https://github.com/<your-username>/xlayer-expert-skill.git ~/.claude/skills/xlayer-expert
```

## File structure

```
xlayer-expert/
├── SKILL.md                        # Main skill file — Golden Rules, triggers, reference guide
├── LICENSE                         # MIT License
├── README.md                       # This file
├── assets/
│   └── xlayer-architecture.png     # Architecture diagram
└── references/
    ├── security.md                 # Solidity security rules, L2 risks, attack patterns
    ├── network-config.md           # RPC URLs, chain IDs, performance specs
    ├── contract-patterns.md        # Hardhat/Foundry config, deploy, verify, proxy
    ├── token-addresses.md          # Token addresses + Multicall3
    ├── l2-predeploys.md            # OP Stack predeploy + L1 bridge addresses
    ├── gas-optimization.md         # OKB fee structure, optimization techniques
    ├── testing-patterns.md         # Forking, security testing, stress testing
    ├── flashblocks.md              # Flashblocks API, reorg risks
    ├── infrastructure.md           # RPC providers, xlayer-reth, monitoring, WebSocket
    ├── onchain-data-api.md         # OKLink REST API — blocks, txs, tokens, logs
    └── zkevm-differences.md        # CDK→OP Stack migration, EVM differences
```

## How it triggers

The skill activates automatically when your code or conversation involves:

| Trigger | Examples |
|---------|----------|
| Chain IDs | `chainId: 196`, `chainId: 1952` |
| RPC URLs | `rpc.xlayer.tech`, `xlayerrpc.okx.com` |
| Tokens | OKB, WOKB, OKB as gas token |
| Infrastructure | `xlayer-reth`, flashblocks |
| Contracts | `GasPriceOracle`, `L2CrossDomainMessenger`, `OptimismPortal` |
| Tools | Hardhat/Foundry with X Layer networks |
| API | `OK-ACCESS-KEY`, `/api/v5/xlayer/`, OKLink queries |

## Security

Every Solidity code block written with this skill is checked against 18 Golden Rules covering:

- Reentrancy (CEI pattern + ReentrancyGuard)
- Authentication (`msg.sender` over `tx.origin`)
- Token decimal handling (USDT=6, OKB=18)
- L2-specific risks (sequencer centralization, forced OKB sends)
- Signature safety (replay protection, malleability)
- Oracle integration (staleness checks, TWAP)
- On-chain data privacy (`private` != secret)

## License

MIT — see [LICENSE](LICENSE) for details.
