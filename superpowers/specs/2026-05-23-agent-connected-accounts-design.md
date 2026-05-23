# Agent Connected Accounts Design

## Goal

Let a SuperAgent bind to one connected account at launch, then allow that binding to be changed later from the live agent settings. The first version supports platform agent wallets for EVM and Solana plus full-access HyperLiquid user integrations, while keeping the model open for future integrations such as Binance.

## Current State

The launch flow currently displays wallet choices in the frontend, but the selected values are not sent to the backend. The GraphQL `LaunchAgentInput`, Nest proxy, Brain internal `/v2/agents/launch` request, and orchestrator launch methods do not accept account selection.

During provisioning, Brain always creates platform-managed agent wallets:

- EVM wallet in `agent_wallets`, currently labeled as `exchange="hyperliquid"`.
- Solana wallet in `agent_wallets`, currently labeled as `exchange="universal"`.

HyperLiquid profile accounts are stored separately in `user_integrations`. Agent execution currently prefers `agent_wallets` for HyperLiquid tools and does not read a launch-time selected profile integration.

## Product Model

The product should present all usable accounts through one concept: a connected account. Internally the account may come from an agent wallet or a user integration.

Launch options in this version:

- Agent EVM wallet.
- Agent Solana wallet.
- Full-access HyperLiquid integrations from the user's profile.

Polymarket is intentionally removed from launch account selection for now because the current Polymarket backend flow creates a platform-managed Privy/Safe wallet and is not yet a general external-wallet choice.

View-only HyperLiquid integrations are not selectable for execution. They may be shown later in monitoring surfaces, but they must not be offered as live trading accounts.

## Data Contract

Add a single account binding object to launch and update APIs:

```ts
type ConnectedAccountSource = 'agent_wallet' | 'user_integration'
type ConnectedAccountType = 'evm' | 'solana' | 'hyperliquid'

type ConnectedAccountInput = {
  source: ConnectedAccountSource
  account_type: ConnectedAccountType
  integration_id?: string
}
```

Store the normalized binding on the agent as `agent.data.connected_account`.

Valid bindings:

- `{ source: 'agent_wallet', account_type: 'evm' }`
- `{ source: 'agent_wallet', account_type: 'solana' }`
- `{ source: 'user_integration', account_type: 'hyperliquid', integration_id: '<uuid>' }`

Invalid bindings:

- `user_integration` without `integration_id`.
- `agent_wallet` with `integration_id`.
- `hyperliquid` with a view-only integration.
- Any integration that is inactive, missing, belongs to another user, or is not `HYPERLIQUID`.

If no account is provided, default to `{ source: 'agent_wallet', account_type: 'evm' }` for backwards compatibility.

## Backend Behavior

Nest GraphQL should validate basic shape and forward `connected_account` to Brain internal API. Brain internal API should perform authoritative validation because it owns the final agent record and runtime behavior.

Brain launch should:

1. Validate `connected_account` against the requesting user.
2. Store the normalized binding in `agent.data.connected_account` during `launch_init()`.
3. Continue provisioning both EVM and Solana agent wallets so the user can switch later without a second provisioning flow.
4. Return launch response as today; wallet address can remain empty during background provisioning.

Add an update endpoint for live agents:

```ts
updateAgentConnectedAccount(agentId: string, connected_account: ConnectedAccountInput)
```

The update endpoint should validate ownership and account availability, write `agent.data.connected_account`, and return the normalized binding. Switching accounts affects future actions only; existing external positions and orders remain on the previous venue/account.

## Runtime Behavior

Runtime tools should resolve the selected account from `agent.data.connected_account`.

For HyperLiquid:

- If selected account is `user_integration/hyperliquid`, load that exact integration by ID and require full-access credentials.
- If selected account is `agent_wallet/evm`, use the existing agent wallet path.
- If selected account is `agent_wallet/solana`, HyperLiquid execution should fail with a clear message that HyperLiquid requires an EVM or HyperLiquid account.

If the selected integration is later removed or deactivated, runtime must fail closed with a clear "connected account needs attention" error instead of falling back to another account.

## Frontend Launch UX

Replace the current stacked "Trading Access" cards with a compact connected-account picker that matches the existing launch-page components and visual density.

The right rail should show:

- Agent avatar and name.
- Model picker.
- Connected Account picker.
- Telegram collapsible control.
- Launch button.

The connected-account picker should behave like a ChatGPT-style control:

- Main collapsed row: icon, selected account label, short address where available, chevron.
- Menu items: Agent EVM wallet, Agent Solana wallet, and each full-access HyperLiquid integration.
- Secondary manage action: link to Profile integrations for adding HyperLiquid accounts.

The launch flow should not include Polymarket.

## Live Agent Settings UX

Add a Connected Account section to the live agent settings page. It should use the same option model as launch and allow changing the selected account.

When a user switches accounts, show concise copy:

> Future actions will use this account. Existing positions and open orders remain on the account where they were created.

When the selected account is unavailable, show a needs-attention state and disable execution-oriented actions until the user selects a valid account.

## Option Source API

The frontend can initially assemble options from existing data:

- User profile `evmAddress` for the EVM agent-wallet option label.
- Built-in Solana agent-wallet option.
- `USER_INTEGRATIONS(type: HYPERLIQUID)` filtered to active and non-view-only.

For scalability, add a backend query after the first implementation if duplication grows:

```ts
agentConnectedAccountOptions(agentId?: string): ConnectedAccountOption[]
```

This query would let future Binance, Coinbase, and Solana-style accounts appear without rebuilding frontend filtering logic each time.

## Testing

Backend tests should cover:

- Launch accepts omitted `connected_account` and defaults to agent EVM wallet.
- Launch accepts agent Solana wallet.
- Launch accepts full-access HyperLiquid integration owned by the user.
- Launch rejects view-only HyperLiquid integration.
- Launch rejects another user's integration.
- Live update validates and persists the new binding.

Frontend tests should cover:

- Polymarket is absent from launch account options.
- View-only HyperLiquid integrations are absent or disabled.
- Selecting HyperLiquid sends `connected_account.source='user_integration'`, `account_type='hyperliquid'`, and the integration ID.
- Selecting EVM/Solana sends the corresponding `agent_wallet` binding.

Runtime tests should cover:

- Agent HyperLiquid tools use the selected HyperLiquid integration when configured.
- Agent HyperLiquid tools fail closed when the selected integration is removed.
- Agent HyperLiquid tools do not fall back to defaults when a selected account is invalid.

## Rollout Notes

This is backwards compatible for existing agents because missing `connected_account` resolves to agent EVM wallet. Existing agents can receive an explicit binding the first time settings are saved.

Do not push directly to `main`. Use feature branches and PRs for each repo touched.
