# DeFi Trading Skills Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 5 new official trading skills (evm-wallet, solana-wallet, evm-swap, solana-swap, cross-chain-bridge) following the existing SKILL.md + uv run CLI pattern.

**Architecture:** Each skill is a self-contained directory with SKILL.md (agent instructions) + scripts/ (PEP 723 Python CLI). CLI scripts follow the two-file pattern: `*_client.py` (thin argparse CLI) + `*_services.py` (business logic). All output is JSON to stdout with `{"success": true/false, ...}`.

**Tech Stack:** web3 + eth-account (EVM), solders + solana (Solana), jup-python-sdk (Jupiter swaps), web3-ethereum-defi (EVM DEX), httpx (Li.Fi bridge API)

**Spec:** `docs/superpowers/specs/2026-03-11-defi-trading-skills-design.md`

**Key Reference Files:**
- Existing skill pattern: `skills/hyperliquid/scripts/hl_client.py` + `hl_services.py`
- Existing skill pattern: `skills/polymarket/scripts/pm_client.py` + `pm_services.py`
- Daemon config (env vars): `intelligence-monorepo/services/daemon/daemon-core/daemon/config.py`
- Skill loader: `intelligence-monorepo/services/daemon/daemon-core/daemon/skills/loader.py`
- Env var names: `EVM_PRIVATE_KEY`, `EVM_WALLET_ADDRESS`, `SOL_PRIVATE_KEY`, `SOL_WALLET_ADDRESS`

---

## Chunk 1: EVM Wallet Skill

### Task 1: Create evm-wallet SKILL.md

**Files:**
- Create: `skills/evm-wallet/SKILL.md`

- [ ] **Step 1: Create the skill directory and SKILL.md**

```markdown
---
name: evm-wallet
description: >-
  Manage EVM wallets across Ethereum, Arbitrum, Base, Polygon, BSC, and Optimism.
  Check native and ERC20 token balances, transfer tokens, view transaction history,
  manage token approvals, and estimate gas costs. Use when the user asks about
  wallet balances, token transfers, approvals, gas fees, or EVM portfolio overview.
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---

# EVM Wallet Management

Manage wallets across EVM chains — balances, transfers, approvals, gas.

` ` `
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py <command> [args]
` ` `

All commands return JSON to stdout with a `success` field.

## Environment

| Variable | Required | Description |
|---|---|---|
| `EVM_WALLET_ADDRESS` | Yes | Your EVM wallet address |
| `EVM_PRIVATE_KEY` | For transfers | Signing key — enables transfers and approvals |

Without `EVM_PRIVATE_KEY`, all read commands work but write commands return an error.

## Supported Chains

`ethereum`, `arbitrum`, `base`, `polygon`, `bsc`, `optimism`

Default chain is `ethereum` if `--chain` is omitted.

## Commands

### Configuration
` ` `bash
# Show config status and supported chains
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py config
` ` `

### Balances
` ` `bash
# Native + top ERC20 token balances on a chain
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py balances --chain base

# Balances across all supported chains
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py balances --all-chains

# Balance of a specific token
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py balance-of --chain base --token USDC
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py balance-of --chain base --token 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
` ` `

### Transfers
` ` `bash
# Transfer native token (ETH, MATIC, etc.)
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py transfer --chain base --token ETH --to 0x... --amount 0.1

# Transfer ERC20 token
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py transfer --chain base --token USDC --to 0x... --amount 100
` ` `

### Token Approvals
` ` `bash
# Check current approval
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py allowance --chain base --token USDC --spender 0x...

# Approve token spending
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py approve --chain base --token USDC --spender 0x... --amount 1000

# Revoke approval
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py revoke --chain base --token USDC --spender 0x...
` ` `

### Gas
` ` `bash
# Current gas price on chain
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py gas --chain ethereum
` ` `

### Token Info
` ` `bash
# Look up token by symbol or address
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_wallet.py token-info --chain base --token USDC
` ` `

## Safety Rules

1. **ALWAYS** check balance before transfers
2. After **EVERY** transfer, verify with `balances`
3. **NEVER** retry failed transfers without user confirmation
4. Explain what approvals do before requesting them
5. Double-check recipient address with user before transfers
```

Note: Replace ` ` ` with actual triple backticks (shown escaped here for plan readability).

- [ ] **Step 2: Verify SKILL.md is valid**

Manually check:
- Frontmatter has `name`, `description`, `license`, `allowed-tools`, `metadata`
- `name` is kebab-case
- Description starts with action verb and ends with "Use when..."

- [ ] **Step 3: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add evm-wallet/SKILL.md
git commit -m "feat(skills): add evm-wallet SKILL.md"
```

---

### Task 2: Create evm_services.py — chain config and RPC

**Files:**
- Create: `skills/evm-wallet/scripts/evm_services.py`

This is the business logic module. The CLI script will import from it.

- [ ] **Step 1: Create evm_services.py with chain configs and wallet operations**

