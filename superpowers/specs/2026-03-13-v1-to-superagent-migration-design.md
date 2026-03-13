# V1 to SuperAgent Migration Design

## Overview

An "Upgrade to SuperAgent" button on V1 (scheduled) agents that converts them in-place to SuperAgents. The agent keeps its same ID, wallet, and funds. A wizard walks users through reviewing an auto-generated soul.md and connecting Telegram before the conversion happens.

## Migration Flow

Three-step modal wizard triggered from V1 agent detail pages:

### Step 1: Review Soul.md
- On wizard open, call backend to auto-generate a soul.md draft from the V1 agent's `goal` and `instructions` fields
- User reviews and edits the generated soul.md in a textarea
- "Next" proceeds to step 2

### Step 2: Connect Telegram (Optional)
- User pastes a Telegram bot token (created via @BotFather)
- "Skip" allows proceeding without Telegram
- "Next" proceeds to step 3

### Step 3: Confirm & Upgrade
- Summary showing: agent name, wallet address (preserved), soul.md preview, Telegram status
- "Upgrade" button triggers the migration
- On success, redirect to the SuperAgent detail page

## Backend

### POST /v2/agents/{agent_id}/generate-soul

Generates a soul.md draft from V1 agent data using LLM.

**Input**: Agent ID (from URL), user ID (from auth)
**Process**:
- Verify agent ownership
- Read agent's `goal`, `instructions`, `name`, `description` from DB
- Send to LLM with a prompt that converts structured V1 config into natural soul.md prose
- Return the generated markdown

**Response**:
```json
{
  "soul_md": "# Agent Name\n\n## Identity\n..."
}
```

### POST /v2/agents/{agent_id}/migrate

Converts a V1 agent to SuperAgent in-place.

**Input**:
```json
{
  "soul_md": "string (required)",
  "telegram_bot_token": "string (optional)"
}
```

**Process**:
1. Verify agent ownership
2. Verify `runtime_type == "v1"` (reject if already migrated)
3. Disable V1 agent (stop triggers)
4. Update DB record:
   - `runtime_type` = "persistent"
   - `agent_status` = "launching"
   - `telegram_bot_token` = provided token (if any)
   - Clear `triggers` (no longer needed)
5. Write soul.md to agent's EFS storage
6. Launch ECS Fargate container (same as normal SuperAgent launch)
7. Skip wallet provisioning (wallet already exists, linked by agent_id)
8. Return updated agent

**Response**: Standard agent object with `runtime_type: "persistent"`

**Errors**:
- 404: Agent not found
- 403: Not owner
- 409: Already a SuperAgent / migration in progress

## Frontend

### V1 Agent Detail Page Changes

**Legacy badge**: V1 agents show a "Legacy" badge next to their name on the detail page.

**Upgrade button**: Primary CTA button "Upgrade to SuperAgent" in the agent header area. Only visible on V1 agents (`runtime_type === "v1"`).

### Migration Wizard Modal

A full-screen or large modal with 3 steps, step indicator at top.

**Step 1 - Soul.md**:
- On mount, call `generateSoulFromLegacy` mutation
- Show loading state while generating
- Render generated soul.md in an editable textarea/editor
- Show original V1 instructions in a collapsed "Original Config" section for reference

**Step 2 - Telegram**:
- Bot token input field
- Link to @BotFather instructions
- Skip button

**Step 3 - Confirm**:
- Read-only summary
- Wallet address displayed (from existing wallet data)
- "Your wallet and funds will be preserved" reassurance text
- Upgrade button with loading state

### NestJS Mutations

```graphql
mutation migrateAgentToSuperAgent($agentId: String!, $soulMd: String!, $telegramBotToken: String) {
  migrateAgentToSuperAgent(agentId: $agentId, soulMd: $soulMd, telegramBotToken: $telegramBotToken) {
    ...AgentFields
  }
}

mutation generateSoulFromLegacy($agentId: String!) {
  generateSoulFromLegacy(agentId: $agentId) {
    soulMd
  }
}
```

## What Stays the Same

- **Agent ID**: No change, same DB record
- **Wallet**: Same wallet, same address, same funds. Wallet FK is on agent_id so it carries over
- **Agent name**: Preserved (user can change later)
- **Description**: Preserved

## What Changes

- **runtime_type**: "v1" to "persistent"
- **Triggers**: Cleared (SuperAgents use always-on execution, not cron/alpha triggers)
- **Instructions/Goal**: Replaced by soul.md
- **Execution model**: From batch trigger-based to always-on container with Telegram interface
- **Skills**: SuperAgent gets access to installable skills ecosystem

## Files to Create/Modify

| File | Action |
|------|--------|
| `brain-core/.../routers/platform_agents.py` | Add `/generate-soul` and `/migrate` endpoints |
| `brain-core/.../orchestrator/launch_service.py` | Add `migrate_agent()` method |
| `gigabrain-backend/.../daemon-agent.service.ts` | Add `migrateAgent` and `generateSoul` methods |
| `gigabrain-backend/.../daemon-agent.resolver.ts` | Add 2 mutations |
| `gigabrain-frontend/.../agents/[id]/page.tsx` | Add Legacy badge + Upgrade button |
| `gigabrain-frontend/.../agents/[id]/components/MigrationWizard.tsx` | **New file** - 3-step wizard modal |
| `gigabrain-frontend/src/graphql/mutations.tsx` | Add 2 mutations |
