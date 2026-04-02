# DeFi Trading Skills — Design Spec

## Overview

Expand GigaBrain's official skills catalog with 5 new DeFi trading skills covering token swaps, cross-chain bridges, and wallet management across EVM and Solana chains. Skills follow the existing SKILL.md + `uv run` CLI script pattern (Agent Skills standard).

## Goals

- Agents can swap tokens on any major EVM chain and Solana
- Agents can bridge tokens between chains
- Agents can check balances, transfer tokens, and manage wallets across all supported chains
- Use existing aggregator APIs and SDKs — no direct protocol integrations
- Follow existing skill patterns exactly (SKILL.md + PEP 723 scripts + JSON output)

## Non-Goals

- Marketplace / third-party skill ecosystem (future)
- LP management, yield farming, lending/borrowing (future skills)
- Chains beyond EVM + Solana (future)

## Skills Catalog

### 1. `evm-swap`

**Purpose:** Swap tokens on any EVM chain via DEX aggregators.

**Underlying package:** `web3-ethereum-defi` (v1.0.2) — covers Uniswap V2/V3/V4, PancakeSwap. Falls back to 1inch/0x REST API for best routing.

**Chains:** Ethereum, Arbitrum, Base, Polygon, BSC, Optimism

**CLI commands:**
```
uv run scripts/evm_swap.py quote --chain base --from USDC --to ETH --amount 100
uv run scripts/evm_swap.py swap --chain base --from USDC --to ETH --amount 100 --slippage 0.5
uv run scripts/evm_swap.py tokens --chain base --search "pepe"
uv run scripts/evm_swap.py price --chain base --token ETH
uv run scripts/evm_swap.py approve --chain base --token USDC --spender <router>
```

**Environment:**
- `EVM_PRIVATE_KEY` — for signing transactions
- `EVM_WALLET_ADDRESS` — wallet address
- `EVM_RPC_URL_<CHAIN>` — per-chain RPC endpoints (e.g., `EVM_RPC_URL_BASE`)
- Optional: `ONEINCH_API_KEY` for 1inch routing

**Script structure:**
```
evm-swap/
├── SKILL.md
└── scripts/
    ├── evm_swap.py          # Main CLI (PEP 723 deps: web3-ethereum-defi, web3, eth-account)
    └── evm_utils.py         # Shared: chain configs, RPC URLs, token resolution
```

---

### 2. `solana-swap`

**Purpose:** Swap SPL tokens on Solana via Jupiter aggregator (routes through Raydium, Orca, Meteora automatically).

**Underlying package:** `jup-python-sdk` (v1.1.0) — official Jupiter SDK

**CLI commands:**
```
uv run scripts/sol_swap.py quote --from SOL --to USDC --amount 1.5
uv run scripts/sol_swap.py swap --from SOL --to USDC --amount 1.5 --slippage 0.5
uv run scripts/sol_swap.py price --token <mint_address_or_symbol>
uv run scripts/sol_swap.py tokens --search "bonk"
uv run scripts/sol_swap.py limit-order --from SOL --to USDC --amount 1 --price 150
uv run scripts/sol_swap.py dca --from USDC --to SOL --total 500 --every 1h --over 7d
```

**Environment:**
- `SOLANA_PRIVATE_KEY` — for signing transactions
- `SOLANA_WALLET_ADDRESS` — wallet address
- `SOLANA_RPC_URL` — Solana RPC (Helius recommended)
- Optional: `JUPITER_API_KEY` for higher rate limits

**Script structure:**
```
solana-swap/
├── SKILL.md
└── scripts/
    ├── sol_swap.py           # Main CLI (PEP 723 deps: jup-python-sdk, solders, solana)
    └── sol_utils.py          # Shared: token resolution, mint lookups
```

---

### 3. `evm-wallet`

**Purpose:** Multi-chain EVM wallet management — balances, transfers, token info, approvals.

**Underlying package:** `web3` + `eth-account`

**Chains:** Ethereum, Arbitrum, Base, Polygon, BSC, Optimism

**CLI commands:**
```
uv run scripts/evm_wallet.py balances --chain base
uv run scripts/evm_wallet.py balances --all-chains
uv run scripts/evm_wallet.py transfer --chain base --token ETH --to 0x... --amount 0.1
uv run scripts/evm_wallet.py transfer --chain base --token USDC --to 0x... --amount 100
uv run scripts/evm_wallet.py history --chain base --limit 20
uv run scripts/evm_wallet.py approve --chain base --token USDC --spender 0x... --amount 1000
uv run scripts/evm_wallet.py revoke --chain base --token USDC --spender 0x...
uv run scripts/evm_wallet.py gas --chain base
```

**Environment:**
- `EVM_PRIVATE_KEY`
- `EVM_WALLET_ADDRESS`
- `EVM_RPC_URL_<CHAIN>`

**Script structure:**
```
evm-wallet/
├── SKILL.md
└── scripts/
    ├── evm_wallet.py         # Main CLI (PEP 723 deps: web3, eth-account)
    ├── evm_utils.py          # Shared: chain configs, RPC URLs, token lists
    └── token_lists.py        # Well-known token addresses per chain
```

---

### 4. `solana-wallet`

**Purpose:** Solana wallet management — SOL + SPL token balances, transfers, token metadata.

**Underlying package:** `solana` + `solders`

**CLI commands:**
```
uv run scripts/sol_wallet.py balances
uv run scripts/sol_wallet.py transfer --token SOL --to <pubkey> --amount 1.5
uv run scripts/sol_wallet.py transfer --token <mint> --to <pubkey> --amount 100
uv run scripts/sol_wallet.py history --limit 20
uv run scripts/sol_wallet.py token-info --mint <address>
```