```python
"""EVM wallet service layer — multi-chain balances, transfers, approvals.

Uses web3.py and eth-account. Supports read-only mode (no private key).
"""

from __future__ import annotations

import logging
from decimal import Decimal
from typing import Any

from web3 import Web3
from web3.middleware import ExtraDataToPOAMiddleware
from eth_account import Account

logger = logging.getLogger("evm_services")

# ---------------------------------------------------------------------------
# Chain configurations
# ---------------------------------------------------------------------------

CHAINS: dict[str, dict[str, Any]] = {
    "ethereum": {
        "chain_id": 1,
        "rpc": "https://eth.llamarpc.com",
        "native": "ETH",
        "explorer": "https://etherscan.io",
    },
    "arbitrum": {
        "chain_id": 42161,
        "rpc": "https://arb1.arbitrum.io/rpc",
        "native": "ETH",
        "explorer": "https://arbiscan.io",
        "poa": True,
    },
    "base": {
        "chain_id": 8453,
        "rpc": "https://mainnet.base.org",
        "native": "ETH",
        "explorer": "https://basescan.org",
        "poa": True,
    },
    "polygon": {
        "chain_id": 137,
        "rpc": "https://polygon-rpc.com",
        "native": "POL",
        "explorer": "https://polygonscan.com",
        "poa": True,
    },
    "bsc": {
        "chain_id": 56,
        "rpc": "https://bsc-dataseed.binance.org",
        "native": "BNB",
        "explorer": "https://bscscan.com",
        "poa": True,
    },
    "optimism": {
        "chain_id": 10,
        "rpc": "https://mainnet.optimism.io",
        "native": "ETH",
        "explorer": "https://optimistic.etherscan.io",
        "poa": True,
    },
}

# Well-known tokens per chain (symbol -> address)
KNOWN_TOKENS: dict[str, dict[str, str]] = {
    "ethereum": {
        "USDC": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "USDT": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
        "WETH": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "DAI": "0x6B175474E89094C44Da98b954EedeAC495271d0F",
        "WBTC": "0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599",
    },
    "arbitrum": {
        "USDC": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
        "USDC.e": "0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8",
        "USDT": "0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9",
        "WETH": "0x82aF49447D8a07e3bd95BD0d56f35241523fBab1",
        "WBTC": "0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f",
        "ARB": "0x912CE59144191C1204E64559FE8253a0e49E6548",
    },
    "base": {
        "USDC": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "USDbC": "0xd9aAEc86B65D86f6A7B5B1b0c42FFA531710b6CA",
        "WETH": "0x4200000000000000000000000000000000000006",
        "DAI": "0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb",
    },
    "polygon": {
        "USDC": "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359",
        "USDC.e": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
        "USDT": "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
        "WETH": "0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619",
        "WMATIC": "0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270",
    },
    "bsc": {
        "USDC": "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d",
        "USDT": "0x55d398326f99059fF775485246999027B3197955",
        "WBNB": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c",
        "BUSD": "0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56",
    },
    "optimism": {
        "USDC": "0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85",
        "USDC.e": "0x7F5c764cBc14f9669B88837ca1490cCa17c31607",
        "USDT": "0x94b008aA00579c1307B0EF2c499aD98a8ce58e58",
        "WETH": "0x4200000000000000000000000000000000000006",
        "OP": "0x4200000000000000000000000000000000000042",
    },
}

# Minimal ERC20 ABI for balanceOf, transfer, approve, allowance
ERC20_ABI = [
    {"constant": True, "inputs": [{"name": "owner", "type": "address"}], "name": "balanceOf", "outputs": [{"name": "", "type": "uint256"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "decimals", "outputs": [{"name": "", "type": "uint8"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "symbol", "outputs": [{"name": "", "type": "string"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "name", "outputs": [{"name": "", "type": "string"}], "type": "function"},
    {"constant": True, "inputs": [{"name": "owner", "type": "address"}, {"name": "spender", "type": "address"}], "name": "allowance", "outputs": [{"name": "", "type": "uint256"}], "type": "function"},
    {"constant": False, "inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "approve", "outputs": [{"name": "", "type": "bool"}], "type": "function"},
    {"constant": False, "inputs": [{"name": "to", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "transfer", "outputs": [{"name": "", "type": "bool"}], "type": "function"},
]


class EVMWalletServices:
    """Multi-chain EVM wallet operations."""

    def __init__(self, wallet_address: str, private_key: str | None = None):
        self.wallet_address = Web3.to_checksum_address(wallet_address)
        self.private_key = private_key
        self._w3_cache: dict[str, Web3] = {}

    def _get_w3(self, chain: str) -> Web3:
        if chain not in CHAINS:
            raise ValueError(f"Unsupported chain: {chain}. Supported: {list(CHAINS.keys())}")
        if chain not in self._w3_cache:
            cfg = CHAINS[chain]
            w3 = Web3(Web3.HTTPProvider(cfg["rpc"]))
            if cfg.get("poa"):
                w3.middleware_onion.inject(ExtraDataToPOAMiddleware, layer=0)
            self._w3_cache[chain] = w3
        return self._w3_cache[chain]

    def _resolve_token(self, chain: str, token: str) -> str | None:
        """Resolve token symbol to address. Returns None for native token."""
        upper = token.upper()
        native = CHAINS[chain]["native"]
        if upper in (native, f"W{native}") and upper == native:
            return None  # Native token
        # Check known tokens
        chain_tokens = KNOWN_TOKENS.get(chain, {})
        if upper in chain_tokens:
            return chain_tokens[upper]
        # If it looks like an address, use directly
        if token.startswith("0x") and len(token) == 42:
            return Web3.to_checksum_address(token)
        # Check case-insensitive
        for sym, addr in chain_tokens.items():
            if sym.upper() == upper:
                return addr
        raise ValueError(f"Unknown token '{token}' on {chain}. Use a contract address or one of: {list(chain_tokens.keys())}")

    def _require_signer(self):
        if not self.private_key:
            raise ValueError("EVM_PRIVATE_KEY must be set for write operations. Run 'config' to check.")

    def show_config(self) -> dict:
        return {
            "success": True,
            "wallet_address": self.wallet_address,
            "has_private_key": bool(self.private_key),
            "supported_chains": list(CHAINS.keys()),
        }

    def get_balances(self, chain: str) -> dict:
        w3 = self._get_w3(chain)
        cfg = CHAINS[chain]
        native_bal = w3.eth.get_balance(self.wallet_address)
        result = {
            "success": True,
            "chain": chain,
            "native": {
                "symbol": cfg["native"],
                "balance": str(Web3.from_wei(native_bal, "ether")),
                "balance_raw": str(native_bal),
            },
            "tokens": [],
        }
        # Check known tokens
        for symbol, addr in KNOWN_TOKENS.get(chain, {}).items():
            try:
                contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
                raw = contract.functions.balanceOf(self.wallet_address).call()
                if raw > 0:
                    decimals = contract.functions.decimals().call()
                    balance = Decimal(raw) / Decimal(10 ** decimals)
                    result["tokens"].append({
                        "symbol": symbol,
                        "address": addr,
                        "balance": str(balance),
                        "decimals": decimals,
                    })
            except Exception:
                continue
        return result

    def get_all_chain_balances(self) -> dict:
        chains_data = {}
        for chain in CHAINS:
            try:
                chains_data[chain] = self.get_balances(chain)
            except Exception as e:
                chains_data[chain] = {"success": False, "error": str(e)}
        return {"success": True, "chains": chains_data}

    def get_token_balance(self, chain: str, token: str) -> dict:
        w3 = self._get_w3(chain)
        addr = self._resolve_token(chain, token)
        if addr is None:
            native_bal = w3.eth.get_balance(self.wallet_address)
            return {
                "success": True,
                "chain": chain,
                "symbol": CHAINS[chain]["native"],
                "balance": str(Web3.from_wei(native_bal, "ether")),
            }
        contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
        raw = contract.functions.balanceOf(self.wallet_address).call()
        decimals = contract.functions.decimals().call()
        symbol = contract.functions.symbol().call()
        return {
            "success": True,
            "chain": chain,
            "symbol": symbol,
            "address": addr,
            "balance": str(Decimal(raw) / Decimal(10 ** decimals)),
            "decimals": decimals,
        }

    def transfer(self, chain: str, token: str, to: str, amount: float) -> dict:
        self._require_signer()
        w3 = self._get_w3(chain)
        to_addr = Web3.to_checksum_address(to)
        addr = self._resolve_token(chain, token)
        nonce = w3.eth.get_transaction_count(self.wallet_address)

        if addr is None:
            # Native transfer
            value = Web3.to_wei(amount, "ether")
            tx = {
                "from": self.wallet_address,
                "to": to_addr,
                "value": value,
                "nonce": nonce,
                "chainId": CHAINS[chain]["chain_id"],
            }
            tx["gas"] = w3.eth.estimate_gas(tx)
            tx["maxFeePerGas"] = w3.eth.gas_price * 2
            tx["maxPriorityFeePerGas"] = Web3.to_wei(1, "gwei")
        else:
            # ERC20 transfer
            contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
            decimals = contract.functions.decimals().call()
            raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))
            tx = contract.functions.transfer(to_addr, raw_amount).build_transaction({
                "from": self.wallet_address,
                "nonce": nonce,
                "chainId": CHAINS[chain]["chain_id"],
            })

        signed = Account.sign_transaction(tx, self.private_key)
        tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)

        return {
            "success": receipt["status"] == 1,
            "chain": chain,
            "tx_hash": receipt["transactionHash"].hex(),
            "token": token,
            "amount": amount,
            "to": to,
            "gas_used": receipt["gasUsed"],
            "explorer": f"{CHAINS[chain]['explorer']}/tx/{receipt['transactionHash'].hex()}",
            "error": "Transaction reverted" if receipt["status"] != 1 else None,
        }

    def get_allowance(self, chain: str, token: str, spender: str) -> dict:
        w3 = self._get_w3(chain)
        addr = self._resolve_token(chain, token)
        if addr is None:
            return {"success": True, "allowance": "unlimited", "note": "Native tokens don't need approvals"}
        contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
        decimals = contract.functions.decimals().call()
        raw = contract.functions.allowance(self.wallet_address, Web3.to_checksum_address(spender)).call()
        return {
            "success": True,
            "chain": chain,
            "token": token,
            "spender": spender,
            "allowance": str(Decimal(raw) / Decimal(10 ** decimals)),
            "allowance_raw": str(raw),
        }

    def approve(self, chain: str, token: str, spender: str, amount: float | None = None) -> dict:
        self._require_signer()
        w3 = self._get_w3(chain)
        addr = self._resolve_token(chain, token)
        if addr is None:
            return {"success": False, "error": "Native tokens don't need approvals"}
        contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
        decimals = contract.functions.decimals().call()
        if amount is None:
            raw_amount = 2**256 - 1  # Max approval
        else:
            raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))
        nonce = w3.eth.get_transaction_count(self.wallet_address)
        tx = contract.functions.approve(Web3.to_checksum_address(spender), raw_amount).build_transaction({
            "from": self.wallet_address,
            "nonce": nonce,
            "chainId": CHAINS[chain]["chain_id"],
        })
        signed = Account.sign_transaction(tx, self.private_key)
        tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
        return {
            "success": receipt["status"] == 1,
            "chain": chain,
            "tx_hash": receipt["transactionHash"].hex(),
            "token": token,
            "spender": spender,
            "amount": "unlimited" if amount is None else amount,
            "explorer": f"{CHAINS[chain]['explorer']}/tx/{receipt['transactionHash'].hex()}",
        }

    def revoke(self, chain: str, token: str, spender: str) -> dict:
        return self.approve(chain, token, spender, amount=0)

    def get_gas_price(self, chain: str) -> dict:
        w3 = self._get_w3(chain)
        gas_price = w3.eth.gas_price
        return {
            "success": True,
            "chain": chain,
            "gas_price_gwei": str(Web3.from_wei(gas_price, "gwei")),
            "gas_price_wei": str(gas_price),
        }

    def get_token_info(self, chain: str, token: str) -> dict:
        addr = self._resolve_token(chain, token)
        if addr is None:
            cfg = CHAINS[chain]
            return {
                "success": True,
                "chain": chain,
                "symbol": cfg["native"],
                "name": cfg["native"],
                "decimals": 18,
                "type": "native",
            }
        w3 = self._get_w3(chain)
        contract = w3.eth.contract(address=Web3.to_checksum_address(addr), abi=ERC20_ABI)
        return {
            "success": True,
            "chain": chain,
            "symbol": contract.functions.symbol().call(),
            "name": contract.functions.name().call(),
            "decimals": contract.functions.decimals().call(),
            "address": addr,
            "type": "ERC20",
        }
```

- [ ] **Step 2: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add evm-wallet/scripts/evm_services.py
git commit -m "feat(skills): add evm-wallet services layer"
```

---

### Task 3: Create evm_wallet.py CLI

**Files:**
- Create: `skills/evm-wallet/scripts/evm_wallet.py`

- [ ] **Step 1: Create the CLI script**

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = ["web3", "eth-account"]
# ///
"""EVM Wallet CLI — multi-chain balances, transfers, approvals.

All output is JSON to stdout. Supports read-only mode (no private key).

Run with: uv run evm_wallet.py <command> [args]
"""

from __future__ import annotations

import argparse
import json
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))


def _get_services(require_key: bool = False):
    from evm_services import EVMWalletServices
    address = os.environ.get("EVM_WALLET_ADDRESS", "")
    private_key = os.environ.get("EVM_PRIVATE_KEY") or None
    if not address:
        _out({"success": False, "error": "EVM_WALLET_ADDRESS must be set. Run 'config' to check."})
        sys.exit(1)
    if require_key and not private_key:
        _out({"success": False, "error": "EVM_PRIVATE_KEY must be set for this operation. Run 'config' to check."})
        sys.exit(1)
    return EVMWalletServices(wallet_address=address, private_key=private_key)


def _out(data):
    print(json.dumps(data, default=str))


def cmd_config(args):
    svc = _get_services()
    _out(svc.show_config())

def cmd_balances(args):
    svc = _get_services()
    if args.all_chains:
        _out(svc.get_all_chain_balances())
    else:
        _out(svc.get_balances(args.chain))

def cmd_balance_of(args):
    svc = _get_services()
    _out(svc.get_token_balance(args.chain, args.token))

def cmd_transfer(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(svc.transfer(args.chain, args.token, args.to, args.amount))

def cmd_allowance(args):
    svc = _get_services()
    _out(svc.get_allowance(args.chain, args.token, args.spender))

def cmd_approve(args):
    svc = _get_services(require_key=True)
    _out(svc.approve(args.chain, args.token, args.spender, args.amount))

def cmd_revoke(args):
    svc = _get_services(require_key=True)
    _out(svc.revoke(args.chain, args.token, args.spender))

def cmd_gas(args):
    svc = _get_services()
    _out(svc.get_gas_price(args.chain))

def cmd_token_info(args):
    svc = _get_services()
    _out(svc.get_token_info(args.chain, args.token))


def main():
    parser = argparse.ArgumentParser(description="EVM Wallet CLI — multi-chain wallet management")
    sub = parser.add_subparsers(dest="command", required=True)

    # config
    sub.add_parser("config", help="Show configuration status")

    # balances
    p = sub.add_parser("balances", help="Token balances on chain(s)")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--all-chains", action="store_true")

    # balance-of
    p = sub.add_parser("balance-of", help="Balance of a specific token")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)

    # transfer
    p = sub.add_parser("transfer", help="Transfer tokens")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)
    p.add_argument("--to", required=True)
    p.add_argument("--amount", type=float, required=True)

    # allowance
    p = sub.add_parser("allowance", help="Check token allowance")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)
    p.add_argument("--spender", required=True)

    # approve
    p = sub.add_parser("approve", help="Approve token spending")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)
    p.add_argument("--spender", required=True)
    p.add_argument("--amount", type=float, default=None)

    # revoke
    p = sub.add_parser("revoke", help="Revoke token approval")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)
    p.add_argument("--spender", required=True)

    # gas
    p = sub.add_parser("gas", help="Current gas price")
    p.add_argument("--chain", default="ethereum")

    # token-info
    p = sub.add_parser("token-info", help="Token metadata")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", required=True)

    args = parser.parse_args()

    handler = {
        "config": cmd_config,
        "balances": cmd_balances,
        "balance-of": cmd_balance_of,
        "transfer": cmd_transfer,
        "allowance": cmd_allowance,
        "approve": cmd_approve,
        "revoke": cmd_revoke,
        "gas": cmd_gas,
        "token-info": cmd_token_info,
    }[args.command]

    try:
        handler(args)
    except Exception as e:
        _out({"success": False, "error": str(e)})
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Test CLI locally (smoke test)**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills/evm-wallet
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run scripts/evm_wallet.py config
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run scripts/evm_wallet.py balances --chain ethereum
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run scripts/evm_wallet.py gas --chain base
```

