# Agent Connected Accounts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire SuperAgent launch and live settings to a generic connected-account binding that supports agent EVM/Solana wallets and full-access HyperLiquid integrations.

**Architecture:** Add a small `connected_account` contract that travels from frontend GraphQL to Nest, then to Brain internal API, then into `agent.data.connected_account`. Runtime HyperLiquid tools resolve this binding and either use the selected HyperLiquid integration or the existing agent wallet path. The frontend presents all launch options through one compact picker and removes Polymarket from this flow.

**Tech Stack:** Next.js/React/Apollo/TypeScript frontend, NestJS GraphQL backend, FastAPI/Pydantic/Python Brain internal API, pytest/Jest/TypeScript verification.

---

### Task 1: Backend GraphQL Contract

**Files:**
- Modify: `gigabrain-backend/src/user-agent/dto/agent-platform.types.ts`
- Modify: `gigabrain-backend/src/user-agent/agent-platform.service.ts`
- Test: `gigabrain-backend/src/user-agent/agent-platform.service.spec.ts`

- [ ] Add `ConnectedAccountInput` with `source`, `account_type`, and optional `integration_id`.
- [ ] Add `connected_account?: ConnectedAccountInput` to `LaunchAgentInput`.
- [ ] Forward `connected_account` through `AgentPlatformService.launch()`.
- [ ] Add a Jest test proving `launch()` forwards the nested object unchanged.
- [ ] Run `pnpm test -- agent-platform.service.spec.ts --runInBand`.

### Task 2: Brain Launch Contract And Validation

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/servers/frontend_api/routers/platform_agents.py`
- Modify: `intelligence-monorepo/agents/brain-core/brain_core/shared/services/orchestrator/launch_service.py`
- Test: `intelligence-monorepo/agents/brain-core/tests/test_connected_account_contract.py`

- [ ] Add Pydantic models for `ConnectedAccountInput` and allowed source/account values.
- [ ] Default missing bindings to `{"source": "agent_wallet", "account_type": "evm"}`.
- [ ] Validate HyperLiquid integration bindings by querying `user_integrations` for user ownership, active status, `HYPERLIQUID`, and non-view-only credentials.
- [ ] Store the normalized binding in `agent.data.connected_account` during `launch_init()`.
- [ ] Keep provisioning both EVM and Solana agent wallets.
- [ ] Add pytest coverage for default binding, agent Solana binding, full-access HyperLiquid binding, view-only rejection, and wrong-user rejection.
- [ ] Run `python3 -m py_compile` on changed Brain files and targeted pytest if dependencies allow.

### Task 3: Brain Runtime Resolution

**Files:**
- Modify: `intelligence-monorepo/agents/brain-core/brain_core/specialists/hyperliquid/hyperliquid_trade_tools.py`
- Test: `intelligence-monorepo/agents/brain-core/tests/test_hyperliquid_connected_account_resolution.py`

- [ ] Load `agent.data.connected_account` when `agent_id` is present.
- [ ] If binding is `user_integration/hyperliquid`, load that exact integration and require trading credentials.
- [ ] If binding is `agent_wallet/evm` or missing, use the existing agent wallet path.
- [ ] If binding is unavailable, removed, view-only, or Solana-only, fail closed with a clear message and do not fall back to defaults.
- [ ] Add targeted tests around resolver choice and fail-closed behavior.
- [ ] Run `python3 -m py_compile` and targeted pytest if dependencies allow.

### Task 4: Frontend Launch Picker

**Files:**
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/types.ts`
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/create/page.tsx`
- Modify: `gigabrain-frontend/src/app/(dashboard)/components/GlobalButton.tsx` only if needed to preserve existing text-fit behavior.

- [ ] Add `ConnectedAccountInput` and `connected_account?: ConnectedAccountInput` to `LaunchAgentInput`.
- [ ] Replace the stacked Trading Access cards with one compact Connected Account picker.
- [ ] Include options for Agent EVM wallet, Agent Solana wallet, and active full-access HyperLiquid integrations only.
- [ ] Remove Polymarket from launch.
- [ ] Send the selected `connected_account` in `handleLaunch`.
- [ ] Preserve the existing launch-page design components and restore button text truncation behavior if the local diff removed it.
- [ ] Run `pnpm -s exec tsc --noEmit --pretty false`.

### Task 5: Live Agent Settings

**Files:**
- Modify: `gigabrain-backend/src/user-agent/dto/agent-platform.types.ts`
- Modify: `gigabrain-backend/src/user-agent/agent-platform.resolver.ts`
- Modify: `gigabrain-backend/src/user-agent/agent-platform.service.ts`
- Modify: `gigabrain-frontend/src/graphql/mutations.tsx`
- Modify: `gigabrain-frontend/src/graphql/queries.tsx`
- Modify: `gigabrain-frontend/src/app/(dashboard)/agents/[id]/components/DaemonSettingsTab.tsx`

- [ ] Add a mutation to update an agent's `connected_account`.
- [ ] Add the selected binding to `PlatformAgent` responses so settings can render current state.
- [ ] Add a Connected Account section to live settings using the same option model.
- [ ] Show copy that future actions use the new account and existing positions/orders remain where created.
- [ ] Run backend TypeScript and frontend TypeScript checks.

### Task 6: Final Verification

**Files:**
- Verify all touched repos.

- [ ] Run backend targeted tests and `pnpm -s exec tsc --noEmit --pretty false`.
- [ ] Run frontend `pnpm -s exec tsc --noEmit --pretty false`.
- [ ] Run Brain `python3 -m py_compile` on changed files and targeted pytest if available.
- [ ] Run `git diff --check` in all touched repos.
- [ ] Summarize changed files, verification output, and any blocked tests.