**Environment:**
- `SOLANA_PRIVATE_KEY`
- `SOLANA_WALLET_ADDRESS`
- `SOLANA_RPC_URL`

**Script structure:**
```
solana-wallet/
├── SKILL.md
└── scripts/
    ├── sol_wallet.py         # Main CLI (PEP 723 deps: solana, solders, spl-token)
    └── sol_utils.py          # Shared: token resolution, mint lookups
```

---

### 5. `cross-chain-bridge`

**Purpose:** Bridge tokens between any supported chains via Li.Fi aggregator.

**Underlying package:** Li.Fi REST API (no Python SDK — direct HTTP calls via `httpx`)

**Supported routes:** 40+ chains, 100+ bridges aggregated by Li.Fi (Stargate, Across, Hop, Wormhole, etc.)

**CLI commands:**
```
uv run scripts/bridge.py quote --from-chain ethereum --to-chain arbitrum --token USDC --amount 500
uv run scripts/bridge.py routes --from-chain ethereum --to-chain base --token ETH --amount 1
uv run scripts/bridge.py execute --from-chain ethereum --to-chain arbitrum --token USDC --amount 500 --route <route_id>
uv run scripts/bridge.py status --tx <tx_hash>
uv run scripts/bridge.py chains
uv run scripts/bridge.py tokens --chain base
```

**Environment:**
- `EVM_PRIVATE_KEY` — for signing on source chain
- `EVM_WALLET_ADDRESS`
- `SOLANA_PRIVATE_KEY` — for Solana-involved bridges
- `SOLANA_WALLET_ADDRESS`
- Optional: `LIFI_API_KEY` for higher rate limits

**Script structure:**
```
cross-chain-bridge/
├── SKILL.md
└── scripts/
    ├── bridge.py             # Main CLI (PEP 723 deps: httpx, web3, eth-account)
    └── bridge_utils.py       # Chain ID mapping, Li.Fi API helpers
```

---

## Shared Patterns

All 5 skills follow these conventions (matching existing HyperLiquid/Polymarket skills):

### SKILL.md Frontmatter
```yaml
---
name: <skill-name>
description: >-
  Action-oriented description. Mention protocols, chains, and use cases.
  End with "Use when the user asks about..."
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---
```

### JSON Output
```json
{"success": true, "data": {...}}
{"success": false, "error": "Human-readable error message"}
```

### Script Conventions
- PEP 723 inline dependencies (no requirements.txt)
- `uv run scripts/<cli>.py <command> [args]`
- Argparse subcommands
- `asyncio.run()` for async operations
- Environment variables for credentials (never hardcoded)
- `_out(data)` helper for JSON stdout
- Error handling: catch exceptions → `{"success": false, "error": "..."}`

### Safety Rules
- ALWAYS check balance before executing swaps/transfers
- ALWAYS show quote/preview before executing
- ALWAYS verify after every write operation (check balances post-swap)
- Never retry failed transactions without user confirmation
- Include slippage protection with sensible defaults (0.5%)
- Gas estimation before execution

## Shared Utilities

Several scripts share common logic. Rather than creating a shared library (which complicates `uv run` isolation), use co-located utility files per skill that can be imported via `sys.path.insert(0, ...)` — matching the existing pattern in HyperLiquid scripts.

Common utilities duplicated across relevant skills:
- **Chain configs** — RPC URLs, chain IDs, explorer URLs
- **Token resolution** — symbol → address lookups for well-known tokens
- **Output formatting** — `_out()` helper, decimal serialization

## Directory Layout

```
skills/
├── hyperliquid/          # Existing
├── polymarket/           # Existing
├── portfolio-tracker/    # Existing
├── brain/               # Existing
├── evm-swap/             # NEW
│   ├── SKILL.md
│   └── scripts/
├── solana-swap/          # NEW
│   ├── SKILL.md
│   └── scripts/
├── evm-wallet/           # NEW
│   ├── SKILL.md
│   └── scripts/
├── solana-wallet/        # NEW
│   ├── SKILL.md
│   └── scripts/
└── cross-chain-bridge/   # NEW
    ├── SKILL.md
    └── scripts/
```

## Implementation Order

1. **evm-wallet** — foundation, needed by other skills for balance checks
2. **solana-wallet** — same, for Solana side
3. **evm-swap** — depends on wallet for pre/post verification
4. **solana-swap** — depends on wallet for pre/post verification
5. **cross-chain-bridge** — depends on both wallet skills for balance verification

## Environment Configuration

New env vars needed per agent (set via daemon config):

```
# EVM (already partially supported via Privy)
EVM_PRIVATE_KEY=0x...
EVM_WALLET_ADDRESS=0x...
EVM_RPC_URL_ETHEREUM=https://...
EVM_RPC_URL_ARBITRUM=https://...
EVM_RPC_URL_BASE=https://...
EVM_RPC_URL_POLYGON=https://...
EVM_RPC_URL_BSC=https://...
EVM_RPC_URL_OPTIMISM=https://...

# Solana (already partially supported)
SOLANA_PRIVATE_KEY=...
SOLANA_WALLET_ADDRESS=...
SOLANA_RPC_URL=https://...

# Optional API keys
JUPITER_API_KEY=...
ONEINCH_API_KEY=...
LIFI_API_KEY=...
```

## Risk Considerations

- **Transaction signing**: Skills sign real transactions with real funds. Safety skill rules apply.
- **Slippage**: Default 0.5%, configurable. Skills MUST show quote before execution.
- **Gas**: Skills MUST estimate gas and warn if unusually high.
- **Token approvals**: Skills MUST explain what approvals do before requesting them.
- **Bridge delays**: Cross-chain bridges can take minutes to hours. Skills MUST show estimated time.
- **Private key exposure**: Keys are in env vars within the container. Container isolation via ECS provides the security boundary.