Expected: JSON output with `"success": true` for each command.

- [ ] **Step 3: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add evm-wallet/scripts/evm_wallet.py
git commit -m "feat(skills): add evm-wallet CLI script"
```

---

## Chunk 2: Solana Wallet Skill

### Task 4: Create solana-wallet SKILL.md

**Files:**
- Create: `skills/solana-wallet/SKILL.md`

- [ ] **Step 1: Create the skill directory and SKILL.md**

```markdown
---
name: solana-wallet
description: >-
  Manage Solana wallets. Check SOL and SPL token balances, transfer SOL and SPL tokens,
  view transaction history, and look up token metadata. Use when the user asks about
  Solana balances, SOL transfers, SPL token operations, or Solana wallet management.
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---

# Solana Wallet Management

Manage Solana wallets — balances, transfers, token info.

` ` `
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py <command> [args]
` ` `

All commands return JSON to stdout with a `success` field.

## Environment

| Variable | Required | Description |
|---|---|---|
| `SOL_WALLET_ADDRESS` | Yes | Your Solana wallet public key |
| `SOL_PRIVATE_KEY` | For transfers | Base58 private key — enables transfers |
| `SOLANA_RPC_URL` | No | Custom RPC endpoint (default: public mainnet) |

Without `SOL_PRIVATE_KEY`, all read commands work but transfers return an error.

## Commands

### Configuration
` ` `bash
# Show config status
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py config
` ` `

### Balances
` ` `bash
# SOL + all SPL token balances
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py balances

# Balance of specific token by mint address
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py balance-of --mint <mint_address>
` ` `

### Transfers
` ` `bash
# Transfer SOL
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py transfer --to <pubkey> --amount 1.5

# Transfer SPL token
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py transfer-spl --mint <mint_address> --to <pubkey> --amount 100
` ` `

### Token Info
` ` `bash
# Look up token metadata by mint address
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_wallet.py token-info --mint <mint_address>
` ` `

## Safety Rules

1. **ALWAYS** check balance before transfers
2. After **EVERY** transfer, verify with `balances`
3. **NEVER** retry failed transfers without user confirmation
4. Double-check recipient address with user before transfers
```

Note: Replace ` ` ` with actual triple backticks.

- [ ] **Step 2: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add solana-wallet/SKILL.md
git commit -m "feat(skills): add solana-wallet SKILL.md"
```

---

### Task 5: Create sol_services.py and sol_wallet.py CLI

**Files:**
- Create: `skills/solana-wallet/scripts/sol_services.py`
- Create: `skills/solana-wallet/scripts/sol_wallet.py`

- [ ] **Step 1: Create sol_services.py**

```python
"""Solana wallet service layer — balances, transfers, token info.

Uses solders and solana-py. Supports read-only mode (no private key).
"""

from __future__ import annotations

import base64
import logging
from decimal import Decimal
from typing import Any

from solders.keypair import Keypair
from solders.pubkey import Pubkey
from solders.system_program import TransferParams, transfer
from solders.transaction import Transaction
from solders.message import Message
from solana.rpc.api import Client
from solana.rpc.commitment import Confirmed

logger = logging.getLogger("sol_services")

DEFAULT_RPC = "https://api.mainnet-beta.solana.com"
SOL_DECIMALS = 9
TOKEN_PROGRAM_ID = Pubkey.from_string("TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA")
ASSOCIATED_TOKEN_PROGRAM_ID = Pubkey.from_string("ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL")


class SolanaWalletServices:
    """Solana wallet operations."""

    def __init__(self, wallet_address: str, private_key: str | None = None, rpc_url: str | None = None):
        self.wallet_pubkey = Pubkey.from_string(wallet_address)
        self.private_key = private_key
        self.keypair: Keypair | None = None
        if private_key:
            try:
                key_bytes = base64.b58decode(private_key)
                self.keypair = Keypair.from_bytes(key_bytes)
            except Exception:
                # Try as raw bytes list
                self.keypair = Keypair.from_base58_string(private_key)
        self.client = Client(rpc_url or DEFAULT_RPC)

    def _require_signer(self):
        if not self.keypair:
            raise ValueError("SOL_PRIVATE_KEY must be set for write operations. Run 'config' to check.")

    def show_config(self) -> dict:
        return {
            "success": True,
            "wallet_address": str(self.wallet_pubkey),
            "has_private_key": self.keypair is not None,
            "rpc_url": self.client._provider.endpoint_uri,
        }

    def get_balances(self) -> dict:
        # SOL balance
        resp = self.client.get_balance(self.wallet_pubkey, commitment=Confirmed)
        sol_lamports = resp.value
        sol_balance = Decimal(sol_lamports) / Decimal(10 ** SOL_DECIMALS)

        # SPL tokens
        tokens = []
        try:
            token_resp = self.client.get_token_accounts_by_owner_json_parsed(
                self.wallet_pubkey,
                opts={"programId": TOKEN_PROGRAM_ID},
                commitment=Confirmed,
            )
            for account in token_resp.value:
                info = account.account.data.parsed["info"]
                mint = info["mint"]
                amount = info["tokenAmount"]
                if float(amount["uiAmount"] or 0) > 0:
                    tokens.append({
                        "mint": mint,
                        "balance": amount["uiAmountString"],
                        "decimals": amount["decimals"],
                    })
        except Exception as e:
            logger.warning(f"Failed to fetch SPL tokens: {e}")

        return {
            "success": True,
            "sol": {"balance": str(sol_balance), "lamports": sol_lamports},
            "tokens": tokens,
        }

    def get_token_balance(self, mint: str) -> dict:
        mint_pubkey = Pubkey.from_string(mint)
        # Find ATA
        ata = Pubkey.find_program_address(
            [bytes(self.wallet_pubkey), bytes(TOKEN_PROGRAM_ID), bytes(mint_pubkey)],
            ASSOCIATED_TOKEN_PROGRAM_ID,
        )[0]
        try:
            resp = self.client.get_token_account_balance(ata, commitment=Confirmed)
            amount = resp.value
            return {
                "success": True,
                "mint": mint,
                "balance": amount.ui_amount_string,
                "decimals": amount.decimals,
            }
        except Exception:
            return {"success": True, "mint": mint, "balance": "0", "decimals": 0}

    def transfer_sol(self, to: str, amount: float) -> dict:
        self._require_signer()
        lamports = int(Decimal(str(amount)) * Decimal(10 ** SOL_DECIMALS))
        to_pubkey = Pubkey.from_string(to)
        ix = transfer(TransferParams(from_pubkey=self.wallet_pubkey, to_pubkey=to_pubkey, lamports=lamports))
        blockhash_resp = self.client.get_latest_blockhash(commitment=Confirmed)
        msg = Message.new_with_blockhash([ix], self.wallet_pubkey, blockhash_resp.value.blockhash)
        tx = Transaction.new_unsigned(msg)
        tx.sign([self.keypair], blockhash_resp.value.blockhash)
        result = self.client.send_transaction(tx)
        sig = str(result.value)
        return {
            "success": True,
            "signature": sig,
            "amount": amount,
            "to": to,
            "explorer": f"https://solscan.io/tx/{sig}",
        }

    def transfer_spl(self, mint: str, to: str, amount: float) -> dict:
        self._require_signer()
        # SPL transfers require the spl-token library
        from spl.token.instructions import transfer_checked, TransferCheckedParams
        from spl.token.constants import TOKEN_PROGRAM_ID as SPL_TOKEN_PROGRAM_ID

        mint_pubkey = Pubkey.from_string(mint)
        to_pubkey = Pubkey.from_string(to)

        # Get decimals
        mint_info = self.client.get_account_info_json_parsed(mint_pubkey, commitment=Confirmed)
        decimals = mint_info.value.data.parsed["info"]["decimals"]
        raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))

        # Source ATA
        source_ata = Pubkey.find_program_address(
            [bytes(self.wallet_pubkey), bytes(SPL_TOKEN_PROGRAM_ID), bytes(mint_pubkey)],
            ASSOCIATED_TOKEN_PROGRAM_ID,
        )[0]

        # Dest ATA
        dest_ata = Pubkey.find_program_address(
            [bytes(to_pubkey), bytes(SPL_TOKEN_PROGRAM_ID), bytes(mint_pubkey)],
            ASSOCIATED_TOKEN_PROGRAM_ID,
        )[0]

        ix = transfer_checked(TransferCheckedParams(
            program_id=SPL_TOKEN_PROGRAM_ID,
            source=source_ata,
            mint=mint_pubkey,
            dest=dest_ata,
            owner=self.wallet_pubkey,
            amount=raw_amount,
            decimals=decimals,
        ))

        blockhash_resp = self.client.get_latest_blockhash(commitment=Confirmed)
        msg = Message.new_with_blockhash([ix], self.wallet_pubkey, blockhash_resp.value.blockhash)
        tx = Transaction.new_unsigned(msg)
        tx.sign([self.keypair], blockhash_resp.value.blockhash)
        result = self.client.send_transaction(tx)
        sig = str(result.value)
        return {
            "success": True,
            "signature": sig,
            "mint": mint,
            "amount": amount,
            "to": to,
            "explorer": f"https://solscan.io/tx/{sig}",
        }

    def get_token_info(self, mint: str) -> dict:
        mint_pubkey = Pubkey.from_string(mint)
        info = self.client.get_account_info_json_parsed(mint_pubkey, commitment=Confirmed)
        if not info.value:
            return {"success": False, "error": f"Mint {mint} not found"}
        parsed = info.value.data.parsed["info"]
        return {
            "success": True,
            "mint": mint,
            "decimals": parsed.get("decimals"),
            "supply": parsed.get("supply"),
            "freeze_authority": parsed.get("freezeAuthority"),
            "mint_authority": parsed.get("mintAuthority"),
        }
```

- [ ] **Step 2: Create sol_wallet.py CLI**

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = ["solana", "solders", "spl-token"]
# ///
"""Solana Wallet CLI — balances, transfers, token info.

All output is JSON to stdout. Supports read-only mode (no private key).

Run with: uv run sol_wallet.py <command> [args]
"""

from __future__ import annotations

import argparse
import json
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))


def _get_services(require_key: bool = False):
    from sol_services import SolanaWalletServices
    address = os.environ.get("SOL_WALLET_ADDRESS", "")
    private_key = os.environ.get("SOL_PRIVATE_KEY") or None
    rpc_url = os.environ.get("SOLANA_RPC_URL") or None
    if not address:
        _out({"success": False, "error": "SOL_WALLET_ADDRESS must be set. Run 'config' to check."})
        sys.exit(1)
    if require_key and not private_key:
        _out({"success": False, "error": "SOL_PRIVATE_KEY must be set for this operation."})
        sys.exit(1)
    return SolanaWalletServices(wallet_address=address, private_key=private_key, rpc_url=rpc_url)


def _out(data):
    print(json.dumps(data, default=str))


def cmd_config(args):
    svc = _get_services()
    _out(svc.show_config())

def cmd_balances(args):
    svc = _get_services()
    _out(svc.get_balances())

def cmd_balance_of(args):
    svc = _get_services()
    _out(svc.get_token_balance(args.mint))

def cmd_transfer(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(svc.transfer_sol(args.to, args.amount))

def cmd_transfer_spl(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(svc.transfer_spl(args.mint, args.to, args.amount))

def cmd_token_info(args):
    svc = _get_services()
    _out(svc.get_token_info(args.mint))


def main():
    parser = argparse.ArgumentParser(description="Solana Wallet CLI")
    sub = parser.add_subparsers(dest="command", required=True)

    sub.add_parser("config", help="Show configuration status")
    sub.add_parser("balances", help="SOL + SPL token balances")

    p = sub.add_parser("balance-of", help="Balance of specific token")
    p.add_argument("--mint", required=True)

    p = sub.add_parser("transfer", help="Transfer SOL")
    p.add_argument("--to", required=True)
    p.add_argument("--amount", type=float, required=True)

    p = sub.add_parser("transfer-spl", help="Transfer SPL token")
    p.add_argument("--mint", required=True)
    p.add_argument("--to", required=True)
    p.add_argument("--amount", type=float, required=True)

    p = sub.add_parser("token-info", help="Token metadata by mint")
    p.add_argument("--mint", required=True)

    args = parser.parse_args()
    handler = {
        "config": cmd_config,
        "balances": cmd_balances,
        "balance-of": cmd_balance_of,
        "transfer": cmd_transfer,
        "transfer-spl": cmd_transfer_spl,
        "token-info": cmd_token_info,
    }[args.command]

    try:
        handler(args)
    except Exception as e:
        _out({"success": False, "error": str(e)})
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: Smoke test**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills/solana-wallet
SOL_WALLET_ADDRESS=So11111111111111111111111111111111111111112 uv run scripts/sol_wallet.py config
```

- [ ] **Step 4: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add solana-wallet/
git commit -m "feat(skills): add solana-wallet skill"
```

---

## Chunk 3: EVM Swap Skill

### Task 6: Create evm-swap SKILL.md

**Files:**
- Create: `skills/evm-swap/SKILL.md`

- [ ] **Step 1: Create SKILL.md**

```markdown
---
name: evm-swap
description: >-
  Swap tokens on EVM chains via DEX aggregators. Get quotes, execute swaps with
  slippage protection, and look up token prices on Ethereum, Arbitrum, Base, Polygon,
  BSC, and Optimism. Routes through Uniswap, PancakeSwap, and other DEXs for best
  pricing. Use when the user asks about token swaps, DEX trading, token prices,
  or buying/selling tokens on EVM chains.
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---

# EVM Token Swaps

Swap tokens on EVM chains via DEX aggregators for best pricing.

` ` `
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_swap.py <command> [args]
` ` `

All commands return JSON to stdout with a `success` field.

## Environment

| Variable | Required | Description |
|---|---|---|
| `EVM_WALLET_ADDRESS` | For quotes | Your EVM wallet address |
| `EVM_PRIVATE_KEY` | For swaps | Signing key — enables swap execution |

## Supported Chains

`ethereum`, `arbitrum`, `base`, `polygon`, `bsc`, `optimism`

## Commands

### Get Quote (no signing needed)
` ` `bash
# Quote a swap
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_swap.py quote --chain base --from ETH --to USDC --amount 1.0

# Quote with custom slippage
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_swap.py quote --chain base --from USDC --to ETH --amount 500 --slippage 1.0
` ` `

### Execute Swap
` ` `bash
# Swap tokens (gets quote, approves if needed, executes)
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_swap.py swap --chain base --from USDC --to ETH --amount 500 --slippage 0.5
` ` `

### Token Lookup
` ` `bash
# Look up token by symbol or address
uv run ${CLAUDE_SKILL_DIR}/scripts/evm_swap.py token-info --chain base --token USDC
` ` `

## Safety Rules

1. **ALWAYS** get a quote before executing a swap
2. **ALWAYS** verify balances before and after swaps
3. Default slippage is 0.5% — adjust for volatile tokens
4. **NEVER** retry failed swaps without user confirmation
5. For large swaps, warn about price impact
```

Note: Replace ` ` ` with actual triple backticks.

- [ ] **Step 2: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add evm-swap/SKILL.md
git commit -m "feat(skills): add evm-swap SKILL.md"
```

---

### Task 7: Create evm_swap_services.py and evm_swap.py CLI

**Files:**
- Create: `skills/evm-swap/scripts/evm_swap_services.py`
- Create: `skills/evm-swap/scripts/evm_swap.py`

- [ ] **Step 1: Create evm_swap_services.py**

This uses the Li.Fi API for swap quotes and execution (same API used for bridges, works as a DEX aggregator too). Li.Fi routes through Uniswap, 1inch, Paraswap, etc. for best pricing. This avoids needing separate integrations for each DEX.

```python
"""EVM swap service layer — token swaps via Li.Fi aggregator.

Li.Fi aggregates multiple DEXs (Uniswap, 1inch, Paraswap, etc.) for best routing.
Uses httpx for API calls, web3 + eth-account for transaction signing.
"""

from __future__ import annotations

import logging
from decimal import Decimal
from typing import Any

import httpx
from web3 import Web3
from web3.middleware import ExtraDataToPOAMiddleware
from eth_account import Account

logger = logging.getLogger("evm_swap_services")

LIFI_API = "https://li.quest/v1"

CHAINS: dict[str, dict[str, Any]] = {
    "ethereum": {"chain_id": 1, "rpc": "https://eth.llamarpc.com", "native": "ETH", "explorer": "https://etherscan.io", "native_address": "0x0000000000000000000000000000000000000000"},
    "arbitrum": {"chain_id": 42161, "rpc": "https://arb1.arbitrum.io/rpc", "native": "ETH", "explorer": "https://arbiscan.io", "poa": True, "native_address": "0x0000000000000000000000000000000000000000"},
    "base": {"chain_id": 8453, "rpc": "https://mainnet.base.org", "native": "ETH", "explorer": "https://basescan.org", "poa": True, "native_address": "0x0000000000000000000000000000000000000000"},
    "polygon": {"chain_id": 137, "rpc": "https://polygon-rpc.com", "native": "POL", "explorer": "https://polygonscan.com", "poa": True, "native_address": "0x0000000000000000000000000000000000000000"},
    "bsc": {"chain_id": 56, "rpc": "https://bsc-dataseed.binance.org", "native": "BNB", "explorer": "https://bscscan.com", "poa": True, "native_address": "0x0000000000000000000000000000000000000000"},
    "optimism": {"chain_id": 10, "rpc": "https://mainnet.optimism.io", "native": "ETH", "explorer": "https://optimistic.etherscan.io", "poa": True, "native_address": "0x0000000000000000000000000000000000000000"},
}

# Same known tokens as evm-wallet
KNOWN_TOKENS: dict[str, dict[str, str]] = {
    "ethereum": {"USDC": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", "USDT": "0xdAC17F958D2ee523a2206206994597C13D831ec7", "WETH": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", "DAI": "0x6B175474E89094C44Da98b954EedeAC495271d0F"},
    "arbitrum": {"USDC": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831", "USDT": "0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9", "WETH": "0x82aF49447D8a07e3bd95BD0d56f35241523fBab1", "ARB": "0x912CE59144191C1204E64559FE8253a0e49E6548"},
    "base": {"USDC": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "WETH": "0x4200000000000000000000000000000000000006"},
    "polygon": {"USDC": "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359", "USDT": "0xc2132D05D31c914a87C6611C10748AEb04B58e8F", "WETH": "0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619"},
    "bsc": {"USDC": "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d", "USDT": "0x55d398326f99059fF775485246999027B3197955", "WBNB": "0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c"},
    "optimism": {"USDC": "0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85", "USDT": "0x94b008aA00579c1307B0EF2c499aD98a8ce58e58", "WETH": "0x4200000000000000000000000000000000000006", "OP": "0x4200000000000000000000000000000000000042"},
}

ERC20_ABI = [
    {"constant": True, "inputs": [{"name": "owner", "type": "address"}, {"name": "spender", "type": "address"}], "name": "allowance", "outputs": [{"name": "", "type": "uint256"}], "type": "function"},
    {"constant": False, "inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "approve", "outputs": [{"name": "", "type": "bool"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "decimals", "outputs": [{"name": "", "type": "uint8"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "symbol", "outputs": [{"name": "", "type": "string"}], "type": "function"},
]


class EVMSwapServices:
    """Token swap operations via Li.Fi aggregator."""

    def __init__(self, wallet_address: str, private_key: str | None = None):
        self.wallet_address = Web3.to_checksum_address(wallet_address)
        self.private_key = private_key
        self._w3_cache: dict[str, Web3] = {}

    def _get_w3(self, chain: str) -> Web3:
        if chain not in CHAINS:
            raise ValueError(f"Unsupported chain: {chain}. Supported: {list(CHAINS.keys())}")
        if chain not in self._w3_cache:
            cfg = CHAINS[chain]
            w3 = Web3(Web3.HTTPProvider(cfg["rpc"]))
            if cfg.get("poa"):
                w3.middleware_onion.inject(ExtraDataToPOAMiddleware, layer=0)
            self._w3_cache[chain] = w3
        return self._w3_cache[chain]

    def _resolve_token_address(self, chain: str, token: str) -> str:
        upper = token.upper()
        native = CHAINS[chain]["native"]
        if upper == native:
            return CHAINS[chain]["native_address"]
        chain_tokens = KNOWN_TOKENS.get(chain, {})
        if upper in chain_tokens:
            return chain_tokens[upper]
        if token.startswith("0x") and len(token) == 42:
            return Web3.to_checksum_address(token)
        for sym, addr in chain_tokens.items():
            if sym.upper() == upper:
                return addr
        raise ValueError(f"Unknown token '{token}' on {chain}. Use a contract address or: {list(chain_tokens.keys())}")

    def _require_signer(self):
        if not self.private_key:
            raise ValueError("EVM_PRIVATE_KEY must be set for swap execution.")

    async def get_quote(self, chain: str, from_token: str, to_token: str, amount: float, slippage: float = 0.5) -> dict:
        chain_id = CHAINS[chain]["chain_id"]
        from_addr = self._resolve_token_address(chain, from_token)
        to_addr = self._resolve_token_address(chain, to_token)

        # Get decimals for from_token
        if from_addr == CHAINS[chain]["native_address"]:
            decimals = 18
        else:
            w3 = self._get_w3(chain)
            contract = w3.eth.contract(address=Web3.to_checksum_address(from_addr), abi=ERC20_ABI)
            decimals = contract.functions.decimals().call()

        raw_amount = str(int(Decimal(str(amount)) * Decimal(10 ** decimals)))

        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{LIFI_API}/quote", params={
                "fromChain": chain_id,
                "toChain": chain_id,
                "fromToken": from_addr,
                "toToken": to_addr,
                "fromAmount": raw_amount,
                "fromAddress": self.wallet_address,
                "slippage": slippage / 100,
            })
            if resp.status_code != 200:
                return {"success": False, "error": f"Li.Fi API error: {resp.text}"}
            data = resp.json()

        estimate = data.get("estimate", {})
        return {
            "success": True,
            "chain": chain,
            "from_token": from_token,
            "to_token": to_token,
            "from_amount": amount,
            "to_amount": estimate.get("toAmountMin"),
            "to_amount_usd": estimate.get("toAmountUSD"),
            "from_amount_usd": estimate.get("fromAmountUSD"),
            "gas_cost_usd": estimate.get("gasCosts", [{}])[0].get("amountUSD") if estimate.get("gasCosts") else None,
            "slippage": slippage,
            "tool": data.get("tool"),
            "transaction_request": data.get("transactionRequest"),
        }

    async def execute_swap(self, chain: str, from_token: str, to_token: str, amount: float, slippage: float = 0.5) -> dict:
        self._require_signer()

        quote = await self.get_quote(chain, from_token, to_token, amount, slippage)
        if not quote["success"]:
            return quote

        tx_request = quote.get("transaction_request")
        if not tx_request:
            return {"success": False, "error": "No transaction request in quote"}

        w3 = self._get_w3(chain)

        # Check if approval needed for non-native tokens
        from_addr = self._resolve_token_address(chain, from_token)
        if from_addr != CHAINS[chain]["native_address"] and tx_request.get("to"):
            contract = w3.eth.contract(address=Web3.to_checksum_address(from_addr), abi=ERC20_ABI)
            decimals = contract.functions.decimals().call()
            raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))
            allowance = contract.functions.allowance(self.wallet_address, Web3.to_checksum_address(tx_request["to"])).call()
            if allowance < raw_amount:
                # Approve
                approve_tx = contract.functions.approve(
                    Web3.to_checksum_address(tx_request["to"]), 2**256 - 1
                ).build_transaction({
                    "from": self.wallet_address,
                    "nonce": w3.eth.get_transaction_count(self.wallet_address),
                    "chainId": CHAINS[chain]["chain_id"],
                })
                signed = Account.sign_transaction(approve_tx, self.private_key)
                tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
                w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)

        # Execute swap
        tx = {
            "from": self.wallet_address,
            "to": Web3.to_checksum_address(tx_request["to"]),
            "data": tx_request["data"],
            "value": int(tx_request.get("value", "0"), 16) if isinstance(tx_request.get("value"), str) else int(tx_request.get("value", 0)),
            "nonce": w3.eth.get_transaction_count(self.wallet_address),
            "chainId": CHAINS[chain]["chain_id"],
            "gas": int(tx_request.get("gasLimit", "0"), 16) if isinstance(tx_request.get("gasLimit"), str) else int(tx_request.get("gasLimit", 500000)),
        }
        if tx_request.get("gasPrice"):
            tx["gasPrice"] = int(tx_request["gasPrice"], 16) if isinstance(tx_request["gasPrice"], str) else int(tx_request["gasPrice"])
        else:
            tx["maxFeePerGas"] = w3.eth.gas_price * 2
            tx["maxPriorityFeePerGas"] = Web3.to_wei(1, "gwei")

        signed = Account.sign_transaction(tx, self.private_key)
        tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=180)

        return {
            "success": receipt["status"] == 1,
            "chain": chain,
            "tx_hash": receipt["transactionHash"].hex(),
            "from_token": from_token,
            "to_token": to_token,
            "from_amount": amount,
            "to_amount_expected": quote.get("to_amount"),
            "gas_used": receipt["gasUsed"],
            "explorer": f"{CHAINS[chain]['explorer']}/tx/{receipt['transactionHash'].hex()}",
            "error": "Transaction reverted" if receipt["status"] != 1 else None,
        }
```

- [ ] **Step 2: Create evm_swap.py CLI**

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = ["web3", "eth-account", "httpx"]
# ///
"""EVM Swap CLI — token swaps via DEX aggregators.

All output is JSON to stdout.

Run with: uv run evm_swap.py <command> [args]
"""

from __future__ import annotations

import argparse
import asyncio
import json
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))


def _get_services(require_key: bool = False):
    from evm_swap_services import EVMSwapServices
    address = os.environ.get("EVM_WALLET_ADDRESS", "")
    private_key = os.environ.get("EVM_PRIVATE_KEY") or None
    if not address:
        _out({"success": False, "error": "EVM_WALLET_ADDRESS must be set."})
        sys.exit(1)
    if require_key and not private_key:
        _out({"success": False, "error": "EVM_PRIVATE_KEY must be set for swaps."})
        sys.exit(1)
    return EVMSwapServices(wallet_address=address, private_key=private_key)


def _out(data):
    print(json.dumps(data, default=str))


async def cmd_quote(args):
    svc = _get_services()
    _out(await svc.get_quote(args.chain, args.from_token, args.to_token, args.amount, args.slippage))

async def cmd_swap(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(await svc.execute_swap(args.chain, args.from_token, args.to_token, args.amount, args.slippage))

async def cmd_token_info(args):
    from evm_swap_services import KNOWN_TOKENS, CHAINS
    chain = args.chain
    if chain not in CHAINS:
        _out({"success": False, "error": f"Unsupported chain: {chain}"})
        return
    tokens = KNOWN_TOKENS.get(chain, {})
    if args.token:
        upper = args.token.upper()
        if upper in tokens:
            _out({"success": True, "chain": chain, "symbol": upper, "address": tokens[upper]})
        else:
            _out({"success": False, "error": f"Unknown token '{args.token}' on {chain}. Known: {list(tokens.keys())}"})
    else:
        _out({"success": True, "chain": chain, "tokens": tokens})


def main():
    parser = argparse.ArgumentParser(description="EVM Swap CLI")
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("quote", help="Get swap quote")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--from", dest="from_token", required=True)
    p.add_argument("--to", dest="to_token", required=True)
    p.add_argument("--amount", type=float, required=True)
    p.add_argument("--slippage", type=float, default=0.5)

    p = sub.add_parser("swap", help="Execute swap")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--from", dest="from_token", required=True)
    p.add_argument("--to", dest="to_token", required=True)
    p.add_argument("--amount", type=float, required=True)
    p.add_argument("--slippage", type=float, default=0.5)

    p = sub.add_parser("token-info", help="Token lookup")
    p.add_argument("--chain", default="ethereum")
    p.add_argument("--token", default=None)

    args = parser.parse_args()
    handler = {
        "quote": cmd_quote,
        "swap": cmd_swap,
        "token-info": cmd_token_info,
    }[args.command]

    try:
        if asyncio.iscoroutinefunction(handler):
            asyncio.run(handler(args))
        else:
            handler(args)
    except Exception as e:
        _out({"success": False, "error": str(e)})
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add evm-swap/
git commit -m "feat(skills): add evm-swap skill"
```

---

## Chunk 4: Solana Swap Skill

### Task 8: Create solana-swap skill

**Files:**
- Create: `skills/solana-swap/SKILL.md`
- Create: `skills/solana-swap/scripts/sol_swap_services.py`
- Create: `skills/solana-swap/scripts/sol_swap.py`

- [ ] **Step 1: Create SKILL.md**

```markdown
---
name: solana-swap
description: >-
  Swap SPL tokens on Solana via Jupiter aggregator. Get quotes, execute swaps with
  slippage protection, look up token prices, place limit orders, and set up DCA.
  Jupiter routes through Raydium, Orca, Meteora, and other Solana DEXs for best
  pricing. Use when the user asks about swapping Solana tokens, Jupiter trading,
  buying/selling SPL tokens, token prices on Solana, or DCA strategies.
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---

# Solana Token Swaps (Jupiter)

Swap SPL tokens via Jupiter — routes through all major Solana DEXs.

` ` `
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py <command> [args]
` ` `

All commands return JSON to stdout with a `success` field.

## Environment

| Variable | Required | Description |
|---|---|---|
| `SOL_WALLET_ADDRESS` | Yes | Your Solana wallet public key |
| `SOL_PRIVATE_KEY` | For swaps | Base58 private key — enables swap execution |
| `SOLANA_RPC_URL` | No | Custom RPC (default: public mainnet) |
| `JUPITER_API_KEY` | No | Jupiter API key for higher rate limits |

## Commands

### Get Quote
` ` `bash
# Quote SOL -> USDC
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py quote --from So11111111111111111111111111111111111111112 --to EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v --amount 1.0

# Use well-known symbols
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py quote --from SOL --to USDC --amount 1.0
` ` `

### Execute Swap
` ` `bash
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py swap --from SOL --to USDC --amount 1.0 --slippage 0.5
` ` `

### Token Price
` ` `bash
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py price --token SOL
uv run ${CLAUDE_SKILL_DIR}/scripts/sol_swap.py price --token <mint_address>
` ` `

## Well-Known Tokens

| Symbol | Mint Address |
|---|---|
| SOL | So11111111111111111111111111111111111111112 |
| USDC | EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v |
| USDT | Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB |

You can use either symbols or mint addresses.

## Safety Rules

1. **ALWAYS** get a quote before executing a swap
2. **ALWAYS** verify balances before and after swaps
3. Default slippage is 0.5% — increase for low-liquidity tokens
4. **NEVER** retry failed swaps without user confirmation
```

Note: Replace ` ` ` with actual triple backticks.

- [ ] **Step 2: Create sol_swap_services.py**

```python
"""Solana swap service layer — token swaps via Jupiter aggregator.

Uses the Jupiter Ultra API for quotes and swap execution.
"""

from __future__ import annotations

import base64
import logging
from decimal import Decimal
from typing import Any

import httpx
from solders.keypair import Keypair
from solders.transaction import VersionedTransaction

logger = logging.getLogger("sol_swap_services")

JUPITER_API = "https://lite-api.jup.ag"
JUPITER_PRICE_API = "https://api.jup.ag/price/v2"

KNOWN_MINTS: dict[str, str] = {
    "SOL": "So11111111111111111111111111111111111111112",
    "USDC": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "USDT": "Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB",
    "BONK": "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263",
    "JUP": "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN",
    "WIF": "EKpQGSJtjMFqKZ9KQanSqYXRcF8fBopzLHYxdM65zcjm",
    "PYTH": "HZ1JovNiVvGrGNiiYvEozEVgZ58xaU3RKwX8eACQBCt3",
    "RAY": "4k3Dyjzvzp8eMZWUXbBCjEvwSkkk59S5iCNLY3QrkX6R",
}


class SolanaSwapServices:
    """Solana token swaps via Jupiter."""

    def __init__(self, wallet_address: str, private_key: str | None = None, rpc_url: str | None = None, api_key: str | None = None):
        self.wallet_address = wallet_address
        self.private_key = private_key
        self.keypair: Keypair | None = None
        if private_key:
            self.keypair = Keypair.from_base58_string(private_key)
        self.rpc_url = rpc_url or "https://api.mainnet-beta.solana.com"
        self.api_base = JUPITER_API
        self.api_key = api_key

    def _resolve_mint(self, token: str) -> str:
        upper = token.upper()
        if upper in KNOWN_MINTS:
            return KNOWN_MINTS[upper]
        # Assume it's a mint address
        return token

    def _require_signer(self):
        if not self.keypair:
            raise ValueError("SOL_PRIVATE_KEY must be set for swap execution.")

    async def get_quote(self, from_token: str, to_token: str, amount: float, slippage: float = 0.5) -> dict:
        input_mint = self._resolve_mint(from_token)
        output_mint = self._resolve_mint(to_token)

        # Get decimals for input token
        decimals = 9 if input_mint == KNOWN_MINTS.get("SOL") else 6  # Default assumption
        # Try to get actual decimals via price API
        raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))

        headers = {}
        if self.api_key:
            headers["Authorization"] = f"Bearer {self.api_key}"

        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{self.api_base}/swap/v1/quote", params={
                "inputMint": input_mint,
                "outputMint": output_mint,
                "amount": raw_amount,
                "slippageBps": int(slippage * 100),
            }, headers=headers)
            if resp.status_code != 200:
                return {"success": False, "error": f"Jupiter API error: {resp.text}"}
            data = resp.json()

        out_amount = int(data.get("outAmount", 0))
        out_decimals = 6  # Most tokens are 6 decimals, SOL is 9
        if output_mint == KNOWN_MINTS.get("SOL"):
            out_decimals = 9

        return {
            "success": True,
            "from_token": from_token,
            "to_token": to_token,
            "from_amount": amount,
            "to_amount": str(Decimal(out_amount) / Decimal(10 ** out_decimals)),
            "price_impact": data.get("priceImpactPct"),
            "slippage_bps": int(slippage * 100),
            "route_plan": [
                {"swap": step.get("swapInfo", {}).get("label"), "percent": step.get("percent")}
                for step in data.get("routePlan", [])
            ],
        }

    async def execute_swap(self, from_token: str, to_token: str, amount: float, slippage: float = 0.5) -> dict:
        self._require_signer()
        input_mint = self._resolve_mint(from_token)
        output_mint = self._resolve_mint(to_token)

        decimals = 9 if input_mint == KNOWN_MINTS.get("SOL") else 6
        raw_amount = int(Decimal(str(amount)) * Decimal(10 ** decimals))

        headers = {"Content-Type": "application/json"}
        if self.api_key:
            headers["Authorization"] = f"Bearer {self.api_key}"

        async with httpx.AsyncClient(timeout=60) as client:
            # Get quote
            quote_resp = await client.get(f"{self.api_base}/swap/v1/quote", params={
                "inputMint": input_mint,
                "outputMint": output_mint,
                "amount": raw_amount,
                "slippageBps": int(slippage * 100),
            }, headers=headers)
            if quote_resp.status_code != 200:
                return {"success": False, "error": f"Quote failed: {quote_resp.text}"}
            quote_data = quote_resp.json()

            # Get swap transaction
            swap_resp = await client.post(f"{self.api_base}/swap/v1/swap", json={
                "quoteResponse": quote_data,
                "userPublicKey": self.wallet_address,
                "wrapAndUnwrapSol": True,
            }, headers=headers)
            if swap_resp.status_code != 200:
                return {"success": False, "error": f"Swap tx build failed: {swap_resp.text}"}
            swap_data = swap_resp.json()

        # Sign and send
        swap_tx_bytes = base64.b64decode(swap_data["swapTransaction"])
        tx = VersionedTransaction.from_bytes(swap_tx_bytes)
        signed_tx = VersionedTransaction(tx.message, [self.keypair])

        async with httpx.AsyncClient(timeout=60) as client:
            send_resp = await client.post(self.rpc_url, json={
                "jsonrpc": "2.0",
                "id": 1,
                "method": "sendTransaction",
                "params": [
                    base64.b64encode(bytes(signed_tx)).decode(),
                    {"encoding": "base64", "skipPreflight": False},
                ],
            })
            result = send_resp.json()

        if "error" in result:
            return {"success": False, "error": result["error"].get("message", str(result["error"]))}

        sig = result.get("result", "")
        return {
            "success": True,
            "signature": sig,
            "from_token": from_token,
            "to_token": to_token,
            "from_amount": amount,
            "explorer": f"https://solscan.io/tx/{sig}",
        }

    async def get_price(self, token: str) -> dict:
        mint = self._resolve_mint(token)
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(JUPITER_PRICE_API, params={"ids": mint})
            if resp.status_code != 200:
                return {"success": False, "error": f"Price API error: {resp.text}"}
            data = resp.json()
        price_data = data.get("data", {}).get(mint)
        if not price_data:
            return {"success": False, "error": f"No price data for {token}"}
        return {
            "success": True,
            "token": token,
            "mint": mint,
            "price_usd": price_data.get("price"),
        }
```

- [ ] **Step 3: Create sol_swap.py CLI**

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = ["httpx", "solders"]
# ///
"""Solana Swap CLI — token swaps via Jupiter.

All output is JSON to stdout.

Run with: uv run sol_swap.py <command> [args]
"""

from __future__ import annotations

import argparse
import asyncio
import json
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))


def _get_services(require_key: bool = False):
    from sol_swap_services import SolanaSwapServices
    address = os.environ.get("SOL_WALLET_ADDRESS", "")
    private_key = os.environ.get("SOL_PRIVATE_KEY") or None
    rpc_url = os.environ.get("SOLANA_RPC_URL") or None
    api_key = os.environ.get("JUPITER_API_KEY") or None
    if not address:
        _out({"success": False, "error": "SOL_WALLET_ADDRESS must be set."})
        sys.exit(1)
    if require_key and not private_key:
        _out({"success": False, "error": "SOL_PRIVATE_KEY must be set for swaps."})
        sys.exit(1)
    return SolanaSwapServices(wallet_address=address, private_key=private_key, rpc_url=rpc_url, api_key=api_key)


def _out(data):
    print(json.dumps(data, default=str))


async def cmd_quote(args):
    svc = _get_services()
    _out(await svc.get_quote(args.from_token, args.to_token, args.amount, args.slippage))

async def cmd_swap(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(await svc.execute_swap(args.from_token, args.to_token, args.amount, args.slippage))

async def cmd_price(args):
    svc = _get_services()
    _out(await svc.get_price(args.token))


def main():
    parser = argparse.ArgumentParser(description="Solana Swap CLI (Jupiter)")
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("quote", help="Get swap quote")
    p.add_argument("--from", dest="from_token", required=True)
    p.add_argument("--to", dest="to_token", required=True)
    p.add_argument("--amount", type=float, required=True)
    p.add_argument("--slippage", type=float, default=0.5)

    p = sub.add_parser("swap", help="Execute swap")
    p.add_argument("--from", dest="from_token", required=True)
    p.add_argument("--to", dest="to_token", required=True)
    p.add_argument("--amount", type=float, required=True)
    p.add_argument("--slippage", type=float, default=0.5)

    p = sub.add_parser("price", help="Token price in USD")
    p.add_argument("--token", required=True)

    args = parser.parse_args()
    handler = {"quote": cmd_quote, "swap": cmd_swap, "price": cmd_price}[args.command]

    try:
        asyncio.run(handler(args))
    except Exception as e:
        _out({"success": False, "error": str(e)})
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add solana-swap/
git commit -m "feat(skills): add solana-swap skill (Jupiter)"
```

---

## Chunk 5: Cross-Chain Bridge Skill

### Task 9: Create cross-chain-bridge skill

**Files:**
- Create: `skills/cross-chain-bridge/SKILL.md`
- Create: `skills/cross-chain-bridge/scripts/bridge_services.py`
- Create: `skills/cross-chain-bridge/scripts/bridge.py`

- [ ] **Step 1: Create SKILL.md**

```markdown
---
name: cross-chain-bridge
description: >-
  Bridge tokens between blockchains via Li.Fi aggregator. Get bridge quotes, compare
  routes across 100+ bridges (Stargate, Across, Hop, Wormhole, etc.), execute cross-chain
  transfers, and track bridge status. Supports EVM chains and Solana. Use when the user
  asks about bridging tokens, cross-chain transfers, moving funds between chains, or
  comparing bridge routes.
license: MIT
allowed-tools: Bash(uv *)
metadata:
  author: gigabrain
  version: "1.0"
---

# Cross-Chain Bridge

Bridge tokens between chains via Li.Fi — aggregates 100+ bridges for best routes.

` ` `
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py <command> [args]
` ` `

All commands return JSON to stdout with a `success` field.

## Environment

| Variable | Required | Description |
|---|---|---|
| `EVM_WALLET_ADDRESS` | Yes | Your EVM wallet address |
| `EVM_PRIVATE_KEY` | For bridging | Signing key — enables bridge execution |
| `LIFI_API_KEY` | No | Li.Fi API key for higher rate limits |

## Supported Chains

EVM: `ethereum` (1), `arbitrum` (42161), `base` (8453), `polygon` (137), `bsc` (56), `optimism` (10)

More chains available via Li.Fi — use `chains` command to see all.

## Commands

### List Chains
` ` `bash
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py chains
` ` `

### Get Bridge Routes
` ` `bash
# Get available routes with pricing
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py routes --from-chain ethereum --to-chain arbitrum --token USDC --amount 500

# Routes for native token
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py routes --from-chain ethereum --to-chain base --token ETH --amount 1
` ` `

### Execute Bridge
` ` `bash
# Bridge using best route
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py execute --from-chain ethereum --to-chain arbitrum --token USDC --amount 500

# Bridge with specific slippage
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py execute --from-chain ethereum --to-chain base --token ETH --amount 1 --slippage 0.5
` ` `

### Check Status
` ` `bash
# Check bridge transaction status
uv run ${CLAUDE_SKILL_DIR}/scripts/bridge.py status --tx 0x...
` ` `

## Safety Rules

1. **ALWAYS** get routes before executing a bridge
2. **ALWAYS** verify balances before and after bridging
3. Bridges can take minutes to hours — inform user of estimated time
4. **NEVER** retry failed bridges without user confirmation
5. Double-check destination chain and amount before executing
6. Warn about bridge fees and slippage before execution
```

Note: Replace ` ` ` with actual triple backticks.

- [ ] **Step 2: Create bridge_services.py**

```python
"""Cross-chain bridge service layer — via Li.Fi aggregator.

Uses Li.Fi REST API for quotes, routing, and execution.
Web3 + eth-account for transaction signing.
"""

from __future__ import annotations

import logging
from decimal import Decimal
from typing import Any

import httpx
from web3 import Web3
from web3.middleware import ExtraDataToPOAMiddleware
from eth_account import Account

logger = logging.getLogger("bridge_services")

LIFI_API = "https://li.quest/v1"

CHAINS: dict[str, dict[str, Any]] = {
    "ethereum": {"chain_id": 1, "rpc": "https://eth.llamarpc.com", "native": "ETH", "explorer": "https://etherscan.io"},
    "arbitrum": {"chain_id": 42161, "rpc": "https://arb1.arbitrum.io/rpc", "native": "ETH", "explorer": "https://arbiscan.io", "poa": True},
    "base": {"chain_id": 8453, "rpc": "https://mainnet.base.org", "native": "ETH", "explorer": "https://basescan.org", "poa": True},
    "polygon": {"chain_id": 137, "rpc": "https://polygon-rpc.com", "native": "POL", "explorer": "https://polygonscan.com", "poa": True},
    "bsc": {"chain_id": 56, "rpc": "https://bsc-dataseed.binance.org", "native": "BNB", "explorer": "https://bscscan.com", "poa": True},
    "optimism": {"chain_id": 10, "rpc": "https://mainnet.optimism.io", "native": "ETH", "explorer": "https://optimistic.etherscan.io", "poa": True},
}

NATIVE_ADDRESS = "0x0000000000000000000000000000000000000000"

KNOWN_TOKENS: dict[str, dict[str, str]] = {
    "ethereum": {"USDC": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", "USDT": "0xdAC17F958D2ee523a2206206994597C13D831ec7", "WETH": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"},
    "arbitrum": {"USDC": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831", "USDT": "0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9", "WETH": "0x82aF49447D8a07e3bd95BD0d56f35241523fBab1"},
    "base": {"USDC": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "WETH": "0x4200000000000000000000000000000000000006"},
    "polygon": {"USDC": "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359", "USDT": "0xc2132D05D31c914a87C6611C10748AEb04B58e8F"},
    "bsc": {"USDC": "0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d", "USDT": "0x55d398326f99059fF775485246999027B3197955"},
    "optimism": {"USDC": "0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85", "USDT": "0x94b008aA00579c1307B0EF2c499aD98a8ce58e58"},
}

ERC20_ABI = [
    {"constant": True, "inputs": [{"name": "owner", "type": "address"}, {"name": "spender", "type": "address"}], "name": "allowance", "outputs": [{"name": "", "type": "uint256"}], "type": "function"},
    {"constant": False, "inputs": [{"name": "spender", "type": "address"}, {"name": "amount", "type": "uint256"}], "name": "approve", "outputs": [{"name": "", "type": "bool"}], "type": "function"},
    {"constant": True, "inputs": [], "name": "decimals", "outputs": [{"name": "", "type": "uint8"}], "type": "function"},
]


class BridgeServices:
    """Cross-chain bridge operations via Li.Fi."""

    def __init__(self, wallet_address: str, private_key: str | None = None, api_key: str | None = None):
        self.wallet_address = Web3.to_checksum_address(wallet_address)
        self.private_key = private_key
        self.api_key = api_key
        self._w3_cache: dict[str, Web3] = {}

    def _get_w3(self, chain: str) -> Web3:
        if chain not in CHAINS:
            raise ValueError(f"Unsupported chain: {chain}")
        if chain not in self._w3_cache:
            cfg = CHAINS[chain]
            w3 = Web3(Web3.HTTPProvider(cfg["rpc"]))
            if cfg.get("poa"):
                w3.middleware_onion.inject(ExtraDataToPOAMiddleware, layer=0)
            self._w3_cache[chain] = w3
        return self._w3_cache[chain]

    def _resolve_token(self, chain: str, token: str) -> str:
        upper = token.upper()
        native = CHAINS[chain]["native"]
        if upper == native:
            return NATIVE_ADDRESS
        chain_tokens = KNOWN_TOKENS.get(chain, {})
        if upper in chain_tokens:
            return chain_tokens[upper]
        if token.startswith("0x") and len(token) == 42:
            return Web3.to_checksum_address(token)
        raise ValueError(f"Unknown token '{token}' on {chain}. Known: {list(chain_tokens.keys())}")

    def _require_signer(self):
        if not self.private_key:
            raise ValueError("EVM_PRIVATE_KEY must be set for bridge execution.")

    def _headers(self) -> dict:
        h = {}
        if self.api_key:
            h["x-lifi-api-key"] = self.api_key
        return h

    async def get_chains(self) -> dict:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(f"{LIFI_API}/chains", headers=self._headers())
            if resp.status_code != 200:
                return {"success": False, "error": f"Li.Fi API error: {resp.text}"}
            data = resp.json()
        chains = [{"name": c["name"], "chain_id": c["id"], "key": c.get("key")} for c in data.get("chains", [])]
        return {"success": True, "chains": chains}

    async def get_routes(self, from_chain: str, to_chain: str, token: str, amount: float) -> dict:
        from_chain_id = CHAINS[from_chain]["chain_id"]
        to_chain_id = CHAINS[to_chain]["chain_id"]
        from_token = self._resolve_token(from_chain, token)

        # Get decimals
        if from_token == NATIVE_ADDRESS:
            decimals = 18
        else:
            w3 = self._get_w3(from_chain)
            contract = w3.eth.contract(address=Web3.to_checksum_address(from_token), abi=ERC20_ABI)
            decimals = contract.functions.decimals().call()

        raw_amount = str(int(Decimal(str(amount)) * Decimal(10 ** decimals)))

        # Find the same token on destination chain, or let Li.Fi figure it out
        to_token = self._resolve_token(to_chain, token) if token.upper() != CHAINS[from_chain]["native"] else NATIVE_ADDRESS

        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.post(f"{LIFI_API}/advanced/routes", json={
                "fromChainId": from_chain_id,
                "toChainId": to_chain_id,
                "fromTokenAddress": from_token,
                "toTokenAddress": to_token,
                "fromAmount": raw_amount,
                "fromAddress": self.wallet_address,
                "toAddress": self.wallet_address,
            }, headers=self._headers())
            if resp.status_code != 200:
                return {"success": False, "error": f"Li.Fi API error: {resp.text}"}
            data = resp.json()

        routes = []
        for route in data.get("routes", []):
            routes.append({
                "id": route.get("id"),
                "from_amount": amount,
                "to_amount": route.get("toAmountMin"),
                "to_amount_usd": route.get("toAmountUSD"),
                "gas_cost_usd": route.get("gasCostUSD"),
                "estimated_time_seconds": sum(s.get("estimate", {}).get("executionDuration", 0) for s in route.get("steps", [])),
                "steps": [
                    {"tool": s.get("tool"), "type": s.get("type")}
                    for s in route.get("steps", [])
                ],
            })

        return {
            "success": True,
            "from_chain": from_chain,
            "to_chain": to_chain,
            "token": token,
            "amount": amount,
            "routes": routes,
        }

    async def execute_bridge(self, from_chain: str, to_chain: str, token: str, amount: float, slippage: float = 0.5) -> dict:
        self._require_signer()

        from_chain_id = CHAINS[from_chain]["chain_id"]
        to_chain_id = CHAINS[to_chain]["chain_id"]
        from_token = self._resolve_token(from_chain, token)
        to_token = self._resolve_token(to_chain, token) if token.upper() != CHAINS[from_chain]["native"] else NATIVE_ADDRESS

        if from_token == NATIVE_ADDRESS:
            decimals = 18
        else:
            w3 = self._get_w3(from_chain)
            contract = w3.eth.contract(address=Web3.to_checksum_address(from_token), abi=ERC20_ABI)
            decimals = contract.functions.decimals().call()

        raw_amount = str(int(Decimal(str(amount)) * Decimal(10 ** decimals)))

        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{LIFI_API}/quote", params={
                "fromChain": from_chain_id,
                "toChain": to_chain_id,
                "fromToken": from_token,
                "toToken": to_token,
                "fromAmount": raw_amount,
                "fromAddress": self.wallet_address,
                "slippage": slippage / 100,
            }, headers=self._headers())
            if resp.status_code != 200:
                return {"success": False, "error": f"Li.Fi quote error: {resp.text}"}
            quote = resp.json()

        tx_request = quote.get("transactionRequest")
        if not tx_request:
            return {"success": False, "error": "No transaction in quote response"}

        w3 = self._get_w3(from_chain)

        # Approve if needed
        if from_token != NATIVE_ADDRESS and tx_request.get("to"):
            contract = w3.eth.contract(address=Web3.to_checksum_address(from_token), abi=ERC20_ABI)
            raw_amt = int(raw_amount)
            allowance = contract.functions.allowance(self.wallet_address, Web3.to_checksum_address(tx_request["to"])).call()
            if allowance < raw_amt:
                approve_tx = contract.functions.approve(
                    Web3.to_checksum_address(tx_request["to"]), 2**256 - 1
                ).build_transaction({
                    "from": self.wallet_address,
                    "nonce": w3.eth.get_transaction_count(self.wallet_address),
                    "chainId": CHAINS[from_chain]["chain_id"],
                })
                signed = Account.sign_transaction(approve_tx, self.private_key)
                tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
                w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)

        # Execute bridge tx
        tx = {
            "from": self.wallet_address,
            "to": Web3.to_checksum_address(tx_request["to"]),
            "data": tx_request["data"],
            "value": int(tx_request.get("value", "0"), 16) if isinstance(tx_request.get("value"), str) else int(tx_request.get("value", 0)),
            "nonce": w3.eth.get_transaction_count(self.wallet_address),
            "chainId": CHAINS[from_chain]["chain_id"],
            "gas": int(tx_request.get("gasLimit", "0"), 16) if isinstance(tx_request.get("gasLimit"), str) else int(tx_request.get("gasLimit", 500000)),
        }
        if tx_request.get("gasPrice"):
            tx["gasPrice"] = int(tx_request["gasPrice"], 16) if isinstance(tx_request["gasPrice"], str) else int(tx_request["gasPrice"])
        else:
            tx["maxFeePerGas"] = w3.eth.gas_price * 2
            tx["maxPriorityFeePerGas"] = Web3.to_wei(1, "gwei")

        signed = Account.sign_transaction(tx, self.private_key)
        tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=180)

        estimate = quote.get("estimate", {})
        return {
            "success": receipt["status"] == 1,
            "from_chain": from_chain,
            "to_chain": to_chain,
            "tx_hash": receipt["transactionHash"].hex(),
            "token": token,
            "amount": amount,
            "to_amount_expected": estimate.get("toAmountMin"),
            "estimated_time_seconds": estimate.get("executionDuration"),
            "gas_used": receipt["gasUsed"],
            "explorer": f"{CHAINS[from_chain]['explorer']}/tx/{receipt['transactionHash'].hex()}",
            "error": "Transaction reverted" if receipt["status"] != 1 else None,
        }

    async def get_status(self, tx_hash: str) -> dict:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(f"{LIFI_API}/status", params={"txHash": tx_hash}, headers=self._headers())
            if resp.status_code != 200:
                return {"success": False, "error": f"Status API error: {resp.text}"}
            data = resp.json()
        return {
            "success": True,
            "status": data.get("status"),
            "substatus": data.get("substatus"),
            "sending": data.get("sending"),
            "receiving": data.get("receiving"),
        }
```

- [ ] **Step 3: Create bridge.py CLI**

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = ["httpx", "web3", "eth-account"]
# ///
"""Cross-Chain Bridge CLI — via Li.Fi aggregator.

All output is JSON to stdout.

Run with: uv run bridge.py <command> [args]
"""

from __future__ import annotations

import argparse
import asyncio
import json
import os
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))


def _get_services(require_key: bool = False):
    from bridge_services import BridgeServices
    address = os.environ.get("EVM_WALLET_ADDRESS", "")
    private_key = os.environ.get("EVM_PRIVATE_KEY") or None
    api_key = os.environ.get("LIFI_API_KEY") or None
    if not address:
        _out({"success": False, "error": "EVM_WALLET_ADDRESS must be set."})
        sys.exit(1)
    if require_key and not private_key:
        _out({"success": False, "error": "EVM_PRIVATE_KEY must be set for bridging."})
        sys.exit(1)
    return BridgeServices(wallet_address=address, private_key=private_key, api_key=api_key)


def _out(data):
    print(json.dumps(data, default=str))


async def cmd_chains(args):
    svc = _get_services()
    _out(await svc.get_chains())

async def cmd_routes(args):
    svc = _get_services()
    _out(await svc.get_routes(args.from_chain, args.to_chain, args.token, args.amount))

async def cmd_execute(args):
    svc = _get_services(require_key=True)
    if args.amount <= 0:
        _out({"success": False, "error": "Amount must be positive"})
        return
    _out(await svc.execute_bridge(args.from_chain, args.to_chain, args.token, args.amount, args.slippage))

async def cmd_status(args):
    svc = _get_services()
    _out(await svc.get_status(args.tx))


def main():
    parser = argparse.ArgumentParser(description="Cross-Chain Bridge CLI (Li.Fi)")
    sub = parser.add_subparsers(dest="command", required=True)

    sub.add_parser("chains", help="List supported chains")

    p = sub.add_parser("routes", help="Get bridge routes")
    p.add_argument("--from-chain", required=True)
    p.add_argument("--to-chain", required=True)
    p.add_argument("--token", required=True)
    p.add_argument("--amount", type=float, required=True)

    p = sub.add_parser("execute", help="Execute bridge")
    p.add_argument("--from-chain", required=True)
    p.add_argument("--to-chain", required=True)
    p.add_argument("--token", required=True)
    p.add_argument("--amount", type=float, required=True)
    p.add_argument("--slippage", type=float, default=0.5)

    p = sub.add_parser("status", help="Check bridge status")
    p.add_argument("--tx", required=True)

    args = parser.parse_args()
    handler = {
        "chains": cmd_chains,
        "routes": cmd_routes,
        "execute": cmd_execute,
        "status": cmd_status,
    }[args.command]

    try:
        asyncio.run(handler(args))
    except Exception as e:
        _out({"success": False, "error": str(e)})
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Commit**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
git add cross-chain-bridge/
git commit -m "feat(skills): add cross-chain-bridge skill (Li.Fi)"
```

---

## Chunk 6: Smoke Testing & Final Verification

### Task 10: Test all skills

- [ ] **Step 1: Test evm-wallet config (read-only, no key needed)**

```bash
cd /Users/chetan/Documents/work/gigabrain/gigabrain.gg/skills
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run evm-wallet/scripts/evm_wallet.py config
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run evm-wallet/scripts/evm_wallet.py gas --chain base
```

- [ ] **Step 2: Test solana-wallet config**

```bash
SOL_WALLET_ADDRESS=So11111111111111111111111111111111111111112 uv run solana-wallet/scripts/sol_wallet.py config
```

- [ ] **Step 3: Test evm-swap token-info**

```bash
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run evm-swap/scripts/evm_swap.py token-info --chain base
```

- [ ] **Step 4: Test solana-swap price**

```bash
SOL_WALLET_ADDRESS=So11111111111111111111111111111111111111112 uv run solana-swap/scripts/sol_swap.py price --token SOL
```

- [ ] **Step 5: Test bridge chains**

```bash
EVM_WALLET_ADDRESS=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045 uv run cross-chain-bridge/scripts/bridge.py chains
```

- [ ] **Step 6: Verify all SKILL.md files are discovered by loader**

Check that the daemon skill loader would pick up all 5 new skills:
```bash
find skills/ -name "SKILL.md" | sort
```

Expected output includes: `evm-wallet`, `solana-wallet`, `evm-swap`, `solana-swap`, `cross-chain-bridge`

- [ ] **Step 7: Final commit with all fixes**

If any smoke tests revealed issues, fix and commit.
